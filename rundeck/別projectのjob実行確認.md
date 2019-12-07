# 別projectのjob実行確認

あるプロジェクトにしかないnodeは、
他のプロジェクトにjob登録した場合、実行できるのか？

例えば・・・
project1にはなく、project2にしかないrundeck_node2で実行するjobを
project1に登録する場合。

## ネットワーク作成

docker network create net_rundeck

## rundeckコンテナ作成
docker pull jordan/rundeck:2.11.4

docker run -it -d --privileged --network net_rundeck -p 4440:4440 -p 122:22 -e EXTERNAL_SERVER_URL=http://localhost:4440 --name rundeck -h rundeck -t jordan/rundeck:2.11.4

## rundeckプロジェクト作成
project1
project2

## job作成
- project1
  - project1_job1_test
- project2
  - project2_job1_test

ローカルで実行チェック

## node用コンテナ作成
docker run -it -d --privileged --network net_rundeck -h rundeck_node1 --name rundeck_node1 -p 222:22 centos /sbin/init
docker run -it -d --privileged --network net_rundeck -h rundeck_node2 --name rundeck_node2 -p 322:22 centos /sbin/init

## sshを許可する

yum install openssh-server

## /etc/ssh/sshd_config
ファイル終端に追加
```
Port 22
PermitRootLogin no
PasswordAuthentication no
PermitEmptyPasswords no
```
ファイル前にPasswordAuthenticationはyesの設定になっているので、コメントアウトする。

### ユーザー追加
useradd -m ruser
passwd ruser
:ruserで設定

### rundeckの公開鍵をコピーして設置

#### キーの場所
/var/lib/rundeck/.ssh/id_rsa.pub

#### nodeサーバに公開キー設置

```
sudo su - ruser
mkdir .ssh
chmod 700 .ssh
echo '{rundeck公開キー}' > .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
```

## project1からproject2:project2_job1_testを実行

## 結果
```
16:53:06	rundeck_node1	 1. Command	project1_job1_test
16:53:07	localhost	 2. project2_job1_test	No nodes matched for the filters: NodeSet{includes={name=rundeck_node2, dominant=false, }}
16:53:07			Execution failed: 8 in project project1: [Workflow result: , step failures: {2=NoMatchedNodes: No nodes matched for the filters: NodeSet{includes={name=rundeck_node2, dominant=false, }}}, status: failed]
```

project1にはrundeck_node2はないと言われる。

### Override Node Filters?のRun a項目をnode stepにする
```
16:57:29	rundeck_node1	 1. Command	project1_job1_test
16:57:30	localhost	 2. project2_job1_test	No nodes matched for the filters: NodeSet{includes={name=rundeck_node2, dominant=false, }}
16:57:30			Failed dispatching to node localhost: com.dtolabs.rundeck.core.execution.workflow.steps.StepException: No nodes matched for the filters: NodeSet{includes={name=rundeck_node2, dominant=false, }}
16:57:30			Execution failed: 11 in project project1: [Workflow result: , step failures: {2=Dispatch failed on 1 nodes: [localhost: Unknown: com.dtolabs.rundeck.core.execution.workflow.steps.StepException: No nodes matched for the filters: NodeSet{includes={name=rundeck_node2, dominant=false, }}]}, Node failures: {localhost=[Unknown: com.dtolabs.rundeck.core.execution.workflow.steps.StepException: No nodes matched for the filters: NodeSet{includes={name=rundeck_node2, dominant=false, }}]}, status: failed]
```
こちらも同等のエラーメッセージが出る。

### project1のnodeにrundeck_node2を追加してみる。

/var/rundeck/projects/project1/etc/resource.xml

以下を追加
```
  <node name="rundeck_node2" description="rundeck_node2" tags="" hostname="rundeck_node2" osArch="amd64" osFamily="unix" osName="Linux" osVersion="4.9.184-linuxkit" username="ruser"/>
```

実行結果
```
16:59:26	rundeck_node1	 1. Command	project1_job1_test
16:59:27	rundeck_node2	 1. Command	project2_job1_test
```

問題なく実行できた。

## まとめ
プロジェクトから別プロジェクトのジョブを実行する際、実行側のプロジェクトにnodeを追加しないと実行で失敗する。
