### 1. **Создать ServiceAccount**

Создадим сервисный аккаунт, от имени которого будет выполняться запрос:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-deleter
  namespace: default
```

Применяем в кластере:
```bash
kubectl apply -f serviceaccount.yaml
```

### 2. **Создать Role или ClusterRole**

Так как скрипт должен удалять поды, создадим **Role** (если нужен доступ только в одном неймспейсе) или **ClusterRole** (если нужен доступ ко всем неймспейсам).

#### **Вариант 1: Role (доступ в одном неймспейсе)**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-deleter-role
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["delete"]
```

#### **Вариант 2: ClusterRole (доступ ко всем неймспейсам)**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-deleter-clusterrole
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["delete"]
```

---
### 3. **Привязать Role к ServiceAccount**

Теперь создадим **RoleBinding** или **ClusterRoleBinding**, чтобы связать сервисный аккаунт с ролью.

#### **Если использовалась Role:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-deleter-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: pod-deleter
    namespace: default
roleRef:
  kind: Role
  name: pod-deleter-role
  apiGroup: rbac.authorization.k8s.io
```

Если использовалась ClusterRole:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-deleter-clusterbinding
subjects:
  - kind: ServiceAccount
    name: pod-deleter
    namespace: default
roleRef:
  kind: ClusterRole
  name: pod-deleter-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

---
### 4. **Получить токен для аутентификации**

Создадим токен для ServiceAccount:

```bash
kubectl create token pod-deleter --namespace default
```

Этот токен можно использовать для аутентификации при выполнении API-запросов.

---

### 5. **Выполнить запрос на удаление пода**

Теперь скрипт может отправлять HTTP-запросы к API Server Kubernetes. Например, с использованием `curl`:

```bash
TOKEN=$(kubectl create token pod-deleter --namespace default)
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
POD_NAME="your-pod-name"
NAMESPACE="default"

curl -X DELETE "$APISERVER/api/v1/namespaces/$NAMESPACE/pods/$POD_NAME" \
     -H "Authorization: Bearer $TOKEN" \
     -H "Accept: application/json" \
     --insecure  # Удалить --insecure, если у вас настроен правильный TLS
```

---
### Альтернативный вариант: Использование `kubeconfig`

Можно также создать kubeconfig с токеном ServiceAccount и использовать его в скрипте. Для этого:
```bash
kubectl config set-credentials pod-deleter-user --token=$(kubectl create token pod-deleter --namespace default)

kubectl config set-context pod-deleter-context --cluster=$(kubectl config view --minify -o jsonpath='{.clusters[0].name}') --user=pod-deleter-user --namespace=default

kubectl config use-context pod-deleter-context
```

Затем в скрипте можно просто вызывать:
```bash
kubectl delete pod $POD_NAME --namespace=$NAMESPACE
```
