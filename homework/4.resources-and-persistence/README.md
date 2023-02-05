# Домашнее задание для к уроку 4 - Хранение данных и ресурсы

Напишите deployment для запуска сервера базы данных Postgresql.

Приложение должно запускаться из образа postgres:10.13

Должен быть описан порт:

- 5432 TCP

В деплойменте должна быть одна реплика, при этом при обновлении образа
НЕ ДОЛЖНО одновременно работать несколько реплик.
(то есть сначала должна удаляться старая реплика и только после этого подниматься новая).

> Это можно сделать или с помощью maxSurge/maxUnavailable или указав стратегию деплоя Recreate.

В базе данных при запуске должен автоматически создаваться пользователь testuser
с паролем testpassword. А также база testdatabase.

> Для этого нужно указать переменные окружения POSTGRES_PASSWORD, POSTGRES_USER, POSTGRES_DB в деплойменте.
> При этом значение переменной POSTGRES_PASSWORD должно браться из секрета.

Так же нужно указать переменную PGDATA со значением /var/lib/postgresql/data/pgdata
См. документацию к образу https://hub.docker.com/_/postgres раздел PGDATA

База данных должна хранить данные в PVC c размером диска в 10Gi, замонтированном в pod по пути /var/lib/postgresql/data


## Проверка

Для проверки работоспособности базы данных:

1. Узнайте IP пода postgresql

kubectl get pod -o wide

2. Запустите рядом тестовый под

kubectl run -t -i --rm --image postgres:10.13 test bash

3. Внутри тестового пода выполните команду для подключения к БД

psql -h <postgresql pod IP из п.1> -U testuser testdatabase

Введите пароль - testpassword

4. Все в том же тестовом поде, после подключения к инстансу БД выполните команду для создания таблицы

CREATE TABLE testtable (testcolumn VARCHAR (50) );

5. Проверьте что таблица создалась. Для этого все в том же тестовом поде выполните команду 

\dt

6. Выйдите из тестового пода. Попробуйте удалить под с postgresql.

7. После его пересоздания повторите все с п.1, кроме п.4
Проверьте что созданная ранее таблица никуда не делась.







$export KUBECONFIG="/media/razif/Новый том/GeekBrains/Новая папка (5)/Новая папка (4)/kubernetes-cluster-8663_kubeconfig.yaml"

$ kubectl cluster-info
Kubernetes control plane is running at https://185.241.195.8:6443
CoreDNS is running at https://185.241.195.8:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get node
NAME                                      STATUS   ROLES    AGE   VERSION
kubernetes-cluster-8663-default-group-0   Ready    <none>   18m   v1.22.9
kubernetes-cluster-8663-master-0          Ready    master   18m   v1.22.9

$ kubectl create ns pg
namespace/pg created

$ kubectl config set-context --current --namespace=pg
Context "default/kubernetes-cluster-8663" modified.

$ kubectl get storageclasses.storage.k8s.io
NAME                       PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
csi-ceph-hdd-gz1           cinder.csi.openstack.org   Delete          Immediate           true                   21m
csi-ceph-hdd-gz1-retain    cinder.csi.openstack.org   Retain          Immediate           true                   21m
csi-ceph-hdd-ms1           cinder.csi.openstack.org   Delete          Immediate           true                   21m
csi-ceph-hdd-ms1-retain    cinder.csi.openstack.org   Retain          Immediate           true                   21m
csi-ceph-ssd-gz1           cinder.csi.openstack.org   Delete          Immediate           true                   21m
csi-ceph-ssd-gz1-retain    cinder.csi.openstack.org   Retain          Immediate           true                   21m
csi-ceph-ssd-ms1           cinder.csi.openstack.org   Delete          Immediate           true                   21m
csi-ceph-ssd-ms1-retain    cinder.csi.openstack.org   Retain          Immediate           true                   21m
csi-high-iops-gz1          cinder.csi.openstack.org   Delete          Immediate           true                   21m
csi-high-iops-gz1-retain   cinder.csi.openstack.org   Retain          Immediate           true                   21m
csi-high-iops-ms1          cinder.csi.openstack.org   Delete          Immediate           true                   21m
csi-high-iops-ms1-retain   cinder.csi.openstack.org   Retain          Immediate           true                   21m

Выбираю csi-ceph-ssd-ms1

$ kubectl apply -f ./homework/4.resources-and-persistence/pvc.yaml
persistentvolumeclaim/pg-storage created

$ kubectl get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
pg-storage   Bound    pvc-26e57360-f992-41c4-be4c-209fa86ed440   10Gi       RWX            csi-ceph-ssd-ms1   43s

$ kubectl create secret generic pg-secret --from-literal=PASS=testpassword
secret/pg-secret created

$ kubectl get secret pg-secret
NAME        TYPE     DATA   AGE
pg-secret   Opaque   1      39s

$ kubectl get secret pg-secret -oyaml
apiVersion: v1
data:
  PASS: dGVzdHBhc3N3b3Jk
kind: Secret
metadata:
  creationTimestamp: "2023-02-05T15:26:33Z"
  name: pg-secret
  namespace: pg
  resourceVersion: "10834"
  selfLink: /api/v1/namespaces/pg/secrets/pg-secret
  uid: 0088ba84-4081-414e-9ce6-38e43bf25608
type: Opaque

$ kubectl apply -f homework/4.resources-and-persistence/deployment.yaml
deployment.apps/pg-db created

$ kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
pg-db-85db8cc466-s9j7w   0/1     Pending   0          37s

$ kubectl describe po pg-db-85db8cc466-s9j7w
Name:             pg-db-85db8cc466-s9j7w
Namespace:        pg
Priority:         0
Service Account:  default
Node:             <none>
Labels:           app=pg-db
                  pod-template-hash=85db8cc466
Annotations:      kubernetes.io/limit-ranger:
                    LimitRanger plugin set: cpu, memory request for init container mount-permissions-fix; cpu, memory limit for init container mount-permissio...
Status:           Pending
IP:               
IPs:              <none>
Controlled By:    ReplicaSet/pg-db-85db8cc466
Init Containers:
  mount-permissions-fix:
    Image:      busybox
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      chmod 777 /var/lib/postgresql/data
    Limits:
      cpu:     500m
      memory:  512Mi
    Requests:
      cpu:        100m
      memory:     64Mi
    Environment:  <none>
    Mounts:
      /var/lib/postgresql/data from data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-48q5s (ro)
Containers:
  postgres:
    Image:      postgres:10.13
    Port:       5432/TCP
    Host Port:  0/TCP
    Limits:
      cpu:     1
      memory:  512Mi
    Requests:
      cpu:     200m
      memory:  256Mi
    Environment:
      POSTGRES_USER:      testuser
      POSTGRES_DB:        testdatabase
      PGDATA:             /var/lib/postgresql/data/pgdata
      POSTGRES_PASSWORD:  <set to the key 'PASS' in secret 'pg-secret'>  Optional: false
    Mounts:
      /var/lib/postgresql/data from data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-48q5s (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pg-storage
    ReadOnly:   false
  kube-api-access-48q5s:
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
  Type     Reason             Age                 From                Message
  ----     ------             ----                ----                -------
  Normal   NotTriggerScaleUp  2m5s                cluster-autoscaler  pod didn't trigger scale-up:
  Warning  FailedScheduling   8s (x3 over 2m16s)  default-scheduler   0/2 nodes are available: 1 node(s) had taint {CriticalAddonsOnly: True}, that the pod didn't tolerate, 1 node(s) had volume node affinity conflict.




