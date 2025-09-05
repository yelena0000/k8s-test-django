## Развертывание в Yandex Cloud (Kubernetes кластер yc-sirius-dev)

### Инструкции для деплоя Django-сайта в dev-окружении Kubernetes (`namespace: edu-yelena-malceva`). 

Сайт доступен по адресу:  [![Demo Version](https://img.shields.io/badge/Cайт-%E2%86%92_edu--yelena--malceva.yc--sirius--dev.pelid.team-blue)](https://edu-yelena-malceva.yc-sirius-dev.pelid.team)

Статика и медиа-файлы раздаются из Yandex Object Storage (`bucket: edu-yelena-malceva`).

### 1. Установите инструменты
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Lens Desktop](https://k8slens.dev/) (бесплатная версия для управления кластером)
- [Docker](https://www.docker.com/get-started/) (для сборки образа)

### 2. Получите доступ к кластеру
1. Войдите в [Yandex Cloud Console](https://console.cloud.yandex.ru).
2. Следуйте инструкциям в консоли для подключения к кластеру Kubernetes (yc-sirius-dev) через kubectl.
3. Установите Lens Desktop, подключите кластер, найдите namespace `edu-yelena-malceva`.

### 3. Создайте Secret для SSL PostgreSQL
Secret `postgres-ssl` нужен для подключения к базе данных. Используйте `root.crt` из Yandex Cloud (или secret `postgres`).
```bash
kubectl create secret generic postgres-ssl --from-file=root.crt=<path_to_root.crt> -n edu-yelena-malceva
```

### 4. Создайте Secret для Django
Файл `deploy/yc-sirius-dev/edu-yelena-malceva/django-secret`.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secret
  namespace: edu-yelena-malceva
type: Opaque
stringData:
  SECRET_KEY: "<generate-your-secret-key>"
```

Примените:
```bash
kubectl apply -f deploy/yc-sirius-dev/edu-yelena-malceva/django-secret.yaml
```

### 5. Создайте ConfigMap для Django

Файл `deploy/yc-sirius-dev/edu-yelena-malceva/django-config`.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
  namespace: edu-yelena-malceva
data:
  DJANGO_SETTINGS_MODULE: "webapp.settings"
  ALLOWED_HOSTS: "127.0.0.1,localhost,edu-yelena-malceva.yc-sirius-dev.pelid.team"
```

Примените:
```bash
kubectl apply -f deploy/yc-sirius-dev/edu-yelena-malceva/django-config.yaml
```

### 6. Настройте Nginx для reverse proxy
Файл `deploy/yc-sirius-dev/edu-yelena-malceva/main-nginx-config.yaml` уже настроен для прокси на Django и редиректа static/media на S3. 

Примените:
```bash
kubectl apply -f deploy/yc-sirius-dev/edu-yelena-malceva/main-nginx-config.yaml
```
Перезапустите Nginx:
```bash
kubectl rollout restart deployment main-nginx -n edu-yelena-malceva
```

### 7. Соберите и опубликуйте Docker-образ

Соберите образ с тегом по git-хэшу:

```bash
docker build -t yelena0000/django-site:$(git rev-parse --short HEAD) .
```

Опубликуйте:

```bash
docker push yelena0000/django-site:$(git rev-parse --short HEAD)
```

### 8. Разверните Django
Файлы `deploy/yc-sirius-dev/edu-yelena-malceva/django-service.yaml` и `django-deployment.yaml`:
```bash
kubectl apply -f deploy/yc-sirius-dev/edu-yelena-malceva/django-service.yaml
kubectl apply -f deploy/yc-sirius-dev/edu-yelena-malceva/django-deployment.yaml
```
Обновите image в `django-deployment.yaml` на новый git-хэш.

### 9. Настройте базу данных
Войдите в Django Pod:
```bash
kubectl exec -it $(kubectl get pod -n edu-yelena-malceva -l app=django -o name) -- bash
```
Выполните:
```bash
python manage.py migrate
python manage.py createsuperuser
```

### 10. Проверьте сайт

Сайт: `https://edu-yelena-malceva.yc-sirius-dev.pelid.team`

Админка: `https://edu-yelena-malceva.yc-sirius-dev.pelid.team/admin`

Логи: `kubectl logs -n edu-yelena-malceva -l app=django`

Статус Pod'ов: `kubectl get pods -n edu-yelena-malceva`

## Требования:

**PostgreSQL**: Managed Service, RW user `edu-yelena-malceva`, max 10 соединений.

**S3**: Bucket `edu-yelena-malceva` для статики/медиа.

**K8s**: Namespace `edu-yelena-malceva` с read-доступом к чужим namespaces.


## Переменные окружения:

`SECRET_KEY`: Из secret **django-secret**.

`DATABASE_URL`: Из secret **postgres-ssl** (ключ dsn).

`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_STORAGE_BUCKET_NAME`: Из secret **bucket**.

`ALLOWED_HOSTS`, `DJANGO_SETTINGS_MODULE`: Из ConfigMap **django-config**.


## Обновление:

Запушьте новый образ: `docker push yelena0000/django-site:<new-git-hash>`.

Обновите image в `django-deployment.yaml`.

Примените: 
```
kubectl apply -f django-deployment.yaml
```
Перезапустите Nginx: 
```
kubectl rollout restart deployment main-nginx -n edu-yelena-malceva
```


**Проверка**:

Статус (все Running): 
```
kubectl get pods -n edu-yelena-malceva
```

Логи/трейсбеки (или через Lens): 
```
kubectl logs -n edu-yelena-malceva -l app=django 
```
