# Домашнее задание к занятию «Запуск приложений в K8S"

> Репозиторий: hw-36\
> Выполнил: Асадбек Асадбеков\
> Дата: сентябрь 2025

## Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

### Манифесты

#### Deployment
<details>
<summary>deployment-multitool.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
  labels:
    app: nginx-multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
      - name: multitool
        image: wbitt/network-multitool
        env:
        - name: HTTP_PORT
          value: "8080"
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
```
</details>

#### Service
<details>
<summary>service-multitool.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-multitool-service
spec:
  selector:
    app: nginx-multitool
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP
```
</details>

#### Тестовый Pod
<details>
<summary>test-multitool.yaml</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-multitool
spec:
  containers:
  - name: multitool
    image: wbitt/network-multitool
    command: ["sleep", "infinity"]
```
</details>

### Выполнение задания

1. Создаем Deployment:
```bash
microk8s kubectl apply -f deployment-multitool.yaml
```
![task1-deployment-created](https://github.com/asad-bekov/hw-36/blob/main/img/1.PNG)

2. Проверяем Pods до масштабирования:
```bash
microk8s kubectl get pods -l app=nginx-multitool
```
![task1-pods-before-scale](https://github.com/asad-bekov/hw-36/blob/main/img/2.PNG)

3. Масштабируем Deployment до 2 реплик:
```bash
microk8s kubectl scale deployment nginx-multitool --replicas=2
```
![task1-pods-after-scale](https://github.com/asad-bekov/hw-36/blob/main/img/3.PNG)

4. Создаем Service:
```bash
microk8s kubectl apply -f service-multitool.yaml
microk8s kubectl get service
```
![task1-service-created](https://github.com/asad-bekov/hw-36/blob/main/img/4.PNG)

5. Создаем тестовый Pod и проверяем доступ:
```bash
microk8s kubectl apply -f test-multitool.yaml
microk8s kubectl wait --for=condition=Ready pod/test-multitool --timeout=60s
microk8s kubectl exec test-multitool -- curl -s http://nginx-multitool-service
```
![task1-curl-access](https://github.com/asad-bekov/hw-36/blob/main/img/5.PNG)

6. Проверяем балансировку нагрузки:
```bash
microk8s kubectl exec test-multitool -- sh -c 'for i in 1 2 3 4; do echo "Request $i:"; curl -s http://nginx-multitool-service | grep "Hostname:"; done'
```
![task1-load-balancing](https://github.com/asad-bekov/hw-36/blob/main/img/6.PNG)

---

## Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

### Манифесты

#### Deployment
<details>
<summary>deployment-init.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-init
  labels:
    app: nginx-init
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-init
  template:
    metadata:
      labels:
        app: nginx-init
    spec:
      initContainers:
      - name: wait-for-service
        image: busybox:1.28
        command: ['sh', '-c', 'until nslookup nginx-init-service; do echo waiting for service; sleep 2; done;']
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```
</details>

#### Service
<details>
<summary>service-init.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-init-service
spec:
  selector:
    app: nginx-init
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP
```
</details>

### Выполнение задания

1. Создаем Deployment с Init-контейнером:
```bash
microk8s kubectl apply -f deployment-init.yaml
```
![task2-pod-before-service](https://github.com/asad-bekov/hw-36/blob/main/img/7.PNG)

2. Смотрим логи Init-контейнера:
```bash
microk8s kubectl logs $POD_NAME -c wait-for-service
```
![task2-init-logs](https://github.com/asad-bekov/hw-36/blob/main/img/8.PNG)

3. Создаем Service и проверяем Pod:
```bash
microk8s kubectl apply -f service-init.yaml
microk8s kubectl get service
microk8s kubectl get pods -l app=nginx-init
```
![task2-pod-after-service](https://github.com/asad-bekov/hw-36/blob/main/img/9.PNG)

4. Проверяем работу nginx:
```bash
microk8s kubectl exec $POD_NAME -- curl -s localhost
```
![task2-nginx-working](https://github.com/asad-bekov/hw-36/blob/main/img/10.PNG)

---

## Результаты выполнения

**Задание 1**
- Deployment с двумя контейнерами создан
- Deployment масштабирован до 2 реплик
- Создан Service для доступа к репликам
- Обеспечен доступ из тестового Pod через curl

**Задание 2**
- Deployment с Init-контейнером создан
- Init-контейнер проверяет доступность Service перед запуском
- Под запускается корректно после создания Service

---


## Установленное окружение
- Kubernetes: MicroK8S v1.32.8
- Контейнеры: nginx:alpine, wbitt/network-multitool, busybox:1.28
- Нода: Debian GNU/Linux 12, IP: 192.168.1.48


