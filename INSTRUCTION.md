# INSTRUCTION.md - Розгортання MateApp на Kubernetes

## Розгортання

### Крок 1: Створити namespace
```bash
kubectl create namespace mateapp
```

### Крок 2: Розгорнути Deployment
```bash
kubectl apply -f deployment.yml
```

Перевірка:
```bash
kubectl get pods -n mateapp
kubectl get deployment -n mateapp
```

### Крок 3: Розгорнути HPA
```bash
kubectl apply -f hpa.yml
```

Перевірка:
```bash
kubectl get hpa -n mateapp
```

### Крок 4: Отримати доступ
```bash
kubectl port-forward svc/mateapp 8080:80 -n mateapp
```

Браузер: http://localhost:8080

---

## Resource Requests і Limits

```yaml
requests:
  memory: "128Mi"
  cpu: "100m"
limits:
  memory: "256Mi"
  cpu: "500m"
```

**Обґрунтування:**

Requests (гарантія):
- 128Mi RAM та 100m CPU на pod
- На 2 pods: 256Mi RAM потрібно
- Дешево та реалістично для типового додатку

Limits (максимум):
- 256Mi RAM та 500m CPU
- Потроєне від requests для запасу під навантаженням
- Захищає кластер від перевибору ресурсів
- Якщо перевищимо - pod перезавантажиться

---

## HPA Конфігурація

```yaml
minReplicas: 2
maxReplicas: 5
CPU: 70%
Memory: 80%
```

**Обґрунтування:**

Min 2, Max 5:
- 2 pods - мінімум для надійності
- 5 pods - максимум щоб контролювати витрати
- 3x різниця - достатня для більшості навантажень

CPU 70%, Memory 80%:
- 70% CPU - добрий баланс для масштабування
- 80% Memory - консервативніше (менше еластична)
- Запобігає перевебоям

Поведінка:
- Scale Up - негайне (при перевищенні порогу)
- Scale Down - повільне (чекаємо 5 хвилин)
- Це запобігає флаттерингу (швидким змінам)

---

## RollingUpdate Стратегія

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

**Обґрунтування:**

Zero-Downtime:
- Користувачі не відчують перебивань
- Якщо щось зломається - миттєво повертаємось
- Стара та нова версія працюють одночасно

maxSurge: 1:
- Під час оновлення може бути 3 pods (2 старих + 1 новий)
- Поступова заміна старих на нові

maxUnavailable: 0:
- Всі pods завжди доступні
- Нема перерв у сервісу

Процес:
1. [pod1 old] [pod2 old] 
2. [pod1 old] [pod2 old] [pod3 new] ← додали новий
3. [pod1 new] [pod2 old] [pod3 new] ← замінили старий
4. [pod1 new] [pod2 new] [pod3 new] ← готово!

---

## Доступ до додатку

### Port-Forward (тестування)
```bash
kubectl port-forward svc/mateapp 8080:80 -n mateapp
```
http://localhost:8080

### LoadBalancer Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mateapp
  namespace: mateapp
spec:
  type: LoadBalancer
  selector:
    app: mateapp
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f service.yml
kubectl get svc -n mateapp
```

### Ingress (production)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mateapp-ingress
  namespace: mateapp
spec:
  rules:
  - host: mateapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mateapp
            port:
              number: 80
```

---

## Моніторинг

Перевірити pods:
```bash
kubectl get pods -n mateapp
kubectl describe pod <pod-name> -n mateapp
kubectl logs <pod-name> -n mateapp
```

Моніторити HPA:
```bash
kubectl get hpa -n mateapp
kubectl describe hpa mateapp-hpa -n mateapp
kubectl top pods -n mateapp
```

Спостерігати за масштабуванням:
```bash
kubectl get hpa mateapp-hpa -n mateapp --watch
```

---

## Корисні команди

Подивитися конфігурацію:
```bash
kubectl get deployment mateapp -n mateapp -o yaml
```

Оновити image:
```bash
kubectl set image deployment/mateapp mateapp=nginx:1.22 -n mateapp
```

Скейлити вручну:
```bash
kubectl scale deployment mateapp --replicas=3 -n mateapp
```

Видалити всё:
```bash
kubectl delete namespace mateapp
```

---

## Troubleshooting

Pods не стартують:
```bash
kubectl describe pod <pod-name> -n mateapp
```

HPA показує "unknown" метрики:
```bash
kubectl get deployment metrics-server -n kube-system
```

Додаток не доступний:
```bash
kubectl get svc -n mateapp
```

---

**Розгортання завершено!**
