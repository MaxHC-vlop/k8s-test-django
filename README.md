# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Разворачиваем приложение в Minikube

- Установите [minikube](https://minikube.sigs.k8s.io/docs/start/) и [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/) .
- Запустите minikube:
```
minikube start
```
- Проверьте работоспособность кластера:
```bash
kubectl cluster-info
# если все хорошо
Kubernetes control plane is running at https://192.168.59.103:8443
CoreDNS is running at https://192.168.59.103:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```
- Настройте окружение Docker с помощью команды:
```
eval $(minikube docker-env)
```
- Соберите образ джанго приложения:
```
docker build -t django_test backend_main_django/
```
- После сборки убедитесь, что его видит minikube:
```bash
minikube image ls
# вывод
registry.k8s.io/pause:3.9
registry.k8s.io/kube-scheduler:v1.26.3
registry.k8s.io/kube-proxy:v1.26.3
registry.k8s.io/kube-controller-manager:v1.26.3
registry.k8s.io/kube-apiserver:v1.26.3
registry.k8s.io/etcd:3.5.6-0
registry.k8s.io/coredns/coredns:v1.9.3
gcr.io/k8s-minikube/storage-provisioner:v5
docker.io/library/django_test:latest # наш образ
```
- Установите [helm](https://helm.sh/docs/intro/install/) .
- Установите PostgreSQL с помощью Helm и запустите чарт бд в кластере, выполните следующие команды:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-postgres bitnami/postgresql
```
- Войдите в чарт с PostgreSQL:
```bash
# экспортируем пароль от постгрес
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default my-postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

# заходим в чарт
kubectl run my-postgres-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:15.3.0-debian-11-r7 --env="PGPASSWORD=$POSTGRES_PASSWORD"       --command -- psql --host my-postgres-postgresql -U postgres -d postgres -p 5432
```
- Создаем в чарте юзера и бд:
```
create database your_db_name;

create user your_user with encrypted password 'your_password';

ALTER USER your_user WITH SUPERUSER;
```
- Заполните `django-app-config.yaml` вашими данными:
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-app-config
data:
  SECRET_KEY: "123"
  DATABASE_URL: "postgres://db_user:db_password@chart_db_name:5432/db_name"
  DEBUG: "True"
  ALLOWED_HOSTS: "*"
```
- Получите IP-адрес MiniKube:
```
minikube ip
```
- Вставьте следующую строку в файл /etc/hosts, заменив <minikube_ip> на IP-адрес MiniKube, полученный на предыдущем шаге:
```
<minikube_ip>  star-burger.test
```
- Включите адоны для ingess:
```
minikube addons enable ingress
```
- Запустите манифесты для запуска приложения:
```
kubectl apply -f kubernets/
```
- Проверьте состояния кластера:
```
minikube dashboard
```
- Чтобы применить новые миграции к приложению, выполните следующую комнаду:
```
kubectl apply -f kubernets/migration.yaml
```