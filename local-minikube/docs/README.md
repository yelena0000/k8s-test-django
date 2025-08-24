## Развертывание в Minikube

### 1. Установите инструменты
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)

### 2. Запустите Minikube
```sh
minikube start
```
### 3. Создайте Secret для Django
Файл `kubernetes/django-secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secret
type: Opaque
data:
  SECRET_KEY: <base64-encoded-secret-key>
  DATABASE_URL: <base64-encoded-database-url>
```
Примените:
```bash
kubectl apply -f kubernetes/django-secret.yaml
```
### 4. Создайте ConfigMap для нечувствительных настроек
Файл `kubernetes/django-config.yaml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-config
data:
  DEBUG: "FALSE"
  ALLOWED_HOSTS: "127.0.0.1,localhost,star-burger.test"
```
Примените:
```bash
kubectl apply -f kubernetes/django-config.yaml
```
### 5. Разверните PostgreSQL в кластере
```bash
helm install postgresql oci://registry-1.docker.io/bitnamicharts/postgresql   --version 15.5.20   --set auth.username=test_user   --set auth.password=test_password   --set auth.database=test_db
```


### 6. Примените манифесты Django
```bash
kubectl apply -f kubernetes/django-deployment.yaml
kubectl apply -f kubernetes/django-service.yaml
kubectl apply -f kubernetes/django-ingress.yaml
kubectl apply -f kubernetes/django-clearsessions-cronjob.yaml
kubectl apply -f kubernetes/django-migrate-job.yaml
```

### 7. Проверьте сайт
После настройки `/etc/hosts` сайт будет доступен по адресу:
[http://star-burger.test](http://star-burger.test)

---