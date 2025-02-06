---
title: "[Flutter] GoogleFitAPIからHealthConnectAPIへの移行手順📄"
emoji: "🚶‍♀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "android"]
published: true
---

# はじめに
:::message
今回の変更はGoogleFitアプリ自体が無くなるのではなく、GoogleFitなどで歩数をHealthConnectに保存し、その保存したデータをHealthConnectAPIを使って取り出すというフローに変わる点が重要です。
:::
2025/06/30にGoogleFitAPIのサポートが終了するのを受け、今更ですがGoogleFitAPIからHealthConnectへの移行した手順をまとめましたので共有します。
<br>
Google公式が出している移行ガイドも置いておきます。
https://developer.android.com/health-and-fitness/guides/health-connect/migrate/migration-guide?hl=ja

# 本記事の環境
Flutterバージョン
```bash
% flutter --version
Flutter 3.27.3 • channel stable • https://github.com/flutter/flutter.git
Framework • revision c519ee916e (2 weeks ago) • 2025-01-21 10:32:23 -0800
Engine • revision e672b006cb
Tools • Dart 3.6.1 • DevTools 2.40.2

```
使用しているパッケージ
```yaml:pubspec.yaml
# サポートが終了するまでGoogleFitも今まで通りサポートする為に10.2.0を使用
# バージョン11以上からGoogleFitが使えなくなります
# https://pub.dev/packages/health/changelog#1100
health: ^10.2.0
appcheck: ^1.5.2
url_launcher: ^6.3.1
permission_handler: ^11.3.1
```

# GoogleFitAPIとHealthConnectAPIの違い
:::message
HealthConnectAPIとGoogleFitAPIにはいくつかの違いがあるので、この記事に載っていないところは各自で詳細を調べていただけますと幸いです。
:::
例として、HealthConnectAPIでは最大30日前までしか取得できません。
https://developer.android.com/health-and-fitness/guides/health-connect/develop/aggregate-data?hl=ja#read-restriction

# 手順
ここからは、実際に行った手順をご紹介します。
## 1. AndroidManifest.xmlの変更
```diff xml:AndroidManifest.xml
 <manifest>
    <uses-permission android:name="android.permission.ACTIVITY_RECOGNITION"/>
    <!-- 任意でパラメータのパーミッションを追加する -->
+   <uses-permission android:name="android.permission.health.READ_STEPS"/>

    <queries>
        <package android:name="com.google.android.apps.fitness" />
+       <package android:name="com.google.android.apps.healthdata" />
+       <intent>
+           <action android:name="androidx.health.ACTION_SHOW_PERMISSIONS_RATIONALE" />
+       </intent>
    </queries>
        <intent-filter>
+                <action android:name="android.intent.action.VIEW_PERMISSION_USAGE" />
+                <category android:name="android.intent.category.HEALTH_PERMISSIONS" />
+                <action android:name="androidx.health.ACTION_SHOW_PERMISSIONS_RATIONALE" />
        </intent-filter>
    <activity>

    </activity>

    <application>
+        <activity-alias
+           android:name="ViewPermissionUsageActivity"
+           android:exported="true"
+           android:targetActivity=".MainActivity"
+           android:permission="android.permission.START_VIEW_PERMISSION_USAGE">
+           <intent-filter>
+               <action android:name="android.intent.action.VIEW_PERMISSION_USAGE" />
+               <category android:name="android.intent.category.HEALTH_PERMISSIONS" />
+           </intent-filter>
+        </activity-alias>
    </application>
 </manifest>
```

## 2. コードの変更
今回はGoogleFitのみでの動作をギリギリまで残すために切り分けてます。

```dart
final types = [HealthDataType.STEPS];
// ヘルスコネクトを使用するため
// Healthのバージョンが11以上であれば不要
await Health().configure(useHealthConnectIfAvailable: true);

final isHealthConnectAvailable = await AppCheck().isAppEnabled("com.google.android.apps.healthdata")
    .catchError((error, stackTrace) => false);
final isGoogleFitAvailable = !isHealthConnectAvailable && (await AppCheck().isAppEnabled("com.google.android.apps.fitness")
    .catchError((error, stackTrace) => false));

if (isHealthConnectAvailable || isGoogleFitAvailable) {
    final hasPermission = await Health().hasPermissions(types) ?? false;
    if (!hasPermission) {
        bool result = await Health().requestAuthorization(types);
        if (!result) return;
    }

    // アクティビティの権限まわり
    bool hasActivityRecognitionPermission = await Permission.activityRecognition.isGranted;
    if (!hasActivityRecognitionPermission) {
        hasActivityRecognitionPermission = (await Permission.activityRecognition.request()).isGranted;
        if (!hasActivityRecognitionPermission) return;
    }

    print("Healthの情報が取得可能になりました！！");
    if (isGoogleFitAvailable) {
        // GoogleFitApiがサポート終了する旨を伝える処理
    }
    // 以下お好きなロジックを実装
} else {
// HealthConnectのインストールを促す
const url = "https://play.google.com/store/apps/details?id=com.google.android.apps.healthdata";
    if (await canLaunchUrlString(url)) {
        await launchUrlString(url, mode: LaunchMode.externalApplication);
    }
}
```

## 3. HealthConnectAPIの利用申請
:::message
従来のGoogleFitAPIとは違い、Googleアカウントの認証を必要としないため、
OAuthの申請は不要になりました！神！
:::
申請方法などはこちらに詳しく書いてあります。
https://developer.android.com/health-and-fitness/guides/health-connect/publish/declare-access?hl=ja

# 最後に
GoogleFitAPIのサポート終了が残り4ヶ月となりましたので、そろそろ重い腰を上げて修正する必要があったので作成しました。
従来のGoogleFitAPIに比べ手順が少なく済み、個人的にはありな変更かなとも思っています。
移行する際のご参考になれば幸いです。
もしこの記事で間違えている部分などあればコメントいただけますと助かります。