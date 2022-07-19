
# Get Repo Info
```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
# Installing the Chart
```sh
helm install grafana -n monitor grafana/grafana
helm install prometheus -n monitor prometheus-community/prometheus
```


default.conf
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

    location /stub_status{
         stub_status  on;
         access_log    off;
}
}

nginx -t 





```
```yaml



    - mountPath: /etc/config                                   
      name: config-volume                                      
    - mountPath: /data                                         
      name: storage-volume                                     
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount 


```


