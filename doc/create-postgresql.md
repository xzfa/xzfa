
# 一、PostgreSQL是什么？
```
PostgreSQL是一个功能强大的开源对象关系数据库管理系统(ORDBMS)。 用于安全地存储数据; 支持最佳做法，并允许在处理请求时检索它们。

PostgreSQL(也称为Post-gress-Q-L)由PostgreSQL全球开发集团(全球志愿者团队)开发。 它不受任何公司或其他私人实体控制。 它是开源的，其源代码是免费提供的。

PostgreSQL是跨平台的，可以在许多操作系统上运行，如Linux，OS X和Microsoft Windows等。
```
 
# 二、PostgreSQL特点
```
跨平台
支持文本、图像、视频、声音等
并提供C/C++、Java、Perl、Python、Ruby放数据库连接(ODBC)的编程接口。
支持SQL的许多功能，例如复杂的SQL查询，子查询，外键，触发器，视图，视图，进程并发控制(MVCC)、异步复制。
在PostgreSQL中，表可以设置为从“父”表继承其特征。
```
PostgreSQL是第一个实现多版本并发控制（MVCC）功能的数据库管理系统，甚至在Oracle之前。MVCC功能在Oracle中称为快照隔离。

PostgreSQL是一个通用的对象 - 关系数据库管理系统。它允许您添加使用不同编程语言（如C / C ++，Java等）开发的自定义函数。

PostgreSQL旨在实现可扩展性。在PostgreSQL中，您可以定义自己的数据类型，索引类型，函数语言等。如果您不喜欢系统的任何部分，您可以随时开发自定义插件以增强它以满足您的要求，例如，添加新的优化。

如果您需要任何支持，可以使用活跃的社区来提供帮助。您可以随时找到PostgreSQL社区的答案，以了解使用PostgreSQL时可能遇到的问题。许多公司在您需要时提供商业支持服务。


# create postgresql according to helm

*  **[helm install postgresql](https://artifacthub.io/packages/helm/bitnami/postgresql)**
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-release bitnami/postgresql
```

**安装后给出的信息**
```sh
NAME: my-release
LAST DEPLOYED: Sun Jul  3 13:47:10 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 11.6.7
APP VERSION: 14.4.0

** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    my-release-postgresql.default.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default my-release-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To connect to your database run the following command:

    kubectl run my-release-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:14.4.0-debian-11-r0 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host my-release-postgresql -U postgres -d postgres -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/my-release-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432

```

密码：   `Td65q2fVEC`

将pg的service暴露出来
```sh
k edit service/my-release-postgresql
```
将type从cluester修改为NodePort即可

进入postgresql的pod
```sh
kubectl exec -it my-release-postgresql-0  -- bash
```

进入容器后发现没有权限，现在的暂时的解决方案如下
```yaml
        securityContext:
          runAsUser: 0
```
将进入容器的权限更改为root，之后再进入容器就拥有权限了。

在我的本机写一个脚本pg.sh
```sh
     #!/bin/bash

#必须加这一句不然kubectl的相关操作不会执行

filename="`date +%F`_bak.sql" 
#备份数据脚本
cat > /home/xzfa/workfiles/exportPG.sh <<EOF
#!/bin/bash
export PGUSER=postgres
export PGPASSWORD='Td65q2fVEC'
export PGHOST=127.0.0.1
export PGPORT=5432
#备份整个集群库中的数据
pg_dumpall -a > ${filename}
#备份整个集群库包含建库建表操作
#pg_dumpall > ${filename}
EOF

#给执行权限
chmod +x /home/xzfa/workfiles/exportPG.sh

#将服务器上的备份脚本复制到对应容器中去
kubectl cp /home/xzfa/workfiles/exportPG.sh my-release-postgresql-0:/exportPG.sh
#在容器外执行该脚本
kubectl exec -it my-release-postgresql-0  -- /exportPG.sh
#将备份后的数据文件复制到容器外
kubectl cp my-release-postgresql-0:${filename} /home/xzfa/workfiles/${filename}
                                                                          
```

运行脚本，进入pg的pod中查看,生成了`2022-07-03_bak.sql`以及`exportPG.sh`

2022-07-03_bak.sql为数据库备份文件、exportPG.sh是备份脚本，可以单独在pod中执行。
```sh
root@my-release-postgresql-0:/# ls
2022-07-03_bak.sql  boot			docker-entrypoint-preinitdb.d  home   media  proc  sbin  tmp
bin		    dev				etc			       lib    mnt    root  srv	 usr
bitnami		    docker-entrypoint-initdb.d	exportPG.sh		       lib64  opt    run   sys	 var
```



# 定时备份
编辑crontab文件，添加定时任务。
```sh
$ vi /etc/crontab

0  0    * * 0   root    ./home/xzfa/workfiles/PG.sh

```
重新启动crontab
```sh
0  0    * * 0   xzfa    ./home/xzfa/workfiles/PG.sh
```

























































# navicat破解
* https://rlds.tk/