この課題を実現するためのLogstashの設定は以下の通りです。Logstashを用いてAWS S3上のファイルを処理する際、特にファイル名のパターンに基づいて処理を分散させることがポイントになります。以下の設定例では、`input`セクションでファイル名のパターンによる振り分けを実現し、`filter`および`output`セクションで共通の処理を行う方法を示します。

### サーバAの設定 (奇数の場合)

```yaml
input {
  s3 {
    access_key_id => "あなたのアクセスキー"
    secret_access_key => "あなたのシークレットキー"
    bucket => "あなたのバケット名"
    prefix => "あなたのプレフィックス"
    backup_to_bucket => "処理済みのファイルを移動するバケット名"
    backup_add_prefix => "backup/"
    additional_settings => {
      force_path_style => true
      follow_redirects => false
    }
    codec => "plain"
    type => "s3"
    interval => 60 # ファイルチェック間隔
    exclude_pattern => ".*[02468]\.csv$" # 偶数を除外
  }
}

filter {
  csv {
    separator => ","
    columns => ["column1", "column2", "column3"]
  }
}

output {
  elasticsearch {
    hosts => ["http://あなたのElasticsearchのホスト:9200"]
    index => "あなたのインデックス名"
    document_type => "_doc"
  }
}
```

### サーバBの設定 (偶数の場合)

```yaml
input {
  s3 {
    access_key_id => "あなたのアクセスキー"
    secret_access_key => "あなたのシークレットキー"
    bucket => "あなたのバケット名"
    prefix => "あなたのプレフィックス"
    backup_to_bucket => "処理済みのファイルを移動するバケット名"
    backup_add_prefix => "backup/"
    additional_settings => {
      force_path_style => true
      follow_redirects => false
    }
    codec => "plain"
    type => "s3"
    interval => 60 # ファイルチェック間隔
    exclude_pattern => ".*[13579]\.csv$" # 奇数を除外
  }
}

filter {
  csv {
    separator => ","
    columns => ["column1", "column2", "column3"]
  }
}

output {
  elasticsearch {
    hosts => ["http://あなたのElasticsearchのホスト:9200"]
    index => "あなたのインデックス名"
    document_type => "_doc"
  }
}
```

### 注意点
- `access_key_id` と `secret_access_key` は、AWSの認証情報を使用します。これらの情報はセキュアな方法で管理する必要があります。
- `bucket` と `prefix` は、S3上のバケット名とファイルが置かれているプレフィックス（フォルダのパス）を指定します。
- `exclude_pattern` を使って、ファイル名のパターンに基づいて処理するファイルをフィルタリングしています。サーバAでは偶数を示す数字で終わるファイル名を除外し、サーバBでは奇数を示す数字で終わるファイル名を除外しています。
- `backup_to_bucket` と `backup_add_prefix` を設定することで、処理後のファイルを別のバケットやプレフィックスに移動させることが可能です。これにより、処理済みのファイルを容易に識別できます。
- Elasticsearchの`

hosts`、`index`、そして必要に応じて`document_type`を適宜設定してください。

この設定を各サーバに適用することで、ファイル名の最後の数字に基づいて処理を振り分け、S3上の膨大なファイル群を効率的に処理することが可能になります。
