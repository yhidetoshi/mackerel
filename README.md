# mackerel


# mackerel
mackerel検証

#### agentインストール
##### Amazon Linux

- Agentインストール
  - `# curl -fsSL https://mackerel.io/file/script/amznlinux/setup-all-yum.sh | MACKEREL_APIKEY='XXXX' sh`
- 自動起動設定
  - `# chkconfig mackerel-agent on`

#### Nginxのプロセスを監視
- rpmパッケージの場合
  - `# yum -y install mackerel-check-plugins`


- `/etc/mackerel-agent/mackerel-agent.conf`
```
apikey = "XXX"

[plugin.checks.check_nginx_worker]
command = "check-procs -p nginx -W 2 -C 2 --user nginx"
```

#### Fluentd(td-agent)を監視

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

#### Redisの監視

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
