# Домашнее задание к занятию «Управление доступом»

## Задание 1. Создайте конфигурацию для подключения пользователя

### 1. Создание SSL-сертификата для пользователя

Сначала создадим приватный ключ и CSR (Certificate Signing Request) для пользователя:

```bash
# Создаем приватный ключ
openssl genrsa -out user1.key 2048

# Создаем CSR
openssl req -new -key user1.key -out user1.csr -subj "/CN=user1/O=dev-group"
```

Теперь подпишем CSR с помощью кластерного CA (в MicroK8S сертификаты находятся в `/var/snap/microk8s/current/certs/`):

```bash
# Подписываем сертификат (адаптируйте пути для вашего кластера)
sudo openssl x509 -req -in user1.csr -CA /var/snap/microk8s/current/certs/ca.crt -CAkey /var/snap/microk8s/current/certs/ca.key -CAcreateserial -out user1.crt -days 365
```

### 2. Настройка конфигурационного файла kubectl

Добавим новую учетную запись в конфиг kubectl:

```bash
kubectl config set-credentials user1 --client-certificate=user1.crt --client-key=user1.key

kubectl config set-context user1-context --cluster=microk8s-cluster --user=user1
```

### 3. Создание ролей и привязок

Создадим манифесты для Role и RoleBinding:

`role-pod-reader.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch", "describe"]
```

`rolebinding-user1.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: user1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Применим манифесты:

```bash
kubectl apply -f role-pod-reader.yaml
kubectl apply -f rolebinding-user1.yaml
```

### 4. Проверка прав пользователя

Проверим доступ пользователя:

```bash
# Переключаем контекст
kubectl config use-context user1-context

# Проверяем доступ
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### 5. Скриншоты и вывод команд

Пример вывода при проверке прав:

```bash
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7c658794b9-2jq5h   1/1     Running   0          5d

$ kubectl describe pod nginx-7c658794b9-2jq5h
Name:         nginx-7c658794b9-2jq5h
Namespace:    default
...
(описание пода выводится успешно)

$ kubectl logs nginx-7c658794b9-2jq5h
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
...
(логи пода выводятся успешно)

$ kubectl get deployments
Error from server (Forbidden): deployments.apps is forbidden: User "user1" cannot list resource "deployments" in API group "apps" in the namespace "default"
(как и ожидалось, доступа к другим ресурсам нет)
```
