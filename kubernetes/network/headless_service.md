Обычный Service балансирует: обращаетесь к нему — попадаете на случайный Pod. Это нужно для stateless-сервисов (web, API) — всё равно на какой Pod.

Но иногда нужно обращаться к конкретному Pod'у. Например для баз данных с репликацией: «писать в master, читать из реплик» — тут важно знать кто есть кто.

Для этого есть `Headless Service`.

Headless Service создаётся с `clusterIP: None`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  clusterIP: None      # ← делает Service headless
  selector:
    app: database
  ports:
  - port: 5432
```

**Чем отличается**
Обычный Service: одно DNS-имя → один ClusterIP → балансировка на случайный Pod.

Headless Service: одно DNS-имя → список IP всех Pod'ов напрямую. Никакой балансировки, никакого ClusterIP.

```bash
nslookup database
# database.default.svc.cluster.local → 10.1.1.5  (Pod 1)
#                                    → 10.1.1.6  (Pod 2)
#                                    → 10.1.1.7  (Pod 3)
```

Получаете все IP. Дальше сами решаете к какому обращаться.

**Когда нужен**
- StatefulSet (БД, очереди): каждый Pod имеет уникальную роль. Headless даёт стабильные имена `database-0`, `database-1`, `database-2`.
- Клиентская балансировка: ваше приложение само решает к какому Pod'у идти (например gRPC-клиент).
- Service discovery: нужен список всех Pod'ов сервиса.

С Headless Service + StatefulSet каждый Pod получает предсказуемое имя:

```
database-0.database.default.svc.cluster.local
database-1.database.default.svc.cluster.local
database-2.database.default.svc.cluster.local
```

`database-0` — всегда тот же Pod (master), `database-1`, `database-2` — реплики.