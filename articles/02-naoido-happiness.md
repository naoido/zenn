---
title: "AWS + LINE Botで感情分析を実装"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["LINE", "AWS", "Lambda"]
published: true
---
# プロジェクト概要
このプロジェクトはLINEで画像を送信すると、[Amazon Rekognition](https://aws.amazon.com/jp/rekognition/)で感情分析をし、幸福度を表示するというBotを作成しました。
背景としましては、結婚式や成人式の時にみんなで遊べるちょっとしたサービスがあったらなと思い急遽実装しました。

Botの画面はこんな感じです
![サンプル](/images/happiness-demo.png)

## 構成図
![構成図](/images/happiness-structure.png)
S3に画像やWebサイトを置いて、バックエンドをLambdaで実装しています。
今回は小規模かつ短期・高リクエストが予想されるのでDynamoDBではなくElastiCacheを採用しています。
API GatewayではRestAPIはもちろん、リアルタイムでユーザーデータを反映させるためにWebsocketでも使用してます。

# Lambdaの実装
今回はコールドスタートができるだけ起きてほしくないので、単一のLambdaで全てを処理しています。
以下のように全て一つのhandlerで受けて、各メソッドに枝分かれさせています(かなり雑ですが)
```python
def lambda_handler(event, context):
    route_key = event["requestContext"].get("routeKey")

    # websocket関係
    if route_key == "$connect":
        return connect_handler(event, context) # コネクションの登録
    elif route_key == "$disconnect":
        return disconnect_handler(event, context) # コネクションの削除
    elif route_key == "$default":
        return default_handler(event, context) # デフォルトハンドラ
    # RestAPI用
    elif method:=event.get("httpMethod"):
        path = event.get('path')
        if method == "POST" and path == "/line":
            return line_handler(event, context) # lineからのWebhookを管理
        if method == "GET":
            params = event.get('queryStringParameters')
            if path == "/users_data": # 全てのユーザーデータを取得
                return {
                        "statusCode": 200,
                        "headers": headers,
                        "body": json.dumps(get_all_users_data())
                }
            elif path.startswith("/user_data"): # 特定のユーザーのデータを取得
                user_id = params.get("user_id")
                message_id = params.get("message_id")
                return {
                    "statusCode": 200,
                    "headers": headers,
                    "body": json.dumps(get_result(user_id, message_id))
                }
```
## Lambdaで関連サービスを使うためのポリシー設定
:::message
もっと制限した方がセキュアですが自主制作なので雑に設定してます。
:::
| ポリシー名 | 用途 |
| --- | --- |
| AmazonRekognitionReadOnlyAccess | LambdaからRekognitionを使用する為 |
| AmazonS3FullAccess | LambdaからS3を操作する為 |
| AWSLambdaVPCAccessExecutionRole | LambdaをVPCに載せる為<br>※これがないとVPCを設定する時にエラーが出ます |
| execute-api | websocketをAPI Gateway経由で送信する為 |

execute-apiはカスタムポリシーで設定してます。
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "execute-api:*",
            "Resource": "*"
        }
    ]
}
```

## LambdaからElastiCacheへのアクセス
今回ElastiCacheではValkeyを使用していますが、Redisと変わりないので
Redisにアクセスするときと同じようにできます。
:::message
詰まりポイントとして、ssl=Trueにしないとタイムアウトします
:::
```python
client = None # clientを使いまわせるように保持
def get_redis_client():
    global client
    if client:
        return client
    REDIS_HOST = os.environ.get("REDIS_HOST")
    REDIS_PORT = int(os.environ.get("REDIS_PORT"))
    REDIS_PASSWORD = os.environ.get("REDIS_PASSWORD", "")

    client = redis.Redis(
        host=REDIS_HOST,
        port=REDIS_PORT,
        password=REDIS_PASSWORD,
        decode_responses=True, # これTureにしないとデータがbyteで返ってくる
        ssl=True # 通信を暗号化するためにTrue
    )

    return client
```
## LambdaでのWebsocket実装
Lambdaでwebsocketのハンドリングをしています。
Lambdaだけではコネクションを保存できないので、ElastiCacheに保存しています。
API GatewayはWebsocket用にRestAPI用とは別に作る必要があります。
```python
def connect_handler(event, context):
    connection_id = event["requestContext"]["connectionId"]
    get_redis_client().sadd(REDIS_CONNECTION_SET_KEY, connection_id)

    return {
        "statusCode": 200,
        "headers": headers, # CORSエラー回避のためにheadersを入れてます
    }


def disconnect_handler(event, context):
    connection_id = event["requestContext"]["connectionId"]
    get_redis_client().srem(REDIS_CONNECTION_SET_KEY, connection_id)

    return {
        "statusCode": 200,
        "headers": headers,
    }


def default_handler(event, context):
    return {
        "statusCode": 200,
        "headers": headers,
    }


def send_content(context):
    """
    Websocketでフロントに配信
    """
    apigw = get_apigw_management_client()
    broadcast_data = context

    redis_client = get_redis_client()
    all_connections = redis_client.smembers(REDIS_CONNECTION_SET_KEY)

    for conn_id in all_connections:
        try:
            apigw.post_to_connection(
                ConnectionId=conn_id,
                Data=broadcast_data
            )
        except apigw.exceptions.GoneException:
            redis_client.srem(REDIS_CONNECTION_SET_KEY, conn_id)
        except Exception as e:
            logger.error(f"Error sending to {conn_id}: {str(e)}")

# boto3でAPI Gatewayを取得
def get_apigw_management_client():
    return boto3.client('apigatewaymanagementapi', endpoint_url=f'https://{ドメイン}/{ステージ}')
```
## LambdaでRekognition画像解析📊
LINE Botで受け取った画像をRekognitionで画像解析するために、まずS3に保存します。
Rekognition自体の設定は特にないです。(便利！！)
:::message
Rekognitionを使用するためにLambdaにポリシーを追加する必要があります。
:::

:::details Rekognitionで取れる情報サンプル
```json:result.json
{
    "FaceDetails": [
        {
            "BoundingBox": {
                "Width": 0.3414241671562195,
                "Height": 0.2682712972164154,
                "Left": 0.36009445786476135,
                "Top": 0.4315587282180786
            },
            "AgeRange": {
                "Low": 15,
                "High": 21
            },
            "Smile": {
                "Value": true,
                "Confidence": 99.76475524902344
            },
            "Eyeglasses": {
                "Value": false,
                "Confidence": 99.99198913574219
            },
            "Sunglasses": {
                "Value": false,
                "Confidence": 99.99812316894531
            },
            "Gender": {
                "Value": "Male",
                "Confidence": 98.1222915649414
            },
            "Beard": {
                "Value": false,
                "Confidence": 96.17134094238281
            },
            "Mustache": {
                "Value": false,
                "Confidence": 99.88905334472656
            },
            "EyesOpen": {
                "Value": true,
                "Confidence": 71.74201965332031
            },
            "MouthOpen": {
                "Value": false,
                "Confidence": 98.43098449707031
            },
            "Emotions": [
                {
                    "Type": "HAPPY",
                    "Confidence": 99.67448425292969
                },
                {
                    "Type": "DISGUSTED",
                    "Confidence": 0.13151168823242188
                },
                {
                    "Type": "CALM",
                    "Confidence": 0.0072479248046875
                },
                {
                    "Type": "SAD",
                    "Confidence": 0.002175569534301758
                },
                {
                    "Type": "CONFUSED",
                    "Confidence": 0.0016142925014719367
                },
                {
                    "Type": "SURPRISED",
                    "Confidence": 0.0005066394805908203
                },
                {
                    "Type": "ANGRY",
                    "Confidence": 0.00043511390686035156
                },
                {
                    "Type": "FEAR",
                    "Confidence": 0.000008940696716308594
                }
            ],
            "Landmarks": [
                {
                    "Type": "eyeLeft",
                    "X": 0.44940710067749023,
                    "Y": 0.5346904397010803
                },
                {
                    "Type": "eyeRight",
                    "X": 0.6057807207107544,
                    "Y": 0.5417009592056274
                },
                {
                    "Type": "mouthLeft",
                    "X": 0.4532623291015625,
                    "Y": 0.6208639144897461
                },
                {
                    "Type": "mouthRight",
                    "X": 0.58384108543396,
                    "Y": 0.6268877387046814
                },
                {
                    "Type": "nose",
                    "X": 0.5132337212562561,
                    "Y": 0.5821033120155334
                },
                {
                    "Type": "leftEyeBrowLeft",
                    "X": 0.39656564593315125,
                    "Y": 0.5124384164810181
                },
                {
                    "Type": "leftEyeBrowRight",
                    "X": 0.481508731842041,
                    "Y": 0.509492039680481
                },
                {
                    "Type": "leftEyeBrowUp",
                    "X": 0.4385930597782135,
                    "Y": 0.5033077597618103
                },
                {
                    "Type": "rightEyeBrowLeft",
                    "X": 0.5705402493476868,
                    "Y": 0.5132946372032166
                },
                {
                    "Type": "rightEyeBrowRight",
                    "X": 0.667339026927948,
                    "Y": 0.5242101550102234
                },
                {
                    "Type": "rightEyeBrowUp",
                    "X": 0.6178725957870483,
                    "Y": 0.5109927654266357
                },
                {
                    "Type": "leftEyeLeft",
                    "X": 0.4228461682796478,
                    "Y": 0.5332108736038208
                },
                {
                    "Type": "leftEyeRight",
                    "X": 0.48010358214378357,
                    "Y": 0.5367599725723267
                },
                {
                    "Type": "leftEyeUp",
                    "X": 0.44891223311424255,
                    "Y": 0.5301703214645386
                },
                {
                    "Type": "leftEyeDown",
                    "X": 0.44956815242767334,
                    "Y": 0.5384357571601868
                },
                {
                    "Type": "rightEyeLeft",
                    "X": 0.5744288563728333,
                    "Y": 0.5409452319145203
                },
                {
                    "Type": "rightEyeRight",
                    "X": 0.6342695951461792,
                    "Y": 0.5426113605499268
                },
                {
                    "Type": "rightEyeUp",
                    "X": 0.6056773066520691,
                    "Y": 0.5371313691139221
                },
                {
                    "Type": "rightEyeDown",
                    "X": 0.604184091091156,
                    "Y": 0.5453396439552307
                },
                {
                    "Type": "noseLeft",
                    "X": 0.4882659316062927,
                    "Y": 0.5915452837944031
                },
                {
                    "Type": "noseRight",
                    "X": 0.5456506013870239,
                    "Y": 0.5940533876419067
                },
                {
                    "Type": "mouthUp",
                    "X": 0.514017641544342,
                    "Y": 0.6131271123886108
                },
                {
                    "Type": "mouthDown",
                    "X": 0.5124568343162537,
                    "Y": 0.6393837332725525
                },
                {
                    "Type": "leftPupil",
                    "X": 0.44940710067749023,
                    "Y": 0.5346904397010803
                },
                {
                    "Type": "rightPupil",
                    "X": 0.6057807207107544,
                    "Y": 0.5417009592056274
                },
                {
                    "Type": "upperJawlineLeft",
                    "X": 0.371329665184021,
                    "Y": 0.5342679023742676
                },
                {
                    "Type": "midJawlineLeft",
                    "X": 0.3919771611690521,
                    "Y": 0.6293109059333801
                },
                {
                    "Type": "chinBottom",
                    "X": 0.5117601752281189,
                    "Y": 0.6852112412452698
                },
                {
                    "Type": "midJawlineRight",
                    "X": 0.6660684943199158,
                    "Y": 0.6410271525382996
                },
                {
                    "Type": "upperJawlineRight",
                    "X": 0.7101116180419922,
                    "Y": 0.5487430095672607
                }
            ],
            "Pose": {
                "Roll": 3.744174003601074,
                "Yaw": -2.709038496017456,
                "Pitch": 5.688099384307861
            },
            "Quality": {
                "Brightness": 74.15724182128906,
                "Sharpness": 86.86019134521484
            },
            "Confidence": 99.99979400634766,
            "FaceOccluded": {
                "Value": false,
                "Confidence": 99.96943664550781
            },
            "EyeDirection": {
                "Yaw": -2.991691827774048,
                "Pitch": -7.667012691497803,
                "Confidence": 99.8233871459961
            }
        }
    ],
    "ResponseMetadata": {
        "RequestId": "6e4a67d5-6677-472e-b3e5-b9dfd658bc25",
        "HTTPStatusCode": 200,
        "HTTPHeaders": {
            "x-amzn-requestid": "6e4a67d5-6677-472e-b3e5-b9dfd658bc25",
            "content-type": "application/x-amz-json-1.1",
            "content-length": "3502",
            "date": "Tue, 14 Jan 2025 19:40:32 GMT"
        },
        "RetryAttempts": 0
    }
}
```
:::
```python
@handler.add(MessageEvent, message=ImageMessage)
def handle_image(line_event):
    s3_client = boto3.client('s3') # S3にアクセスするため
    message_id = line_event.message.id # メッセージID
    user_id = line_event.source.user_id # ユーザーID
    message_content = line_bot_api.get_message_content(message_id) # 画像を取得

    # /tmpに画像を一時的に保存
    with open(path:=f"/tmp/{message_id}.jpg", "wb") as f:
        for chunk in message_content.iter_content():
            f.write(chunk)
    # /tmpに保存した画像をS3にアップロード
    s3_client.upload_file(path, BUCKET_NAME, key:=f"images/{user_id}/{message_id}.jpg")

    # rekognitionのclientを作成
    rekognition = boto3.client('rekognition')
    try:
        # ここで顔の表情を分析してもらってる(めっちゃ簡単！！)
        response = rekognition.detect_faces(
            Image={
                "S3Object": {
                    "Bucket": BUCKET_NAME,
                    "Name": key
                }
            },
            Attributes=["ALL"]
        )
        # 複数人認識されるのでループして合計を算出
        if face_details:=response.get("FaceDetails", []):
            emotions = defaultdict(float)
            for face in face_details:
                for emotion in face.get("Emotions"):
                    emotions[emotion["Type"]] += emotion["Confidence"]
            emotion = dict(emotion)
        else:
            emotions = {}
        
        # ~~ 色々な処理(データの整形) ~~

        # FlexMessageを送信(次のセクションでFlexMessageの作り方を記載してます)
        line_bot_api.push_message(user_id, messages=get_flex_message(user_id, message_id, result_json["total_score"]))
    except Exception as e:
        logger.error(e)
        line_bot_api.push_message(user_id, messages=TextSendMessage(text="画像がうまく読み込めませんでした。"))
```
### FlexMessageを送る
↓のサイトでメッセージのベースを作成します。
https://developers.line.biz/flex-simulator/
作成したテンプレートのjsonを保存してpythonコードに直書きします
今回は初期からほぼ触ってない感じです。
```python
def get_flex_message(user_id, message_id, score):
    # 今回LIFFで表示したいのでLIFFのリンクを指定しています。
    result_page = f"https://liff.line.me/{liffアプリID}?user_id={user_id}&message_id={message_id}"
    flex = {
        "type": "bubble",
        "hero": {
            "type": "image",
            "url": f"https://{Coudfrontのドメイン}/images/{user_id}/{message_id}.jpg",
            "size": "full",
            "aspectRatio": "20:13",
            "aspectMode": "cover",
            "action": {
            "type": "uri",
            "uri": result_page
            }
        },
        "body": {
            "type": "box",
            "layout": "vertical",
            "contents": [
            {
                "type": "text",
                "text": f"幸福度{score}pt！！",
                "weight": "bold",
                "size": "xl"
            }
            ]
        },
        "footer": {
            "type": "box",
            "layout": "vertical",
            "spacing": "sm",
            "contents": [
            {
                "type": "button",
                "style": "link",
                "height": "sm",
                "action": {
                "type": "uri",
                "label": "詳しく見る",
                "uri": result_page
                }
            },
            {
                "type": "box",
                "layout": "vertical",
                "contents": [],
                "margin": "sm"
            }
            ],
            "flex": 0
        }
    }
    return FlexSendMessage(
        alt_text="解析結果",
        contents=flex
    )
```
これを送信すると以下のようになります。
![FlexMessage](/images/flex-message-sample.png)

# フロントの実装
フロントは雑にVuetifyを使用して作ってます。
liffを使って詳細画面が開けます。
![liff](/images/happiness-liff-sample.jpg)
## エンドポイントの実装
LIFFで色々な画面を開きたかったので、
`/liff`をエンドポイントURLにして、パラメータを見てページ遷移させてます。
```vue:liff.vue
<template>
    <div>
        リダイレクト中...
    </div>
</template>

<script setup>
import { useRoute, useRouter } from 'vue-router';

const router = useRouter()
const route = useRoute()

let { user_id, message_id } = route.query;

// user_idがパラメータに存在したら詳細ページに遷移
if (user_id) {
    router.push(`/user/${user_id}/${message_id}`)
}
</script>
```
## ページ構成
vue-routerでは`[パラメータ]`でルートパラメーターを設定できます。
```sh
.
├── index.vue # indexページ
├── liff.vue # liffのエンドポイント
└── user
    └── [user_id] # user_idを取得できるように
        ├── [id].vue # message_idで特定の結果を見れるように
        └── index.vue # 最後の結果が表示されます
```
# あとがき
今回は初めてRekognitionを使用しましたが、めちゃくちゃ簡単に使用でき感動しました。
2日で雑に作った感じですが、なかなか楽しめるものができ満足してます。
複数の顔がある場合もその分合算されるので集合写真とかめちゃくちゃ最強です。
ランキング機能も簡単に実装できると思うので実装したい！

そして、Lambdaでのwebsocketの実装簡単すぎない？？
普段使いだとDynamoDBとかにセッション情報保持すると思うけど、なかなかサーバレスな時代になる理由も納得。

# 参考
https://aws.amazon.com/jp/rekognition/
https://qiita.com/charon/items/18f781024ff6ec0854a6
https://developers.line.biz/ja/docs/messaging-api/using-flex-message-simulator