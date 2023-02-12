# Домашнее задание для к уроку 5 - Сетевые абстракции Kubernetes

* Разверните в кластере сервер базы данных Postgresql. Из предыдущего задания.

* Добавьте к нему service c портом 5432 и именем database.

* В этом же неймспэйсе создайте deployment с образом redmine:4.1.1

Для запуска нужно передать переменные окружения:

REDMINE_DB_POSTGRES = database
REDMINE_DB_USERNAME = <postgres_user>
REDMINE_DB_PASSWORD = <postgres_password> (значение должно браться из секрета)
REDMINE_DB_DATABASE = <postgres_database>
REDMINE_SECRET_KEY_BASE = supersecretkey (значение должно браться из секрета)

> Обратите внимание что имя пользователя, пароль и база данных должны соответствовать
> значениям которые указаны в переменных окружения деплоймента postgresql

В деплойменте приложения должен быть описан порт 3000

* Создайте serivce для приложения с портом 3000

* Создайте ingress для приложения, так чтобы запросы с любым доменом на белый IP
вашего сервиса nginx-ingress-controller (тот что в нэймспэйсе ingress-nginx с типом LoadBalancer)
шли на приложение

* Проверьте что при обращении из браузера на белый IP вы видите открывшееся
приложение Redmine (https://www.redmine.org/)





$export KUBECONFIG="/media/razif/Новый том/GeekBrains/Новая папка (5)/Новая папка (5)/kubernetes-cluster-8594_kubeconfig.yaml"

$ kubectl cluster-info
Kubernetes control plane is running at https://185.241.195.8:6443
CoreDNS is running at https://185.241.195.8:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get node
NAME                                      STATUS   ROLES    AGE     VERSION
kubernetes-cluster-8594-default-group-0   Ready    <none>   7m28s   v1.22.9
kubernetes-cluster-8594-default-group-1   Ready    <none>   7m29s   v1.22.9
kubernetes-cluster-8594-master-0          Ready    master   13m     v1.22.9

$ kubectl create ns pg
namespace/pg created

$ kubectl config set-context --current --namespace=pg
Context "default/kubernetes-cluster-8594" modified.

$ kubectl apply -f ./homework/5.kubernetes-network/postgresql
deployment.apps/postgres created
persistentvolume/pv1 created
persistentvolumeclaim/pvc1 created
secret/secure created
service/database created

$ kubectl get po
NAME                        READY   STATUS              RESTARTS   AGE
postgres-7f8677b965-885js   0/1     ContainerCreating   0          15s

$ kubectl describe po postgres-7f8677b965-885js
Name:             postgres-7f8677b965-885js
Namespace:        pg
Priority:         0
Service Account:  default
Node:             kubernetes-cluster-8594-default-group-1/10.0.0.4
Start Time:       Sun, 12 Feb 2023 22:06:46 +0500
Labels:           app=postgres
                  pod-template-hash=7f8677b965
Annotations:      cni.projectcalico.org/containerID: 392e84869287fe86aae3cff91d1b690d02ab5514f410023f98365bd7e7712904
                  cni.projectcalico.org/podIP: 10.100.136.0/32
                  cni.projectcalico.org/podIPs: 10.100.136.0/32
Status:           Running
IP:               10.100.136.0
IPs:
  IP:           10.100.136.0
Controlled By:  ReplicaSet/postgres-7f8677b965
Containers:
  postgres:
    Container ID:   cri-o://82919645ac92942687d27803bd5b7b214fb6137480e292559845fa3da365a757
    Image:          postgres:10.13
    Image ID:       docker.io/library/postgres@sha256:0bed71d0c0837b4afcd4e04ea5f5a6c680d96049a0ab3770d0fbc22504282abc
    Port:           5432/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 12 Feb 2023 22:07:02 +0500
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     1
      memory:  512Mi
    Requests:
      cpu:     200m
      memory:  256Mi
    Environment:
      PGDATA:             /var/lib/postgresql/data/pgdata
      POSTGRES_DB:        testdatabase
      POSTGRES_USER:      testuser
      POSTGRES_PASSWORD:  <set to the key 'POSTGRES_PASSWORD' in secret 'secure'>  Optional: false
    Mounts:
      /var/lib/postgresql/data from pv1 (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fm2f2 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  pv1:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc1
    ReadOnly:   false
  kube-api-access-fm2f2:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  100s  default-scheduler  0/3 nodes are available: 3 persistentvolumeclaim "pvc1" not found.
  Normal   Scheduled         98s   default-scheduler  Successfully assigned pg/postgres-7f8677b965-885js to kubernetes-cluster-8594-default-group-1
  Normal   Pulling           98s   kubelet            Pulling image "postgres:10.13"
  Normal   Pulled            82s   kubelet            Successfully pulled image "postgres:10.13" in 15.480811082s
  Normal   Created           82s   kubelet            Created container postgres
  Normal   Started           82s   kubelet            Started container postgres

$ kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE    IP             NODE                                      NOMINATED NODE   READINESS GATES
postgres-7f8677b965-885js   1/1     Running   0          3m9s   10.100.136.0   kubernetes-cluster-8594-default-group-1   <none>           <none>

$ kubectl get svc database -o wide
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE    SELECTOR
database   ClusterIP   10.254.79.144   <none>        5432/TCP   7m2s   app=postgres

$ kubectl apply -f ./homework/5.kubernetes-network/redmine
deployment.apps/redmine created
ingress.networking.k8s.io/ingress-myserviceb created
secret/redmine-secret created
service/redmine-service created

$ kubectl get po
NAME                        READY   STATUS    RESTARTS   AGE
postgres-7f8677b965-885js   1/1     Running   0          10m
redmine-5fbdc6cc5d-q7q8t    1/1     Running   0          31s

$ kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE    IP               NODE                                      NOMINATED NODE   READINESS GATES
postgres-7f8677b965-885js   1/1     Running   0          11m    10.100.136.0     kubernetes-cluster-8594-default-group-1   <none>           <none>
redmine-5fbdc6cc5d-q7q8t    1/1     Running   0          2m1s   10.100.199.128   kubernetes-cluster-8594-default-group-0   <none>           <none>

!!!   https://kubernetes.github.io/ingress-nginx/deploy/

$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/aws/deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created

$ kubectl get ingress
NAME                 CLASS   HOSTS               ADDRESS           PORTS   AGE
ingress-myserviceb   nginx   myservice.foo.org   146.185.240.140   80      30m

$ kubectl get svc -A
NAMESPACE        NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                      AGE
default          kubernetes                           ClusterIP      10.254.0.1       <none>            443/TCP                      42m
ingress-nginx    ingress-nginx-controller             LoadBalancer   10.254.205.46    146.185.240.140   80:31924/TCP,443:30042/TCP   12m
ingress-nginx    ingress-nginx-controller-admission   ClusterIP      10.254.161.151   <none>            443/TCP                      12m
kube-system      calico-node                          ClusterIP      None             <none>            9091/TCP                     42m
kube-system      calico-typha                         ClusterIP      10.254.198.120   <none>            5473/TCP                     42m
kube-system      csi-cinder-controller-service        ClusterIP      10.254.102.51    <none>            12345/TCP                    41m
kube-system      dashboard-metrics-scraper            ClusterIP      10.254.66.35     <none>            8000/TCP                     41m
kube-system      kube-dns                             ClusterIP      10.254.0.10      <none>            53/UDP,53/TCP,9153/TCP       41m
kube-system      kubernetes-dashboard                 ClusterIP      10.254.31.163    <none>            443/TCP                      41m
kube-system      metrics-server                       ClusterIP      10.254.246.254   <none>            443/TCP                      41m
opa-gatekeeper   gatekeeper-webhook-service           ClusterIP      10.254.188.198   <none>            443/TCP                      40m
pg               database                             ClusterIP      10.254.79.144    <none>            5432/TCP                     26m
pg               redmine-service                      ClusterIP      10.254.84.105    <none>            80/TCP                       16m

!!! Смотрим EXTERNAL-IP у NAME=nginx-ingress-controller с TYPE=LoadBalancer

146.185.240.140

$ kubectl get svc -n ingress-nginxNAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.254.205.46    146.185.240.140   80:31924/TCP,443:30042/TCP   27m
ingress-nginx-controller-admission   ClusterIP      10.254.161.151   <none>            443/TCP                      27m





