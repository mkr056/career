Deployment отлично подходит для stateless-сервисов (web, API). Но для stateful (базы данных, очереди, кластеры) у него проблемы. Для них есть StatefulSet.

Deployment создаёт Pod'ы с случайными именами:
```
web-7d4b8c9f-x7k2p
web-7d4b8c9f-9mn4q
web-7d4b8c9f-pl2zx
```

Они взаимозаменяемы — какой Pod обработает запрос, неважно. Для web это норм.

Но для базы данных с репликацией важно кто есть кто:  
- `db-0` — master (принимает запись).  
- `db-1`, `db-2` — реплики (только чтение).

Со случайными именами это невозможно. Нужны стабильные, предсказуемые имена.

**Что даёт StatefulSet**
1. Стабильные имена

```
db-0
db-1
db-2
```

Имена предсказуемые, с индексами. `db-0` — всегда `db-0`, даже после перезапуска. Можно строить логику «db-0 это master».

2. Порядок создания и удаления
Pod'ы создаются по очереди: сначала `db-0`, потом `db-1`, потом `db-2`. Удаляются в обратном порядке. Для кластеров БД это критично — реплики должны подключаться к уже готовому master.

3. Свой PVC у каждого Pod

Каждый Pod получает собственное хранилище (через volumeClaimTemplates). `db-0` пишет в свой диск, `db-1` в свой. При перезапуске `db-0` подключается к тому же диску. Данные сохраняются за конкретным Pod'ом.

В Deployment все реплики делили бы один PVC (что для RWO невозможно).

4. Стабильный сетевой identity
С Headless Service каждый Pod получает стабильное DNS-имя:
```
db-0.database.default.svc.cluster.local
db-1.database.default.svc.cluster.local
```

StatefulSet почти всегда работает в паре с Headless Service. Вместе они дают каждому Pod стабильное DNS-имя.

```yaml
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  clusterIP: None          # ← headless
  selector:
    app: database
  ports:
  - port: 5432

---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: database    # ← ссылка на headless service
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:15
```

`serviceName: database` связывает StatefulSet с Headless Service.

Каждый Pod получает предсказуемое имя:

```
db-0.database.default.svc.cluster.local
db-1.database.default.svc.cluster.local
db-2.database.default.svc.cluster.local
```

Формат: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`.

Теперь можно настроить репликацию по именам:  
- Приложение знает: master = `db-0.database`.  
- Реплики подключаются к master по имени `db-0.database`.  
- Запись идёт в `db-0.database:5432`, чтение распределяется по `db-1`, `db-2`.

Имена стабильны — `db-0` после перезапуска остаётся `db-0`, подключается к тому же диску, сохраняет роль master.

Обычный Service тоже можно добавить. Headless Service даёт прямые имена Pod'ов. Но для балансировки чтения часто добавляют ещё обычный ClusterIP Service:

```yaml
# Обычный Service для read-балансировки
apiVersion: v1
kind: Service
metadata:
  name: database-read
spec:
  selector:
    app: database
  ports:
  - port: 5432
```

`database-read` балансирует на все реплики (для read-запросов). А `db-0.database` — прямой доступ к master (для записи).

Ключевая особенность StatefulSet — volumeClaimTemplates. Это шаблон по которому каждый Pod получает собственный PVC.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: database
  replicas: 3
  selector:
    matchLabels: { app: database }
  template:
    metadata:
      labels: { app: database }
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:        # ← шаблон PVC
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 10Gi
```

StatefulSet создаст 3 Pod'а и для каждого — свой PVC:

```
Pod db-0  →  PVC data-db-0  →  свой диск 10Gi
Pod db-1  →  PVC data-db-1  →  свой диск 10Gi
Pod db-2  →  PVC data-db-2  →  свой диск 10Gi
```

Каждый Pod пишет в свой диск. `db-0` не видит данные `db-1`. Это правильно — каждая реплика БД хранит свою копию данных.

Магия: если `db-0` упал и пересоздался — он подключается к тому же PVC `data-db-0`, к тем же данным. Не к новому пустому диску.

```
db-0 упал
   ↓
StatefulSet создаёт новый db-0
   ↓
он подключается к существующему PVC data-db-0
   ↓
данные на месте
```

Это невозможно с Deployment — там Pod'ы взаимозаменяемы, нет привязки «этот Pod → этот диск».

Помните проблему RWO + масштабирование из урока про PV? В StatefulSet её нет: каждый Pod имеет свой PVC (не общий). RWO означает «один Pod пишет в свой диск» — что и нужно. Поэтому облачные блочные диски (EBS/PD с RWO) прекрасно работают со StatefulSet.

Важно: при удалении StatefulSet PVC остаются (защита данных). Если хотите удалить и данные — удалите PVC вручную:

`kubectl delete pvc data-db-0 data-db-1 data-db-2`

