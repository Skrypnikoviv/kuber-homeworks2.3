# Домашнее задание к занятию «Конфигурация приложений»

## Задание 1. Deployment с Nginx и Multitool с ConfigMap

### 1. Создаем Deployment с двумя контейнерами (nginx и multitool)

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
        - name: html-volume
          mountPath: /usr/share/nginx/html
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 80
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: html-volume
        configMap:
          name: html-content
```

### 2. Создаем ConfigMap для конфигурации Nginx и веб-страницы

```yaml
# configmap-nginx.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
        listen       80;
        server_name  localhost;

        location / {
            root   /usr/share/nginx/html;
            index  index.html;
        }
    }
```

```yaml
# configmap-html.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: html-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>My Web Page</title>
    </head>
    <body>
        <h1>Welcome to my Kubernetes Web App!</h1>
        <p>This page is served from a ConfigMap.</p>
    </body>
    </html>
```

### 3. Создаем Service для доступа к приложению

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

### 4. Применяем манифесты

```bash
kubectl apply -f configmap-nginx.yaml
kubectl apply -f configmap-html.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

### 5. Проверяем работу

```bash
kubectl get pods
kubectl exec -it <pod-name> -c multitool -- curl http://localhost
```

Вывод должен показать HTML-страницу из ConfigMap.

## Задание 2. Веб-приложение с HTTPS

### 1. Создаем Deployment для Nginx

```yaml
# https-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: https-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: https-app
  template:
    metadata:
      labels:
        app: https-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 443
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
        - name: html-volume
          mountPath: /usr/share/nginx/html
        - name: cert-volume
          mountPath: /etc/nginx/ssl
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-https-config
      - name: html-volume
        configMap:
          name: https-html-content
      - name: cert-volume
        secret:
          secretName: tls-secret
```

### 2. Создаем ConfigMap для конфигурации и HTML

```yaml
# configmap-https-nginx.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-https-config
data:
  default.conf: |
    server {
        listen       443 ssl;
        server_name  localhost;

        ssl_certificate      /etc/nginx/ssl/tls.crt;
        ssl_certificate_key  /etc/nginx/ssl/tls.key;

        location / {
            root   /usr/share/nginx/html;
            index  index.html;
        }
    }
```

```yaml
# configmap-https-html.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: https-html-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Secure Page</title>
    </head>
    <body>
        <h1>Welcome to my Secure Kubernetes Web App!</h1>
        <p>This page is served via HTTPS.</p>
    </body>
    </html>
```

### 3. Генерируем самоподписанный сертификат

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=my-app.example.com"
```

### 4. Создаем Secret для сертификата

```bash
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```

### 5. Создаем Service и Ingress

```yaml
# https-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: https-service
spec:
  selector:
    app: https-app
  ports:
    - protocol: TCP
      port: 443
      targetPort: 443
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: https-ingress
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
  - hosts:
    - my-app.example.com
    secretName: tls-secret
  rules:
  - host: my-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: https-service
            port:
              number: 443
```

### 6. Применяем манифесты

```bash
kubectl apply -f configmap-https-nginx.yaml
kubectl apply -f configmap-https-html.yaml
kubectl apply -f https-deployment.yaml
kubectl apply -f https-service.yaml
kubectl apply -f ingress.yaml
```

### 7. Проверяем доступность

```bash
# Добавляем запись в /etc/hosts или используем curl с заголовком Host
curl -k https://my-app.example.com --resolve my-app.example.com:443:<ingress-ip>
```
