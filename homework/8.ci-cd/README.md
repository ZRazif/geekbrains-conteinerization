# Домашнее задание для к уроку 8 - CI/CD

> В рамках данного задания нужно продолжить работу с CI приложения из лекции.
> То есть для начала выполнения этого домашнего задания необходимо проделать то,
> что показывалось в лекции.
> Все задание должно выполняться применительно к файлам в директории practice/8.ci-cd/app

Переделайте шаг деплоя в CI/CD, который демонстрировался на лекции
таким образом, чтобы при каждом прогоне шага deploy в кластер применялись
манифесты приложения. При этом версия докер образа в деплойменте при апплае
должна подменяться на ту, что была собрана в шаге build.

Для этого самым очевидным способом было бы воспользоваться утилитой sed.

* Измените образ в деплойменте приложения (файл kube/deployment.yaml) на плейсхолдер.

Вот это

```yaml
image: nginx:1.12 # это просто плэйсхолдер
```

На это

```yaml
image: __IMAGE__
```

* Измените шаг деплоя в .gitlab-ci.yml,
чтобы изменять __IMAGE__ на реальное имя образа и тег

Это

```yaml
- kubectl set image deployment/$CI_PROJECT_NAME *=$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG.$CI_PIPELINE_ID --namespace $CI_ENVIRONMENT_NAME
```

На это

```yaml
- sed -i "s,__IMAGE__,$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG.$CI_PIPELINE_ID,g" kube/deployment.yaml
- kubectl apply -f kube/ --namespace $CI_ENVIRONMENT_NAME
```

> Вторую строчку шага деплоя (которая отслеживает статус деплоя) оставьте без изменений.

* Попробуйте закоммитить свои изменения, запушить их в репозиторий
(тот же, который вы создавали во время лекции на Gitlab.com)
и посмотреть на выполнение CI в интерфейсе Gitlab.

> Так как окружений у нас два (stage и prod), то помимо образа при апплае из CI
> нам также было бы хорошо подменять host в ingress.yaml.
> Попробуйте реализовать это по аналогии, подставляя в ингресс вместо
> плэйсхолдера значение переменной $CI_ENVIRONMENT_NAME

* Так же попробуйте протестировать откат на предыдущую версию,
при возникновении ошибки при деплое

Для этого можно изменить значение переменной DB_HOST в deployment.yaml на какое нибудь несуществующее.
Тогда при старте приложения оно не сможет найти БД и будет постоянно рестрартовать. CI должен в течении progressDeadlineSeconds: 300 и по.сле этого запустить процедуру отката.
При этом не должно возникать недоступности приложения, так как старая реплика должна продолжать работать, пока новая пытается стартануть.



---------------------------------------------------------------------------------

Регистрируемся на https://gitlab.com

Добавляем свой публичный ключ для подключения по ssh

cd ./practice/8.ci-cd/app

git init

git remote add origin git@gitlab.com:ZRazif/geekbrains.git

git add .

git commit -m "Initial commit"

git push --set-upstream origin master

Settings - CI/CD - Runners - отключаем Shared Runners



export KUBECONFIG="/media/razif/Новый том/GeekBrains/Новая папка (5)/Новая папка (8)/kubernetes-cluster-8594_kubeconfig.yaml"

kubectl cluster-info



kubectl create ns gitlab

Открываем gitlab-runner.yaml - ищем <CHANGE ME> и вставляем вместо него токен
Settings - CI/CD - Runners - Set up a project runner for a project - And this registration token

kubectl apply --namespace gitlab -f ./homework/8.ci-cd/gitlab-runner/gitlab-runner.yaml

Runner появился в списке Project runners



kubectl create ns stage

kubectl create ns prod

kubectl create sa deploy --namespace stage

kubectl create rolebinding deploy --serviceaccount stage:deploy --clusterrole edit --namespace stage

kubectl create sa deploy --namespace prod

kubectl create rolebinding deploy --serviceaccount prod:deploy --clusterrole edit --namespace prod

Получаем токены:

export NAMESPACE=stage; kubectl get secret $(kubectl get sa deploy --namespace $NAMESPACE -o jsonpath='{.secrets[0].name}') --namespace $NAMESPACE -o jsonpath='{.data.token}'

export NAMESPACE=prod; kubectl get secret $(kubectl get sa deploy --namespace $NAMESPACE -o jsonpath='{.secrets[0].name}') --namespace $NAMESPACE -o jsonpath='{.data.token}'

Помещаем токены в новые переменные проекта в Gitlab: Settings - CI/CD - Variables
соответственно:
K8S_STAGE_CI_TOKEN
K8S_PROD_CI_TOKEN

Создаем Token: Settings - Repository - Deploy Tokens
Из него нам понадобятся Username и Password

Создаем секреты для авторизации Kubernetes в Gitlab registry - используем для этого Username и Password из предыдущего шага

kubectl create secret docker-registry gitlab-registry --docker-server=registry.gitlab.com --docker-username=gitlab+deploy-token-519505 --docker-password=rYrYaa__TTFirYoQ9_LB --docker-email=zaripovrazif@yandex.ru --namespace stage

kubectl create secret docker-registry gitlab-registry --docker-server=registry.gitlab.com --docker-username=gitlab+deploy-token-519505 --docker-password=rYrYaa__TTFirYoQ9_LB --docker-email=zaripovrazif@yandex.ru --namespace prod

kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gitlab-registry"}]}' -n stage

kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gitlab-registry"}]}' -n prod



kubectl apply --namespace stage -f ./homework/8.ci-cd/app/kube/postgres/

kubectl apply --namespace prod -f ./homework/8.ci-cd/app/kube/postgres/

Редактируем ingress.yaml - для host прописываем значение stage

kubectl apply --namespace stage -f ./homework/8.ci-cd/app/kube

Редактируем ingress.yaml - для host прописываем значение prod

kubectl apply --namespace prod -f ./homework/8.ci-cd/app/kube

kubectl get svc -A

!!! Смотрим EXTERNAL-IP у NAME=nginx-ingress-controller с TYPE=LoadBalancer.

146.185.240.172
