# 参考にしたページ

### ECS
https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/ECS_AWSCLI_Fargate.html
https://docs.aws.amazon.com/cli/latest/reference/ecs/index.html#cli-aws-ecs

### NATインスタンス
https://www.cloudnotes.tech/entry/nat-instanse-amazonlinux2

### iptables
https://virment.com/iptables-setting-example/?utm_source=pocket_mylist

# ECScliコマンド 

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
```
aws ecs create-service --cluster fargate-cluster --service-name nginx-service --task-definition nginx-task:2 --desired-count 1 --launch-type "FARGATE" --network-configuration "awsvpcConfiguration={subnets=[subnet-0a018a1dfc1f26537],securityGroups=[sg-0291e9e3021339168]}" --region us-east-1
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
ip6tables -F
```
OS設定でipフォワーディングを有効にしておく(ipv4)
```
echo 1 > /proc/sys/net/ipv4/ip_forward
echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf
```

ipv6フォワーディング
* RouterAdvertisementを受け取る設定を強制
* DefaultGatewayは指定しない。
```
echo IPV6FORWARDING=yes >> /etc/sysconfig/network
echo net.ipv6.conf.eth0.accept_ra=2 >> /etc/sysctl.conf
```

iptablesでprivate⇒外部の通信はipマスカレード(アクセス元アドレスの動的変換)で出れるようにしておく
```
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
ip6tables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```
iptables設定保存
```
service iptables save
service ip6tables save
```
iptablesのサービス有効化
```
systemctl start iptables
systemctl start ip6tables
systemctl enable iptables
systemctl enable ip6tables
```
iptablesの実行状況確認
```
systemctl status iptables
systemctl status ip6tables
```
## EC2設定で送信元/送信先 チェックの無効化
* 管理コンソール⇒EC2⇒インスタンスの「アクション」⇒ネットワーク⇒「ソース/宛先チェックを変更」⇒「停止」にチェックして「保存」
* ※NATインスタンスのため、サーバ自身のIPアドレス宛以外のパケットも受信するための設定。
## セキュリティグループ
* private ネットワークからNATインスタンスへのアクセスはSecurityGroupで開けておく。
## NAT設定の例
```
iptables -t nat -A PREROUTING -d 10.0.0.5 -p tcp --dport 8001 -j DNAT --to 10.0.135.174:80 -m comment --comment "PortMapping"
ip6tables -t nat -A PREROUTING -d 2600:1f18:1bc7:8f00:65b0:adc4:6e38:7fd0 -p tcp --dport 8001 -j DNAT --to-destination [2600:1f18:1bc7:8f01:9e25:e543:8a42:24e9]:80 -m comment --comment "PortMapping"
service iptables save
service ip6tables save
```
# iptablesコマンドメモ
NAT設定状況の確認
```
iptables -t nat -L -n
ip6tables -t nat -L -n
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
ip6tables-save | grep -v "PortMapping" | ip6tables-restore
```

# デバッグ用コマンド(ipv6)
```
ip -6 route
tcpdump icmp6 -n
```
ip fowardの状態確認
```
sysctl -a|grep forward
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

# SSMによるRunCommand

①実行対象のEC2にAmazonSSMManagedInstanceCoreポリシーのIAMロールを付ける。
②実行対象のSSM Agentをrestart
```
sudo systemctl restart amazon-ssm-agent
```
③SSMから認識されていることを確認
```
aws ssm describe-instance-information --region us-east-1
```
④コマンド実行
aws ssm send-command --document-name "AWS-RunShellScript" --instance-ids "i-0cc3c1f2c309db122" --parameters commands="ip a" --region us-east-1 --query "Command.CommandId"

aws ssm list-command-invocations --command-id "c6b61520-240c-47e3-9334-19a13db9d297" --details --region us-east-1

https://dev.classmethod.jp/articles/does-aws-runshellscript-require-shebang/
https://blog.takeyuweb.co.jp/entry/2020/08/14/005728

# ECR

https://qiita.com/yumatsud/items/0acad37d10a6782ecec8#amazon-ecr
https://nogson2.hatenablog.com/entry/2020/10/14/191850

# EBSマウント
sudo mount -t xfs -o nouuid /dev/nvme1n1p2 /mnt/data


===================================================
❯ aws ecs create-cluster --cluster-name fargate-cluster --region us-east-1
{
    "cluster": {
        "clusterArn": "arn:aws:ecs:us-east-1:463024324356:cluster/fargate-cluster",
        "clusterName": "fargate-cluster",
        "status": "ACTIVE",
        "registeredContainerInstancesCount": 0,
        "runningTasksCount": 0,
        "pendingTasksCount": 0,
        "activeServicesCount": 0,
        "statistics": [],
        "tags": [],
        "settings": [
            {
                "name": "containerInsights",
                "value": "disabled"
            }
        ],
        "capacityProviders": [],
        "defaultCapacityProviderStrategy": []
    }
}

❯ aws ecs register-task-definition --cli-input-json file://./task_def_httpd80.json --region us-east-1
{
    "taskDefinition": {
        "taskDefinitionArn": "arn:aws:ecs:us-east-1:463024324356:task-definition/httpd-task:1",
        "containerDefinitions": [
            {
                "name": "fargate-app",
                "image": "public.ecr.aws/docker/library/httpd:latest",
                "cpu": 0,
                "portMappings": [
                    {
                        "containerPort": 80,
                        "hostPort": 80,
                        "protocol": "tcp"
                    }
                ],
                "essential": true,
                "entryPoint": [
                    "sh",
                    "-c"
                ],
                "command": [
                    "/bin/sh -c \"echo '<html> <head> <title>Amazon ECS Sample App</title> <style>body {margin-top: 40px; background-color: #333;} </style> </head><body> <div style=color:white;text-align:center> <h1>Amazon ECS Sample App</h1> <h2>Congratulations!</h2> <p>Your application is now running on a container in Amazon ECS.</p> </div></body></html>' >  /usr/local/apache2/htdocs/index.html && httpd-foreground\"" 
                ],
                "environment": [],
                "mountPoints": [],
                "volumesFrom": []
            }
        ],
        "family": "httpd-task",
        "networkMode": "awsvpc",
        "revision": 1,
        "volumes": [],
        "status": "ACTIVE",
        "requiresAttributes": [
            {
                "name": "com.amazonaws.ecs.capability.docker-remote-api.1.18"
            },
            {
                "name": "ecs.capability.task-eni"
            }
        ],
        "placementConstraints": [],
        "compatibilities": [
            "EC2",
            "FARGATE"
        ],
        "requiresCompatibilities": [
            "FARGATE"
        ],
        "cpu": "256",
        "memory": "512",
        "registeredAt": "2022-09-17T10:34:02.865000+09:00",
        "registeredBy": "arn:aws:iam::463024324356:user/yoshi"
    }
}

❯ aws ecs list-task-definitions --region us-east-1
{
    "taskDefinitionArns": [
        "arn:aws:ecs:us-east-1:463024324356:task-definition/httpd-task:1"
    ]
}

❯ aws ecs create-service --cluster fargate-cluster --service-name fargate-service --task-definition httpd-task:1 --desired-count 1 --launch-type "FARGATE" --network-configuration "awsvpcConfiguration={subnets=[subnet-0a018a1dfc1f26537],securityGroups=[sg-0291e9e3021339168]}" --region us-east-1
{
    "service": {
        "serviceArn": "arn:aws:ecs:us-east-1:463024324356:service/fargate-cluster/fargate-service",
        "serviceName": "fargate-service",
        "clusterArn": "arn:aws:ecs:us-east-1:463024324356:cluster/fargate-cluster",
        "loadBalancers": [],
        "serviceRegistries": [],
        "status": "ACTIVE",
        "desiredCount": 1,
        "runningCount": 0,
        "pendingCount": 0,
        "launchType": "FARGATE",
        "platformVersion": "LATEST",
        "taskDefinition": "arn:aws:ecs:us-east-1:463024324356:task-definition/httpd-task:1",
        "deploymentConfiguration": {
            "deploymentCircuitBreaker": {
                "enable": false,
                "rollback": false
            },
            "maximumPercent": 200,
            "minimumHealthyPercent": 100
        },
        "deployments": [
            {
                "id": "ecs-svc/3699880533669224298",
                "status": "PRIMARY",
                "taskDefinition": "arn:aws:ecs:us-east-1:463024324356:task-definition/httpd-task:1",
                "desiredCount": 1,
                "pendingCount": 0,
                "runningCount": 0,
                "failedTasks": 0,
                "createdAt": "2022-09-17T10:41:36.247000+09:00",
                "updatedAt": "2022-09-17T10:41:36.247000+09:00",
                "launchType": "FARGATE",
                "platformVersion": "1.4.0",
                "networkConfiguration": {
                    "awsvpcConfiguration": {
                        "subnets": [
                            "subnet-0a018a1dfc1f26537"
                        ],
                        "securityGroups": [
                            "sg-0291e9e3021339168"
                        ],
                        "assignPublicIp": "DISABLED"
                    }
                },
                "rolloutState": "IN_PROGRESS",
                "rolloutStateReason": "ECS deployment ecs-svc/3699880533669224298 in progress."
            }
        ],
        "roleArn": "arn:aws:iam::463024324356:role/aws-service-role/ecs.amazonaws.com/AWSServiceRoleForECS",
        "events": [],
        "createdAt": "2022-09-17T10:41:36.247000+09:00",
        "placementConstraints": [],
        "placementStrategy": [],
        "networkConfiguration": {
            "awsvpcConfiguration": {
                "subnets": [
                    "subnet-0a018a1dfc1f26537"
                ],
                "securityGroups": [
                    "sg-0291e9e3021339168"
                ],
                "assignPublicIp": "DISABLED"
            }
        },
        "schedulingStrategy": "REPLICA",
        "createdBy": "arn:aws:iam::463024324356:user/yoshi",
        "enableECSManagedTags": false,
        "propagateTags": "NONE"
    }
}