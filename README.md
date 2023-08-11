# Fargate PID namespace共有テストサンプル

FargateでPID(process id) namespace共有が使えるようになったので、それをテストするためのサンプルです。


PID namespace共有+PTRACEを用いてweb, appコンテナを監視する事を想定し、web + app + 監視用コンテナ(を模したalpine) の3コンテナがあるタスク定義を作っています。

* Web
  * nginx-unprivilegedを使用。*.phpを単にphp-fpmに流すだけ。 [non-rootユーザ実行]
* App
  * php:fpm-alpineを使用。$_SERVERを出力する test.php だけあり。 [rootユーザ実行]
* 監視用コンテナ
  * ただのalpine。監視に使うことを想定してPTRACE capabilityを付与。[rootユーザ実行]

※監視用コンテナからの見え方の違いを確認するために、appは敢えてrootユーザ実行にしています

# 使い方
## 各コンテナをビルドしてECRにプッシュ
CDKとか面倒なので各々手動でやってください。

### ECRでリポジトリ作成
test-nginx, test-app, test-alpine という名前で作る。

### イメージのbuild & push
リポジトリをcloneし、各dirにて以下実行
```
docker build ./ -t test-***
docker tag test-***:latest xxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/test-***:latest

aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin xxxxx.dkr.ecr.ap-northeast-1.amazonaws.com
docker push xxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/test-***:latest
```

## ECSでタスク定義
task_definition.jsonをベースに使って定義。適当にmytaskとか。
imageのURIは自分のリポジトリ似合わせる。
ELBは不要、nginxコンテナ8080にhttpで直に繋げる。

特別なところは以下2点のみ
* pidMode=task
  * タスク内でPID namespace空間を共有
* alpineコンテナにSYS_PTRACEケーパビリティを設定
  * これで他コンテナを監視する想定(テストではstraceを使う)


## ECSでクラスターとサービス定義を作成、起動
さっきのタスクで作成。適当にmycluster, myserviceとか。

またECS Execを有効にする→[ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/ecs-exec.html)

準備ができたらサービスの必要数を1にしてタスクを起動する。

http://{test-webのIP}/test.php で接続確認。


## ECS Execでalpineコンテナに接続
```
aws ecs execute-command --cluster mytest --task {タスクID} --container test-alpine --interactive --command sh
```
なお、バグなのか仕様なのか、pidMode=task設定があると最後に起動したコンテナにしか接続できないようなので、test-alpineが最後に起動するよう指定している。




# pidMode:taskのテスト


上手く行けば下のように、PIDがコンテナで別れず同じ空間になっている。
```
/ # ps auxw
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    6 root      0:00 php-fpm: master process (/usr/local/etc/php-fpm.conf)
   12 root      0:00 /managed-agents/execute-command/amazon-ssm-agent
   18 101       0:00 nginx: master process nginx -g daemon off;
   24 82        0:00 php-fpm: pool www
   25 82        0:00 php-fpm: pool www
   31 root      0:00 /managed-agents/execute-command/amazon-ssm-agent
   42 root      0:00 sleep infinity
   58 101       0:00 nginx: worker process
   59 101       0:00 nginx: worker process
   69 root      0:00 /managed-agents/execute-command/amazon-ssm-agent
  102 root      0:00 /managed-agents/execute-command/ssm-agent-worker
  115 root      0:00 /managed-agents/execute-command/ssm-session-worker ecs-execute-command-049ad95608c7b1bab
  124 root      0:00 sh
  125 root      0:00 ps auxw
```


またphp-fpmにアタッチしてstraceが出来ることを確認。
```
~ # ps auxw
PID   USER     TIME  COMMAND
...
   24 82        0:00 php-fpm: pool www ←これにアタッチ
   25 82        0:00 php-fpm: pool www
...

~ # strace -p 24 -e trace=%file
#ここで存在しない/hoge.phpにアクセス
lstat("/var/www/html/hoge.php", 0x7ffe34210d90) = -1 ENOENT (No such file or directory)
stat("/var/www/html", {st_mode=S_IFDIR|S_ISVTX|0777, st_size=4096, ...}) = 0
stat("/var/www", {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
stat("/var", {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
stat("", 0x7ffe34212fb0)                = -1 ENOENT (No such file or directory)
```

`/proc/{pid}/root` で当該プロセスからの`/`が見えるので、他コンテナのファイルシステムにここからアクセスできる。

`/proc/{pid}/environ` で同様に環境変数が見えるはずだが、これは他コンテナ相手だと上手く動かない模様。


# 注意点
実際あり得る構成で、確認用の最低限の設定にしているので、セキュリティや運用性などは一切考えていない。
テスト用途以外に使わないように。
また使用後は停止しておく。

