В Kubernetes большинство приложений запускают через `Deployment` — ресурс, который управляет набором одинаковых подов. Deployment не даёт подам никаких гарантий относительно имён или порядка запуска: поды при создании получают случайные суффиксы вроде `app-7d9f4b6c8-xkqzp`, и если под умирает, его замена появляется с совершенно другим именем. Deployment запускает все поды параллельно и не следит за порядком. Для stateless-сервисов (web, API) это нормально — все поды взаимозаменяемы, какой Pod обработает запрос, неважно. Но для stateful (базы данных, очереди, кластеры) у него проблемы. Каждый узел кластера имеет роль, хранит уникальные данные и должен быть доступен под постоянным адресом. Для таких случаев есть StatefulSet.

Для базы данных с репликацией важно кто есть кто:  
- `db-0` — master (принимает запись).  
- `db-1`, `db-2` — реплики (только чтение).

Со случайными именами это невозможно. Нужны стабильные, предсказуемые имена.

**Что даёт StatefulSet**
1. Стабильные имена (`{имя-ресурса}-{порядковый номер}`)

```
db-0
db-1
db-2
```

Имена предсказуемые, с индексами. `db-0` — всегда `db-0`, даже после перезапуска. Можно строить логику «db-0 это master» — обращаться к конкретному поду по стабильному DNS-имени вида `db-1.db-headless.default.svc.cluster.local`. Для работы стабильных DNS-имён StatefulSet требует `Headless Service` (сервис без ClusterIP, с `clusterIP: None`). Именно он регистрирует DNS-записи для каждого пода отдельно, а не балансирует трафик между ними.

2. Порядок создания и удаления
Pod'ы создаются по очереди: сначала `db-0`, потом `db-1`, потом `db-2`. Следующий под стартует только после того, как предыдущий перешёл в состояние `Running` и прошёл проверку готовности (readiness probe). Удаляются в обратном порядке. Это критично для кластерных приложений, где первый узел должен инициализировать кластер, а остальные — присоединиться к уже работающему лидеру.

3. Свой PVC у каждого Pod
Когда StatefulSet создаёт несколько реплик, каждый под должен иметь своё хранилище. Если подключить один общий PVC, все поды будут писать в одно место (что для RWO невозможно) — это разрушит изоляцию данных, необходимую, например, для узлов кластера базы данных. Именно для этого в StatefulSet существует поле `volumeClaimTemplates` (шаблон заявок на тома).

Каждый Pod получает собственное хранилище (через volumeClaimTemplates). `db-0` пишет в свой диск, `db-1` в свой. При перезапуске `db-0` подключается к тому же диску. Данные сохраняются за конкретным Pod'ом.

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

`serviceName: database` связывает StatefulSet с Headless Service. Поле `serviceName` обязательно — без него StatefulSet не сможет зарегистрировать стабильные DNS-записи для своих подов. Если его пропустить, Kubernetes вернёт ошибку валидации при применении манифеста.

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

Ключевая особенность StatefulSet — volumeClaimTemplates. `volumeClaimTemplates` — это список шаблонов PVC, встроенный прямо в манифест StatefulSet. При создании каждого пода Kubernetes автоматически создаёт отдельный PVC по этому шаблону и привязывает его только к данному поду.

Имя PVC формируется по правилу: `<имя-шаблона>-<имя-statefulset>-<порядковый-номер>`.

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
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```

Здесь `volumeClaimTemplates` описывает один шаблон с именем `data`. Kubernetes создаст три PVC по 10 Gi каждый, используя StorageClass `standard` для динамического выделения. StatefulSet создаст 3 Pod'а и для каждого — свой PVC:

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

Каждый из этих PVC живёт независимо. Если под `postgres-1` перезапустится или переедет на другую ноду, он снова получит именно `data-postgres-1` — не чужой том.

Помните проблему RWO + масштабирование из урока про PV? В StatefulSet её нет: каждый Pod имеет свой PVC (не общий). RWO означает «один Pod пишет в свой диск» — что и нужно. Поэтому облачные блочные диски (EBS/PD с RWO) прекрасно работают со StatefulSet.

Важно: при удалении StatefulSet PVC остаются (защита данных). Если хотите удалить и данные — удалите PVC вручную:

`kubectl delete pvc data-db-0 data-db-1 data-db-2`

Шаблон нельзя изменить после создания StatefulSet. Если нужно увеличить размер тома, придётся редактировать каждый PVC отдельно или пересоздавать StatefulSet.

Один StatefulSet может содержать несколько шаблонов. Например, отдельный том для данных и отдельный для логов:

```
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    resources:
      requests:
        storage: 50Gi
- metadata:
    name: logs
  spec:
    resources:
      requests:
        storage: 5Gi
```

В этом случае для каждого пода создаётся пара PVC: `data-postgres-0` и `logs-postgres-0`.