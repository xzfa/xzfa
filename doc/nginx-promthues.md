
### Get Repo Info
```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
### Installing the Chart
```sh
helm install grafana -n monitor grafana/grafana
helm install prometheus -n monitor prometheus-community/prometheus
```
### 一些修改
We need to visit the UI page to see the effect, so we expose the ports of prometheus-altermanager, prometheus, and granfana. And made ingress for grafana (although it was not used much).
我们应该修改svc的type为NodePort， 接下来是grafana的ingress文件
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitor
spec:
  tls:
    - hosts:
      - grafana.digiits.xzf
  rules:
  - host: grafana.digiits.xzf
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 80

```
```sh
k apply -f ingress.yaml
```
get user/password from secret
```sh
xzfa@xiongzfa:~/workfiles/yaml/monitor$ k get secrets -n monitor 
NAME                                        TYPE                                  DATA   AGE
grafana                                     Opaque                                3      10d

xzfa@xiongzfa:~/workfiles/yaml/monitor$ k edit  secrets -n monitor grafana
```
base64 user/password 

###  配置nginx接入prometheus
 nginx需要把自己的信息给暴露出来，需要再default.conf文件中进行配置，因为测试需要经常修改，因此我只是做了一个pvc把配置文件挂载出来。之后会用configmap的方式挂载配置文件。

因为关于vue的配置在之前的部署在部署文档中写过，这里只是做了一个修改，因此ingress、svc、vue的挂载就不再多说。下面只是对deployment的一些修改、以及nginx配置文件的挂载
 ```yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx
spec:
  capacity:
    storage: 100m
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany
  hostPath:
    path: "/mnt/nginx/"
    type: DirectoryOrCreate
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-path

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 100m
  storageClassName: "local-path"

apiVersion: apps/v1
kind: Deployment
metadata:
  name: vue-nginx
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: vue-nginx
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: nginx
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d/
      volumes:
      - name: html
        persistentVolumeClaim:
          claimName: vue-nginx-claim
      - name: nginx-conf
        persistentVolumeClaim:
          claimName: nginx

 ```
下面是挂载的nginx的配置文件`default.conf`
```sh
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;


    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    location /nginx_status{
         stub_status  on;
         access_log    off;
}
}

```

`tips`：下面是我测试时使用的方法、在default.conf修改好后再进行挂载是不需要这`nginx -t` `nginx -s reload`

开启ngixn_status后进行测试，得到如下结果即为成功
```sh
root@vue-nginx-8ddff5cdc-ffn7c:/# curl 127.0.0.1/nginx_status
Active connections: 2 
server accepts handled requests
 2 2 32 
Reading: 0 Writing: 1 Waiting: 1 
```
### 接下来安装nginx-prometheus-exporter
`nginx-prometheus-exporter`的作用就是把nginx暴露出来的信息采集成为`metrics`从而弄够被`prometheus-server`采集并识别。

以下是nginx-prometheus-exporter的部署
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-exporter
  name: nginx-exporter
  namespace: monitor
spec:
  replicas: 1
  revisionHistoryLimit: 5 
  selector:
    matchLabels:
      app: nginx-exporter
  template:
    metadata:
      labels:
        app: nginx-exporter
    spec:
      containers:
      - image: nginx/nginx-prometheus-exporter
        imagePullPolicy: IfNotPresent
        name: nginx-prometheus-exporter
        command : ["nginx-prometheus-exporter"]             #进入容器后执行命令nginx-prometheus-exporter
        args: ["-nginx.scrape-uri=http://vue-nginx.default/nginx_status" ]    #参数-nginx.scrape-uri为nginx-prometheus-exporter需要采集的nginx的地址。
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-exporter
  name: nginx-exporter
  namespace: monitor
spec:
  selector:
    app: nginx-exporter
  type: ClusterIP
  ports:
    - name: nginx-exporter
      port: 9113
      targetPort: 9113
```
配置完成后在进入pod`vue-nginx`测试nginx-prometheus-exporter是否部署成功,这里也可以使用Nodeport把端口暴露出来进行测试，这个只是我的一个测试方法。
```sh
root@vue-nginx-8ddff5cdc-ffn7c:/# curl nginx-exporter.monitor:9113/metrics
# HELP nginx_connections_accepted Accepted client connections
# TYPE nginx_connections_accepted counter
nginx_connections_accepted 2
# HELP nginx_connections_active Active client connections
# TYPE nginx_connections_active gauge
nginx_connections_active 1
# HELP nginx_connections_handled Handled client connections
# TYPE nginx_connections_handled counter
nginx_connections_handled 2
# HELP nginx_connections_reading Connections where NGINX is reading the request header
# TYPE nginx_connections_reading gauge
nginx_connections_reading 0
# HELP nginx_connections_waiting Idle client connections
# TYPE nginx_connections_waiting gauge
nginx_connections_waiting 0
# HELP nginx_connections_writing Connections where NGINX is writing the response back to the client
# TYPE nginx_connections_writing gauge
nginx_connections_writing 1
# HELP nginx_http_requests_total Total http requests
# TYPE nginx_http_requests_total counter
nginx_http_requests_total 905
# HELP nginx_up Status of the last metric scrape
# TYPE nginx_up gauge
nginx_up 1
# HELP nginxexporter_build_info Exporter build information
# TYPE nginxexporter_build_info gauge
nginxexporter_build_info{commit="7a03d0314425793cf4001f0d9b0b2cfd19563433",date="2021-12-21T19:24:34Z",version="0.10.0"} 1
```
### Metrics for NGINX OSS:
Name | Type | Description | Labels
----|----|----|----|
`nginx_connections_accepted` | Counter | Accepted client connections. | [] |
`nginx_connections_active` | Gauge | Active client connections. | [] |
`nginx_connections_handled` | Counter | Handled client connections. | [] |
`nginx_connections_reading` | Gauge | Connections where NGINX is reading the request header. | [] |
`nginx_connections_waiting` | Gauge | Idle client connections. | [] |
`nginx_connections_writing` | Gauge | Connections where NGINX is writing the response back to the client. | [] |
`nginx_http_requests_total` | Counter | Total http requests. | [] |
`nginxexporter_build_info` | Gauge | Shows the exporter build information. | `gitCommit`, `version` |
`nginx_up` | Gauge | Shows the status of the last metric scrape: `1` for a successful scrape and `0` for a failed one | [] |

### 配置prometheus-server获取nginx
helm默认把prometheus-server的配置文件`prometheus.yaml`以cofigmap的方式挂载了出来，并且在prometheus-server的pod中有两个容器，分别为`prometheus-server`与`prometheus-server-configmap-reload`，在我们修改configmap后，pod会识别到改变并且自动的更新pod中prometheus-server的配置文件。

下面我们修改prometheus.yaml
```sh
xzfa@xiongzfa:~/workfiles/github$ k get -n monitor cm
NAME                      DATA   AGE
prometheus-server         5      9d
```
prometheus-server的configmap中有很多配置文件，这里我们只需要在prometheus.yml文件中添加一个job即可。

`tips`：这里的targets中的地址端口为91加13,需要在域名后加上。如果是80端口的话怎么不需要加。
```yaml
---
    - job_name: 'nginx-exporter'
      static_configs:
        - targets: ["nginx-exporter:9113"]
---
```
配置完成后可以在prometheus-server的UI页面中看到已经添加上去的nginx的PromQL。
![](../../doc-img/prometheus-nginx.png)
### 配置grafana进行nginx监控的可视化
访问grafana的UI页面,添加一个prometheus的`data source`
url填写k3s中的域名`prometheus-server`

随后以json文件的方式导入nginx的dashboard。
- [ngixn-dashboard](./nginx.json)

### 设置prometheus告警规则
设置两个告警规则分别是：
- 当nginx服务down时进行告警
- 当niginx每五分钟的平均访问总量超过2时进行告警（这里我也不是很清楚，先这么解释吧）

在prometheus的configmap中修改配置文件`alerting_rules.yml`
```yaml
  alerting_rules.yml: |
    groups:
    - name: nginx
      rules:
      - alert: nginx-down
        expr: nginx_up{job="nginx-exporter"}==0
        for: 1s
        labels:
          severity: lalala
        annotations:
          summary: nginx down
          description: nginx down 
      - alert: nginx_http_requests_total
        expr: irate(nginx_http_requests_total{job="nginx-exporter"}[5m])>2
        for: 1s
        labels:
          severity: nginx
        annotations:
          summary: nginx_http_requests_total>2
          description: nginx_http_requests_total>2
```
并且修改部分`prometheus.yml`文件配置
```yaml
  prometheus.yml: |
    global:
      evaluation_interval: 1s  #指的是计算rule的间隔
      scrape_interval: 1s      #拉取 targets 的默认时间间隔
      scrape_timeout: 1s       #拉取一个 target 的超时时间
    rule_files:
    - /etc/config/recording_rules.yml
    - /etc/config/alerting_rules.yml
    - /etc/config/rules
    - /etc/config/alerts
            ---
```
配置完成、接下来就开始测试。
### 测试nginx服务down掉时的告警
我们手动把nginx服务给删除
```sh
xzfa@xiongzfa:~/workfiles/yaml/vue$ k delete deployments.apps vue-nginx 
deployment.apps "vue-nginx" deleted
```
然后观察prometheus-server的UI页面可以看到告警成功
![](../../doc-img/alert-test.png)

### 配置prometheus-server、alertmanager进行告警转发

需要把prometheus-server的告警发送给alertmanager，在这里碰见一个大坑

prometheus-server产生了报警之后alertmanager没有收集到，沿着这个问题、我尝试了三个方向
- 配置prometheus-server的转发地址
- alertmanager的`cluester-status`为`disable`状态
- alertmanager的日志中有`tls is disable`字样
####  配置prometheus-server的转发地址
在prometheus-server的configmap中的prometheus.yaml中添加如下配置
```yaml
      alertmanagers:
      - static_configs:
        - targets: ["prometheus-alertmanager"]
```
修改后alertmanager即可收到告警
#### alertmanager的`cluester-status`为`disable`状态
在alertmanager的depolyment.yaml中配置cluster.listen-address,helm默认安装时该参数为空
```yaml
    spec:
      containers:
      - args:
        - --config.file=/etc/config/alertmanager.yml
        - --storage.path=/data
        - --cluster.listen-address=0.0.0.0:6783
        - --web.external-url=http://localhost:9093
```
修改后cluester-status即可变为ready，但是并没有解决问题。就算不修改也可以收到告警。我不知道它有什么用。
#### alertmanager的日志中有`tls is disable`字样
tls功能默认没有开启，可以在altermanager配置文件中开启，目前没有尝试。

测试alertmanager接收prometheus-server的告警,我们手动把nginx服务给删除
```sh
xzfa@xiongzfa:~/workfiles/yaml/vue$ k delete deployments.apps vue-nginx 
deployment.apps "vue-nginx" deleted
```
在prometheus-server的UI中可以看到告警，在观察alertmanager的UI页面，发现了相同的告警
![](../../doc-img/alertmanager.png)


今晚就写到这里，我要睡觉了！