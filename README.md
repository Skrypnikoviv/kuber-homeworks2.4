# Домашнее задание к занятию «Управление доступом» - Скрыпников Илья

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

![image](https://github.com/user-attachments/assets/6f45576f-1830-4a07-a3c5-ac9418adc048)

![image](https://github.com/user-attachments/assets/bcf1beb5-cb7c-4998-99b4-aece9a68eb72)

![image](https://github.com/user-attachments/assets/43d037fc-800e-4458-a9eb-cf7f7c66aad3)

![image](https://github.com/user-attachments/assets/60aba52b-1a24-40af-b479-8a78d1fce32b)


