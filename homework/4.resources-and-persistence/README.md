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

$ kubectl apply -f ./homework/4.resources-and-persistence
deployment.apps/postgres created
persistentvolume/pv1 created
persistentvolumeclaim/pvc1 created
secret/secure created

$ kubectl get po
NAME                        READY   STATUS              RESTARTS   AGE
postgres-7f8677b965-2r9k5   0/1     ContainerCreating   0          17s

$ kubectl describe po postgres-7f8677b965-2r9k5
Name:             postgres-7f8677b965-2r9k5
Namespace:        pg
Priority:         0
Service Account:  default
Node:             kubernetes-cluster-8663-default-group-0/10.0.0.22
Start Time:       Sun, 05 Feb 2023 23:04:40 +0500
Labels:           app=postgres
                  pod-template-hash=7f8677b965
Annotations:      cni.projectcalico.org/containerID: 2e50f99be342a0cb33d9bca69f7887bb10d0a490b550087ef5b12065bc6f5490
                  cni.projectcalico.org/podIP: 10.100.230.193/32
                  cni.projectcalico.org/podIPs: 10.100.230.193/32
Status:           Running
IP:               10.100.230.193
IPs:
  IP:           10.100.230.193
Controlled By:  ReplicaSet/postgres-7f8677b965
Containers:
  postgres:
    Container ID:   cri-o://9125bd4e707082aa0230c1d06519f4d896c74d888c18d3b32af8a764b4c3ee68
    Image:          postgres:10.13
    Image ID:       docker.io/library/postgres@sha256:0bed71d0c0837b4afcd4e04ea5f5a6c680d96049a0ab3770d0fbc22504282abc
    Port:           5432/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 05 Feb 2023 23:04:57 +0500
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
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-cxtqb (ro)
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
  kube-api-access-cxtqb:
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
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  3m41s  default-scheduler  0/2 nodes are available: 2 persistentvolumeclaim "pvc1" not found.
  Normal   Scheduled         3m39s  default-scheduler  Successfully assigned pg/postgres-7f8677b965-2r9k5 to kubernetes-cluster-8663-default-group-0
  Normal   Pulling           3m38s  kubelet            Pulling image "postgres:10.13"
  Normal   Pulled            3m22s  kubelet            Successfully pulled image "postgres:10.13" in 15.711765617s
  Normal   Created           3m22s  kubelet            Created container postgres
  Normal   Started           3m22s  kubelet            Started container postgres

$ kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP               NODE                                      NOMINATED NODE   READINESS GATES
postgres-7f8677b965-2r9k5   1/1     Running   0          4m52s   10.100.230.193   kubernetes-cluster-8663-default-group-0   <none>           <none>

$ kubectl run -t -i --rm --image postgres:10.13 test bash
If you don't see a command prompt, try pressing enter.
root@test:/# psql -h 10.100.230.193 -U testuser testdatabase
Password for user testuser:                                        - testpassword
psql (10.13 (Debian 10.13-1.pgdg90+1))
Type "help" for help.

testdatabase=# CREATE TABLE testtable (testcolumn VARCHAR (50) );
CREATE TABLE
testdatabase=# \dt
           List of relations
 Schema |   Name    | Type  |  Owner   
--------+-----------+-------+----------
 public | testtable | table | testuser
(1 row)

testdatabase=# \q
root@test:/# exit
exit
Session ended, resume using 'kubectl attach test -c test -i -t' command when the pod is running
pod "test" deleted

$ kubectl delete pod postgres-7f8677b965-2r9k5
pod "postgres-7f8677b965-2r9k5" deleted

$ kubectl get po
NAME                        READY   STATUS    RESTARTS   AGE
postgres-7f8677b965-wltm2   1/1     Running   0          32s

$ kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE    IP               NODE                                      NOMINATED NODE   READINESS GATES
postgres-7f8677b965-wltm2   1/1     Running   0          118s   10.100.230.195   kubernetes-cluster-8663-default-group-0   <none>           <none>

$ kubectl run -t -i --rm --image postgres:10.13 test bash
If you don't see a command prompt, try pressing enter.
root@test:/# psql -h 10.100.230.195 -U testuser testdatabase
Password for user testuser: 
psql (10.13 (Debian 10.13-1.pgdg90+1))
Type "help" for help.

testdatabase=# \dt
           List of relations
 Schema |   Name    | Type  |  Owner   
--------+-----------+-------+----------
 public | testtable | table | testuser
(1 row)

testdatabase=# \q
root@test:/# exit
exit
Session ended, resume using 'kubectl attach test -c test -i -t' command when the pod is running
pod "test" deleted






