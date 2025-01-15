---
title: "LINE Botで感情分析を実装"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["LINE", "AWS", "Lambda"]
published: false
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