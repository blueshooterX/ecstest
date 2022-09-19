# 参考にしたページ

### ECS
https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/ECS_AWSCLI_Fargate.html
https://docs.aws.amazon.com/cli/latest/reference/ecs/index.html#cli-aws-ecs

### NATインスタンス
https://www.cloudnotes.tech/entry/nat-instanse-amazonlinux2

### iptables
https://virment.com/iptables-setting-example/?utm_source=pocket_mylist

# ECS cliコマンド

クラスタ作成
```
aws ecs create-cluster --cluster-name fargate-cluster
```
タスク定義の登録
```
aws ecs register-task-definition --cli-input-json file://.\xxxx.json
```
タスク定義を表示
```
aws ecs list-task-definitions
```
サービスを作成
※クラスター名称・サービス名称・XXXX部分は適時修正。
```
aws ecs create-service --cluster fargate-cluster --service-name fargate-service --task-definition XXXX:1 --desired-count 1 --launch-type "FARGATE" --network-configuration "awsvpcConfiguration={subnets=[subnet-XXXX],securityGroups=[sg-XXXX]}"
```

サービスをリスト表示する
```
aws ecs list-services --cluster fargate-cluster
```
実行中のサービスを記述する
```
aws ecs describe-services --cluster fargate-cluster --services fargate-service
```
タスクリストを取得
```
aws ecs list-tasks --cluster fargate-cluster
```
タスクの情報を取得(tasks部分の指定はタスクのidでもARNでも良い模様)
```
aws ecs describe-tasks --cluster fargate-cluster --tasks XXXXXXXXXXXXXXXXXXXXXX
```

# NATインスタンスの作成
## iptablesのインストール・設定
インストール
```
yum -y install iptables-services
```
iptablesの定義は全て空にしておく
```
iptables -F
```
OS設定でipフォワーディングを有効にしておく
```
echo 1 > /proc/sys/net/ipv4/ip_forward
echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf
```
iptablesでprivate⇒外部の通信はipマスカレード(アクセス元アドレスの動的変換)で出れるようにしておく
```
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
iptables設定保存
```
service iptables save
```
iptablesのサービス有効化
```
systemctl start iptables
systemctl enable iptables
```
iptablesの実行状況確認
```
systemctl status iptables
```
## EC2設定で送信元/送信先 チェックの無効化
* 管理コンソール⇒EC2⇒インスタンスの「アクション」⇒ネットワーク⇒「ソース/宛先チェックを変更」⇒「停止」にチェックして「保存」
* ※NATインスタンスのため、サーバ自身のIPアドレス宛以外のパケットも受信するための設定。
## セキュリティグループ
* private ネットワークからNATインスタンスへのアクセスはSecurityGroupで開けておく。
## NAT設定
```
iptables -t nat -A PREROUTING -d 10.0.64.230 -p tcp --dport 8001 -j DNAT --to 10.0.50.84:80 -m comment --comment "PortMapping"
service iptables save
```
# iptablesコマンドメモ
NAT設定状況の確認
```
iptables -t nat -L -n
```
NAT設定状況の確認(行番号付き)
```
iptables -t nat -L -n --line-numbers
```
NAT設定を全部消す方法(MASQUERADEも消える)
```
iptables -t nat -F
```
特定の文字列を含む設定を削除する
```
iptables-save | grep -v "PortMapping" | iptables-restore
```

# jq使用例
list-tasksの結果からtaskのARNリストを作成
```
type .\task_list.json | jq -r '.taskArns[]'
```
describe-task の結果からアドレスを抽出
```
type .\task_desc.json | jq -r '.tasks[].attachments[].details[] | select( .name == \"privateIPv4Address\" ) | .value '
```

# ECR

https://qiita.com/yumatsud/items/0acad37d10a6782ecec8#amazon-ecr
https://nogson2.hatenablog.com/entry/2020/10/14/191850
