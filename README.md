# mackerel

# mackrel-CLIのセットアップ

- Mac
```
$ brew tap mackerelio/mackerel-agent
$ brew install mkr
$ go get github.com/mackerelio/mkr
```
- `export MACKEREL_APIKEY=<API key>`

##### コマンド参照
- APIドキュメント(公式)
  - https://mackerel.io/ja/api-docs/
- `$ mkr --help`

```
NAME:
   mkr - A CLI tool for mackerel.io
USAGE:
   mkr [global options] command [command options] [arguments...]
VERSION:
   0.21.0 (rev:abe6e7e)
AUTHOR:
   Hatena Co., Ltd.
COMMANDS:
     status       Show the host
     hosts        List hosts
     create       Create a new host
     update       Update the host
     throw        Post metric values
     fetch        Fetch latest metric values
     retire       Retire hosts
     services     List services
     monitors     Manipulate monitors
     alerts       Retrieve/Close alerts
     dashboards
     annotations  Manipulate graph annotations
     help, h      Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --conf value     Config file path (default: "/Users/yajima/Library/mackerel-agent/mackerel-agent.conf")
   --apibase value  API Base (default: "https://api.mackerelio.com")
   --help, -h       show help
   --version, -v    print the version
```


### GETリクエスト
- GET
  - ホスト一覧を取得
    - サービスに紐づくホスト一覧を取得(退役ホストを除く)
        - `$ mkr hosts --service <SERVICENAME>`
        - `$ mkr hosts --service Beaconnect-Prd | jq '.[].name'`
  - ホストが(working|standby|poweroff|maintenance)状態のインスタンスを取得
      - `mkr hosts --status working`
      - `mkr hosts --status standby`
      - `mkr hosts --status poweroff`
      - `mkr hosts --status maintenance`
  - Serviceの一覧を取得
      - `mkr services`
  - 監視設定の一覧を取得
      - `mkr monitors`
  - アラート(現在出ている)の一覧を取得
      -  `mkr alerts`

### POSTリクエスト

- あるホストを退役させる
  - `mkr retire <HOSTID>`


# Mackerelの監視設定をgitで管理する

- 現在の監視状態をjsonでpullする
  - `mkr monitors pull`
- git管理にして、monitors.jsonを編集
- mkr で 反映する
  - `mkr monitors push`
```
(Response)
mkr monitors push
      info Update a rule.
{
  "id": "XXXX",
  "name": "CPU %",
  "memo": "CPU使用率監視",
  "type": "host",
  "metric": "cpu%",
  "operator": ">",
  "warning": 85,
  "critical": 90,
  "duration": 5,
  "scopes": [
    "Hoge-stg"
  ]
},
```

- Mackerelとローカルファイルとの差分を確認する
`$ mkr monitors diff`

```
Summary: 1 modify, 0 append, 0 remove

 {
   "critical": 90,
   "duration": 5,
   "memo": "CPU使用率監視",
   "metric": "cpu%",
   "name": "CPU %",
   "operator": ">",
   "scopes": [
     "Hoge-stg"
   ],
   "type": "host",
-  "warning": 85
+  "warning": 80
 }
```
- dry-run実行
  - `mkr monitors push --dry-run`

# agentインストール & プロセス監視
## Amazon Linux

- Agentインストール
  - `# curl -fsSL https://mackerel.io/file/script/amznlinux/setup-all-yum.sh | MACKEREL_APIKEY='XXXX' sh`
- 自動起動設定
  - `# chkconfig mackerel-agent on`

## Nginxのプロセスを監視
- rpmパッケージの場合
  - `# yum -y install mackerel-check-plugins`


- `/etc/mackerel-agent/mackerel-agent.conf`
```
apikey = "XXX"

[plugin.checks.check_nginx_worker]
command = "check-procs -p nginx -W 2 -C 2 --user nginx"
```

## Fluentd(td-agent)を監視

`# ps -ef | grep td-agent`
```
td-agent  4936     1  0 Jun26 ?        00:00:00 /usr/lib64/fluent/ruby/bin/ruby /usr/sbin/td-agent --group td-agent --log /var/log/td-agent/td-agent.log --daemon /var/run/td-agent/td-agent.pid
td-agent  4939  4936  0 Jun26 ?        00:44:38 /usr/lib64/fluent/ruby/bin/ruby /usr/sbin/td-agent --group td-agent --log /var/log/td-agent/td-agent.log --daemon /var/run/td-agent/td-agent.pid
```

`# which check-procs`
```
/usr/bin/check-procs
```

- マカレルが実行するコマンドでプロセスがひっかける事ができるかを確認
  - `# /usr/bin/check-procs --pattern /usr/lib64/fluent/ruby/bin/ruby --user td-agent`
```
Procs OK: Found 2 matching processes; cmd //usr/lib64/fluent/ruby/bin/ruby/; user /td-agent/
```
→ hitしたので、監視設定を入れる


`# check-procs -p /usr/lib64/fluent/ruby/bin/ruby -W 3 -C 3 --user td-agent`
```
Procs CRITICAL: Found 2 matching processes; cmd //usr/lib64/fluent/ruby/bin/ruby/; user /td-agent/
```

```
[plugin.checks.check_check_td-agent]
command = "check-procs -p /usr/lib64/fluent/ruby/bin/ruby -W 2 -C 2 --user td-agent"
```

## Redisの監視

`# ps -ef | grep redis`
```
redis     2483     1  0 Aug13 ?        00:00:23 /usr/sbin/redis-server /etc/redis.conf
```

`# check-procs -p /usr/sbin/redis-server -W 1 -C 1 --user redis`
```
Procs OK: Found 1 matching processes; cmd //usr/sbin/redis-server/; user /redis/
```
```
[plugin.checks.check_redis]
command = "check-procs -p /usr/sbin/redis-server -W 1 -C 1 --user redis"
```

## ログ監視(1分間)
- `which check-log`
> /usr/local/bin/check-log

- `/etc/mackerel-agent/mackerel-agent.conf`
```
[plugin.checks.jenkins-log-ERROR]
command = "/usr/local/bin/check-log --file /var/log/jenkins/jenkins.log --pattern 'SUCCESS' --warning-over 1 --critical-over 10 --return"
```
- [オプション説明]
```
/usr/local/bin/check-log: Command-Path
--warning-over 1: 2回以上でアラート検知　- 
--return: 検知した行を通知時に表示させる
```

- 検知させたログ（サンプル）
```
11 29, 2017 2:44:04 午前 hudson.model.Run execute
情報: get_instanceInfo #18 main build action completed: SUCCESS
```


### 退役ホストを戻す
```
service mackerel-agent stop
cd /var/lib/mackerel-agent
mv id /tmp/
service mackerel-agent start
diff -su /tmp/id id   # 違うと出ていればOK
```

### エージェントをアンインストールする
```
$ sudo yum erase mackerel-agent
```

# AWSのマネージドサービスを監視する

- AWSとマカレルを連携する方法
  - `https://mackerel.io/ja/docs/entry/integrations/aws`
    - IAMロールに必要なサービスのReadOnly権限を付与する

- ELBとRDSをNameタグで絞る場合は以下の通り
  - `Name:hoge-db,Name:hoge-lb`

- 死活監視(マカレルサイトの引用)
  - mackerel-agentをインストール・起動 したホストに対しては、死活監視が自動的に行なわれます。
  - 死活監視はmackerel-agentからのメトリックの定期投稿を監視しています。
  - 一定期間この投稿がない場合、Mackerelはそのホストに異常が発生したと判断してアラートを発生させます。
    - 約7分間データの投稿がないと死活監視にひっかかりました。
