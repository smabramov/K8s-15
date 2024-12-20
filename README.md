# Домашнее задание к занятию Troubleshooting

### Цель задания

Устранить неисправности при деплое приложения.

### Чеклист готовности к домашнему заданию

1. Кластер K8s.

### Задание. При деплое приложение web-consumer не может подключиться к auth-db. Необходимо это исправить

1. Установить приложение по команде:
```shell
kubectl apply -f https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml
```
2. Выявить проблему и описать.
3. Исправить проблему, описать, что сделано.
4. Продемонстрировать, что проблема решена.

### Решение

Запускаю приложение.

```
serg@k8snode:~/git/K8s-15$ kubectl apply -f https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml
Error from server (NotFound): error when creating "https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml": namespaces "web" not found
Error from server (NotFound): error when creating "https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml": namespaces "data" not found
Error from server (NotFound): error when creating "https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml": namespaces "data" not found
```

Не запускается, т.к. не созданны пространства имен web и data, создаю:

```
serg@k8snode:~/git/K8s-15$ kubectl create namespace web
namespace/web created
serg@k8snode:~/git/K8s-15$ kubectl create namespace data
namespace/data created
serg@k8snode:~/git/K8s-15$ kubectl get namespace
NAME                     STATUS   AGE
data                     Active   28s
default                  Active   42d
ingress                  Active   34d
kube-node-lease          Active   42d
kube-public              Active   42d
kube-system              Active   42d
nfs-server-provisioner   Active   28d
web                      Active   38s
```
Зупускаю:

```
serg@k8snode:~/git/K8s-15$ kubectl apply -f https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml
deployment.apps/web-consumer created
deployment.apps/auth-db created
service/auth-db created
serg@k8snode:~/git/K8s-15$ kubectl get all -n data
NAME                           READY   STATUS    RESTARTS   AGE
pod/auth-db-79c4894db7-n4j8s   1/1     Running   0          18s

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/auth-db   ClusterIP   10.152.183.42   <none>        80/TCP    18s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/auth-db   1/1     1            1           18s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/auth-db-79c4894db7   1         1         1       18s
serg@k8snode:~/git/K8s-15$ kubectl get all -n web
NAME                               READY   STATUS    RESTARTS   AGE
pod/web-consumer-6ccf95f84-npbrz   1/1     Running   0          43s
pod/web-consumer-6ccf95f84-xd42k   1/1     Running   0          43s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web-consumer   2/2     2            2           43s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/web-consumer-6ccf95f84   2         2         2       43s
```
Смотрим логи:

```
serg@k8snode:~/git/K8s-15$ kubectl logs deployments/auth-db -n data
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
serg@k8snode:~/git/K8s-15$ kubectl -n web logs web-consumer-6ccf95f84-npbrz 
curl: (6) Couldn't resolve host 'auth-db'
curl: (6) Couldn't resolve host 'auth-db'
curl: (6) Couldn't resolve host 'auth-db'
curl: (6) Couldn't resolve host 'auth-db'
curl: (6) Couldn't resolve host 'auth-db'
curl: (6) Couldn't resolve host 'auth-db'
curl: (6) Couldn't resolve host 'auth-db'
curl: (6) Couldn't resolve host 'auth-db
serg@k8snode:~/git/K8s-15$ kubectl describe deployment web-consumer -n web
Name:                   web-consumer
Namespace:              web
CreationTimestamp:      Fri, 20 Dec 2024 11:09:24 +0300
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=web-consumer
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=web-consumer
  Containers:
   busybox:
    Image:      radial/busyboxplus:curl
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      while true; do curl auth-db; sleep 5; done
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   web-consumer-6ccf95f84 (2/2 replicas created)
Events:          <none>
```
Видно что с приложением auth-db все хорошо, а вот с web есть проблемы, не может достучаться до имени хоста.

Необходимо поменять в манифесте do curl auth-db на do curl auth-db.data

[task.yaml](https://github.com/smabramov/K8s-15/blob/7157ee422c92ec128824e44286c70dabbe8554c0/files/task.yaml)

Применяю изменения:

```
serg@k8snode:~/git/K8s-15$ kubectl apply -f files/task.yaml 
deployment.apps/web-consumer configured
deployment.apps/auth-db unchanged
service/auth-db unchanged
serg@k8snode:~/git/K8s-15$ kubectl get deployments.apps -A
NAMESPACE     NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
data          auth-db                     1/1     1            1           27m
kube-system   calico-kube-controllers     1/1     1            1           42d
kube-system   coredns                     1/1     1            1           42d
kube-system   csi-nfs-controller          1/1     1            1           28d
kube-system   dashboard-metrics-scraper   1/1     1            1           42d
kube-system   hostpath-provisioner        1/1     1            1           28d
kube-system   kubernetes-dashboard        1/1     1            1           42d
kube-system   metrics-server              1/1     1            1           42d
web           web-consumer                2/2     2            2           27m
```
Проверяю логи:

```
serg@k8snode:~/git/K8s-15$ kubectl -n web logs pod/web-consumer-5549454c78-m8cjw 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
100   612  100   612    0     0   314k      0 --:--:-- --:--:-- --:--:--  597k

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0   367k      0 --:--:-- --:--:-- --:--:--  597k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
```
Все работает.


### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
