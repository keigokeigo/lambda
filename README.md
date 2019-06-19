# AWS Lambda のお勉強

* 2019-06-19
* ここを参考に。 [初めてのAWS Lambda　～AWS Lambdaで始めるイベントドリブンアプリケーション (1/5)：CodeZine（コードジン）](https://codezine.jp/article/detail/8446)

# Lambda の特徴

* インフラ管理が不要
  * EC2起動して、いろいろインストールしたり、監視する必要がない。
  * サイジングも不要。
  * 耐障害性を考慮する必要もなし。
  * LambdaファンクションとしてUploadするだけ。
* オートスケール
* 料金体系
  * 100ms の時間単位
  * AWSの従量課金の特徴がより色濃く反映されている。

# Lambdaファンクション

* コンソール上での編集、 OR ZIPで固めてUploadが可能。
* イベントソース（S3, Kinesis, DynamoDBなど）がトリガーとなり、自動で必要に応じてEC2インスタンスを起動する。
* AWS SDKやAWS CLIからも操作可能。

# プログラミングモデル

* Node.jsが基本。でもいろんな言語がサポートされている。
* JSONでイベント内容の詳細を、Lambdaが受け取れる。
  * ARN（Amazon Resource Name、AWSリソースを一意に識別するためのID）や実際にPUTされたオブジェクトのキーなど。
  * カスタムイベントを発行することも。その際は任意のデータを渡せる。
* ステートレス
  * 関数が処理終了したら何も残らない。
  * /tmp領域もクリアされる。
  * 永続化したステートを保持したいなら、S3やDynamoDBに保存すること。

# イベントソース

## PUSHモデル

* S3にファイルがPUTされたケースなど。
  * Invocation（＝祈り、発動）Role はS3。
  * Execution RoleはLambda
* 順序は保証されない。

## PULLモデル

* DynamoDB や Kinesis と連携する場合。
* 順序は保証される。
* Job Queueみたい。

## 注意

* InvocationとExecutionは同じRegionである必要がある。

# IAMでの定義

## Invocation Role

* Access-Policy と、Trust-Policyを設定する必要がある。
  * Access-Policy － Lambdaにイベント発行できる権限を宣言。
    * "Action": [ "lambda:InvokeAsync" ]
  * Trust-Policy － 誰がそのロールを引き受けられるか？
    * 特定のBucketののみでイベントが発行できるよう制限
      ```
      "Action": "sts:AssumeRole",
      "Condition": {
          "StringEquals":{
            "sts:ExternalId": "arn:aws:s3:::bucket name"
          }
      }
      ```

## Execution Role

* 必要なAWSリソースへのアクセスを許可するIAMロールのこと
* こちらも Access-Policy と Trust-Policy の定義が必要

* Access-Policy
  * Cloudwatchログへの書き込み権限は必須
  ```
  {
      "Statement": [
          {
              "Action": [
                  "logs:*"
              ],
              "Effect": "Allow",
              "Resource": "arn:aws:logs:*:*:*"
          }
      ]
  }
  ```
* Trust-Policy
  * Lambda自身が「そのロールを引き受けられるよう」定義する。

# モニタリング

* 以下をCloudwatchに保存される。
  * リクエスト数
  * 遅延
  * 可用性
  * エラー率

* LambdaコンソールのDashboardからも閲覧可能。
* LogもCloudWatch Logへ保存される。
  * 任意のメッセージをログへ出力可能。


# 料金体系

* 100ms ごとで単価
  * メモリ容量に応じて単価が違う。
* 単価 x リクエスト数 ＝ コスト
* 月間100万リクエストまでは無料


# ユースケース

## 画像のリサイズ、サムネイル生成

## ログの監査と通知

* AWS APIのコールや、AWSコンソールでの操作を監査ログとして保存。→ AWS CloudTrail へ保存される。
* そのイベントを拾って、怪しい行動があったら、LambdaからPUSH通知

## 動画のアップロード

* S3に動画アップされたことをトリガーに、
* Amazon Elastic Transcoder を利用し、デバイス向け動画ファイルが自動生成されるようにする。



# やってみた

* [サンプル Amazon Simple Storage Service 関数コード - AWS Lambda](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/with-s3-example-deployment-pkg.html#with-s3-example-deployment-pkg-nodejs) を参考にした。

## はまったところ

* IAM の設定、大事

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::*"
      ]
    }
  ]
}

```

## Node.js のバージョン

* Lambdaコンソール上で、デフォルト 10.1 が選択されるようになっているが、[サンプル Amazon Simple Storage Service 関数コード - AWS Lambda](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/with-s3-example-deployment-pkg.html#with-s3-example-deployment-pkg-nodejs) のコードは 8.1 でないと動かない。注意。

## Nodeモジュール

* `gm` やら、 `imagemagick` やら、自前でUPしないでいいものと思っていた。
* ローカルのプロジェクトディレクリで `npm install gm async` とかして、そこのCurrent DIRでZIPファイル作ってUPする必要があるのを知った。 → ZIPでツリー構造（ `npm_module` DIR含む）をUPしなければならない。


