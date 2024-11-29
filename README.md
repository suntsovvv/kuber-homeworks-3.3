# Домашнее задание к занятию «Как работает сеть в K8s»

### Цель задания

Настроить сетевую политику доступа к подам.

-----

### Задание 1. Создать сетевую политику или несколько политик для обеспечения доступа

1. Создать deployment'ы приложений frontend, backend и cache и соответсвующие сервисы.
2. В качестве образа использовать network-multitool.
3. Разместить поды в namespace App.
4. Создать политики, чтобы обеспечить доступ frontend -> backend -> cache. Другие виды подключений должны быть запрещены.
5. Продемонстрировать, что трафик разрешён и запрещён.


Создатл deployment'ы приложений frontend, backend и cache и соответсвующие сервисы:   
Манифест одного из них остальные по аналогии:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: app
  labels:
    app: frontend
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
      - name: multitool
        image: wbitt/network-multitool
        env:
          - name: HTTP_PORT
            value: "8080"
        ports:
         - containerPort: 8080

---
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
    name: multitool
    port: 8080
    targetPort: 8080


```
Проверяю:
```bash
user@microk8s:~/kuber-homeworks-3.3$ kubectl get deployments -o wide
NAME       READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                    SELECTOR
backend    1/1     1            1           3h15m   multitool    wbitt/network-multitool   app=backend
cache      1/1     1            1           3h15m   multitool    wbitt/network-multitool   app=cache
frontend   1/1     1            1           3h25m   multitool    wbitt/network-multitool   app=frontend
user@microk8s:~/kuber-homeworks-3.3$ kubectl get svc
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
backend-svc    ClusterIP   10.152.183.102   <none>        8080/TCP   3h19m
cache-svc      ClusterIP   10.152.183.130   <none>        8080/TCP   3h18m
frontend-svc   ClusterIP   10.152.183.222   <none>        8080/TCP   3h20m
```


Создал политики, чтобы обеспечить доступ frontend -> backend -> cache. Другие виды подключений должны быть запрещены.:   
Блокирующая политика по умолчанию:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: app
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Ingress
```
Разрешающая политика от frontend до backend:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-backend
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: frontend 
```
Разрешающая политика от backend до cache:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-cache
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: cache  
  policyTypes:
  - Ingress    
  ingress:
    - from:
      - podSelector:
          matchLabels:
```
Проверяю:
```bash
user@microk8s:~/kuber-homeworks-3.3$ kubectl get networkpolicies
NAME                   POD-SELECTOR   AGE
access-backend         app=backend    166m
access-cache           app=cache      12s
default-deny-ingress   <none>         170m
```

Демонстрирую, что трафик разрешён и запрещён.   
_frontend_
```bash
user@microk8s:~/kuber-homeworks-3.3$ kubectl exec deployments/frontend -it -- bash
frontend-5694589b48-8krvb:/# curl backend-svc:8080
WBITT Network MultiTool (with NGINX) - backend-847c446948-rw4tg - 10.1.128.250 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
frontend-5694589b48-8krvb:/# curl cache-svc:8080
^C
frontend-5694589b48-8krvb:/# curl cache-svc:8080 -vv
* processing: cache-svc:8080
*   Trying 10.152.183.130:8080...
```
_backend_
```bash
user@microk8s:~/kuber-homeworks-3.3$ kubectl exec deployments/backend -it -- bash
backend-847c446948-rw4tg:/# curl cache-svc:8080 
WBITT Network MultiTool (with NGINX) - cache-667ff55c4b-26gsl - 10.1.128.251 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
backend-847c446948-rw4tg:/# curl frontend-svc:8080 -vv
* processing: frontend-svc:8080
*   Trying 10.152.183.222:8080...

```
_cashe_
```bash
user@microk8s:~/kuber-homeworks-3.3$ kubectl exec deployments/backend -it -- bash
backend-847c446948-rw4tg:/# curl cache-svc:8080 
WBITT Network MultiTool (with NGINX) - cache-667ff55c4b-26gsl - 10.1.128.251 - HTTP: 8080 , HTTPS: 443 . (Formerly praqma/network-multitool)
backend-847c446948-rw4tg:/# curl frontend-svc:8080 -vv
* processing: frontend-svc:8080
*   Trying 10.152.183.222:8080...
```

Ссылка на манифесты https://github.com/suntsovvv/kuber-homeworks-3.3.git

------
