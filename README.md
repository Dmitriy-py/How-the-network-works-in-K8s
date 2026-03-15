# Домашнее задание к занятию «Как работает сеть в K8s»

## ` Дмитрий Климов `

# Цель задания
## Настроить сетевую политику доступа к подам.

## Чеклист готовности к домашнему заданию
   1. Кластер K8s с установленным сетевым плагином Calico.

## Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

  1. Документация Calico.
  2. Network Policy.
  3. About Network Policy.

## Задание 1. Создать сетевую политику или несколько политик для обеспечения доступа

### 1. Создать deployment'ы приложений frontend, backend и cache и соответсвующие сервисы.
### 2. В качестве образа использовать network-multitool.
### 3. Разместить поды в namespace App.
### 4. Создать политики, чтобы обеспечить доступ frontend -> backend -> cache. Другие виды подключений должны быть запрещены.
### 5. Продемонстрировать, что трафик разрешён и запрещён.

## Ответ:

## **Цель:** Настроить сетевую политику доступа к подам, обеспечив строгое следование потоку: `Frontend -> Backend -> Cache`.

## **Сетевой плагин:** Calico (Установка и настройка Calico была выполнена для корректной работы политик).

---

## 1. Манифесты (YAML)

### 1.1. Namespace и Deployment (`deployments.yaml`, `namespace.yaml`)

Поды создаются в неймспейсе `app` с необходимыми метками (`app: frontend`, `app: backend`, `app: cache`).

### 1.1. Namespace (`namespace.yaml`)
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app
```
### 1.2. Deployments (`deployments.yaml`)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: network-multitool
        image: praqma/network-multitool
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: network-multitool
        image: praqma/network-multitool
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cache
  template:
    metadata:
      labels:
        app: cache
    spec:
      containers:
      - name: network-multitool
        image: praqma/network-multitool
        ports:
        - containerPort: 80
```
### 1.3. Services (`services.yaml`)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: app
spec:
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: app
spec:
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: cache-svc
  namespace: app
spec:
  selector:
    app: cache
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### 1.4. Network Policies (`network-policies.yaml`)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: app
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: app
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-cache
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: cache
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
```
## 2. Проверка работы

### 2.1. Развертывание
```yaml
kubectl apply -f namespace.yaml
kubectl apply -f deployments.yaml
kubectl apply -f services.yaml
kubectl apply -f network-policies.yaml
```

### 2.2. Демонстрация

### Frontend -> Backend (Успешно): `kubectl exec -it deployment/frontend -n app -- curl -m 3 backend-svc`













