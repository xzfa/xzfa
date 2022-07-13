# 执行pg备份的命令
```sh
pg_dumpall -U postgres  --host=keycloak-postgresql >> /tmp/pgdump/$(date +\"%Y_%m_%d_%H:%M\")
```
其中`-U`为pg用户名，这里为明文写在yaml中、密码使用secret保存在变量中
# 创建一个pv、pvc来存放cronjob生成的备份文件
pv.yaml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pg-pv
  labels:
    pv: pgdata
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: "/home/xzfa/workfiles/yaml/keycloak/pgdata"
    type: DirectoryOrCreate
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-path

```
```sh
k apply -f pv.yaml
```
pvc.yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pgdata
  namespace: default
spec:
  storageClassName: 'local-path'
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeMode: Filesystem

```
```sh
k apply -f pvc.yaml
```
查看pvc是否与pv绑定成功
```sh
xzfa@xiongzfa:~/workfiles/yaml/keycloak/pgdata$ k get pvc
NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-my-release-postgresql-0   Bound    pvc-48af2fe3-4a47-4c4e-abbb-0151f3786d12   8Gi        RWO            local-path     4d21h
data-keycloak-postgresql-0     Bound    pvc-9590c5c3-9f42-40d5-b2d4-a0f5bc255bb4   8Gi        RWO            local-path     4d20h
pgdata                         Bound    pg-pv                                      1Gi        RWO            local-path     3h31m
``` 

# 创建cronjob
获取pg的密码
```sh
xzfa@xiongzfa:~/workfiles/yaml/keycloak/pgdata$ k get secrets 
NAME                                         TYPE                                  DATA   AGE
keycloak-postgresql                          Opaque                                2      4d21h
```
```sh
xzfa@xiongzfa:~/workfiles/yaml/keycloak/pgdata$ k describe secrets keycloak-postgresql
Name:         keycloak-postgresql
Namespace:    default
Labels:       app.kubernetes.io/instance=keycloak
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=postgresql
              helm.sh/chart=postgresql-11.6.6
Annotations:  meta.helm.sh/release-name: keycloak
              meta.helm.sh/release-namespace: default

Type:  Opaque

Data
====
password:           10 bytes
postgres-password:  10 bytes
```
我们需要使用的pg密码就是`postgres-password`


由于pod中使用的时间为UST世界标准时间，我们需要使用的是CST东八区的上海时间，因此将宿主机的时间文件挂载到pod中

cronjob.yaml
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgresql-backup
spec:
  schedule: "* * * * *"       #这里是任务的执行时间，我这里为了快速看到效果设置成为了每分钟都执行。
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3    #成功的任务仅保留显示三个
  failedJobsHistoryLimit: 2        #失败的任务仅保留显示二个
  jobTemplate:
    spec:
      template:
        spec:
          securityContext:         #pod执行是时使用root用户
            runAsUser: 0
          containers:
            - name: postgresql-backup
              image: bitnami/postgresql
              imagePullPolicy: "IfNotPresent"
              command : ["/bin/sh", "-c"]
              args: ["pg_dumpall -U postgres  --host=keycloak-postgresql >> /tmp/pgdump/$(date +\"%Y_%m_%d_%H:%M\")"]      
              env:                 #设置执行所需要的变量
              - name: PGPASSWORD    #pod中的变量名称
                valueFrom:
                  secretKeyRef:     #使用keycloak-postgresql这个secret中的postgres-password这个值
                    name: keycloak-postgresql
                    key: postgres-password
              volumeMounts:         # 挂载到容器的目录
              - name: timezone       
                mountPath: "/etc/localtime"                          
              - name: pgdump
                mountPath: "/tmp/pgdump/"
          restartPolicy: Never
          volumes:                  
          - name: pgdump
            persistentVolumeClaim:
              claimName: pgdata
              volumes:
          - name: timezone
            hostPath:
              path: /usr/share/zoneinfo/Asia/Shanghai            # 宿主机的时间文件
```
```sh
k apply -f cronjob.yaml
```
创建成功crojob。可以看到crojob启动的pod以及生成的备份文件。
```sh
xzfa@xiongzfa:~/workfiles/yaml/keycloak/pgdata$ k get all
NAME                                   READY   STATUS      RESTARTS         AGE
pod/postgresql-backup-27626829-s4qcm   0/1     Completed   0                2m5s
pod/postgresql-backup-27626830-hnn4j   0/1     Completed   0                65s
pod/postgresql-backup-27626831-jrlzd   0/1     Completed   0                5s

service/keycloak-postgresql      NodePort       10.43.174.180   <none>        5432:30608/TCP               4d21h


NAME                              SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/postgresql-backup   * * * * *   False     0        5s              66m

NAME                                   COMPLETIONS   DURATION   AGE
job.batch/postgresql-backup-27626829   1/1           3s         2m5s
job.batch/postgresql-backup-27626830   1/1           3s         65s
job.batch/postgresql-backup-27626831   1/1           3s         5s

xzfa@xiongzfa:~/workfiles/yaml/keycloak/pgdata$ ls
2022_07_12_14:42  2022_07_12_14:46  2022_07_12_14:50  2022_07_12_14:54  2022_07_12_14:58  2022_07_12_15:02  2022_07_12_15:06  2022_07_12_15:10
2022_07_12_14:43  2022_07_12_14:47  2022_07_12_14:51  2022_07_12_14:55  2022_07_12_14:59  2022_07_12_15:03  2022_07_12_15:07  cronjob.yaml
2022_07_12_14:44  2022_07_12_14:48  2022_07_12_14:52  2022_07_12_14:56  2022_07_12_15:00  2022_07_12_15:04  2022_07_12_15:08  pvc.yaml
2022_07_12_14:45  2022_07_12_14:49  2022_07_12_14:53  2022_07_12_14:57  2022_07_12_15:01  2022_07_12_15:05  2022_07_12_15:09  pv.yaml

```
