このシナリオでは、複数のサーバ上でLogstashを実行して、AWS S3からのファイルを処理する方法について検討します。サーバの負荷を最小限に抑えつつ、S3との通信を効率的に行う必要があります。オンプレミス環境での実行を前提としているため、AWS上のマネージドサービスは使用しません。

以下の構成案を提案します。

### 1. Logstash設定ファイルの構成

各サーバのLogstash設定ファイルはほぼ同じ内容になりますが、`input`セクションでファイルをフィルタリングする条件を変えます。ファイル名の最終文字が奇数か偶数かによって、処理するサーバを分ける設定です。

#### サーバA（奇数のファイル名を処理）

```ruby
input {
  file {
    path => "/path/to/s3/mounted/folder/*"
    sincedb_path => "/path/to/sincedb_file_A"
    mode => "read"
    file_completed_action => "log_and_delete"
    file_completed_log_path => "/path/to/completed_files_A.log"
    exclude => ["*0.csv", "*2.csv", "*4.csv", "*6.csv", "*8.csv"]
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
    hosts => ["http://localhost:9200"]
    index => "files_index"
  }
}
```

#### サーバB（偶数のファイル名を処理）

```ruby
input {
  file {
    path => "/path/to/s3/mounted/folder/*"
    sincedb_path => "/path/to/sincedb_file_B"
    mode => "read"
    file_completed_action => "log_and_delete"
    file_completed_log_path => "/path/to/completed_files_B.log"
    exclude => ["*1.csv", "*3.csv", "*5.csv", "*7.csv", "*9.csv"]
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
    hosts => ["http://localhost:9200"]
    index => "files_index"
  }
}
```

### 2. 処理フロー

1. **起動**: 各サーバでLogstashを起動します。
2. **ファイル取得**: `input`セクションで指定された条件に基づき、S3からファイルを取得します。S3のファイルをオンプレミスのサーバにマウントするか、あるいは定期的に同期する仕組みが必要になります。AWSサービスが使えないため、サードパーティのツールやスクリプトで同期を行う必要があります。
3. **データ処理**: `filter`セクションでファイル内容を処理します。例ではCSVファイルを想定しています。
4. **データ出力**: 処理結果を`output`セクションでElasticsearchに出力します。

この構成案では、サーバ間で処理を分散し、各サーバが特定の条件にマッチするファイルのみを処理するように設定しています。これにより、サーバの負荷分散とS3との通信量の削減を図ります。また、`file_completed_action`オプションを使用して処理が完了したファイルをログに記録し、削除することで、処理の重複を避けることができます。
