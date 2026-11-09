Когда контейнер перезапускается, всё что было записано внутри него — исчезает. Файловая система контейнера живёт только пока живёт контейнер.

**Почему просто volume из Docker не хватает**
В Docker мы использовали volumes. В K8s сложнее — кластер из многих нод. Pod может переехать с ноды A на ноду B. Если данные лежали локально на ноде A — на ноде B их нет.

Нужно сетевое хранилище, доступное с любой ноды: облачные диски (AWS EBS, GCP PD), NFS, Ceph. И абстракция чтобы работать с ними единообразно.

**Три уровня абстракции K8s**
K8s вводит три объекта для хранилищ:
1. PersistentVolume (PV) — само хранилище. «Вот диск на 10 ГБ, тип AWS EBS». Создаётся админом или автоматически. Это ресурс кластера.
2. PersistentVolumeClaim (PVC) — заявка на хранилище. «Мне нужно 5 ГБ с доступом на запись». Создаёт разработчик. K8s находит подходящий PV и связывает.
3. StorageClass — шаблон для автосоздания PV. «Когда кто-то просит хранилище типа fast-ssd — создаётся AWS EBS gp3». Автоматизация.

Разработчик пишет PVC («хочу 5 ГБ»), не зная деталей хранилища. K8s находит/создаёт PV и связывает. Разделение забот: разработчик просит, инфраструктура предоставляет.

```
Разработчик: PVC "хочу 5Gi RWO"
        ↓
K8s ищет подходящий PV (или StorageClass создаёт новый)
        ↓
PV связывается с PVC (Bound)
        ↓
Pod монтирует PVC как volume
        ↓
Данные пишутся на реальный диск, переживают перезапуск Pod
```

# PersistentVolume (PV)
Описывает конкретное хранилище:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /data         # для теста; в проде — awsElasticBlockStore, nfs, csi
```

PV — это ресурс уровня кластера (не в namespace). Создаётся админом или динамически.

# PersistentVolumeClaim (PVC)
«Заявка» от приложения:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

PVC говорит «мне нужно 5Gi с доступом RWO». K8s ищет PV который подходит (≥5Gi, RWO) и связывает их. Статус PVC становится `Bound`.

# StorageClass — динамическое создание
Создавать PV вручную для каждого приложения — морока. StorageClass автоматизирует:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com    # драйвер AWS EBS
parameters:
  type: gp3
```

Теперь в PVC указываете StorageClass:
```yaml
spec:
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 5Gi
```

K8s сам создаёт AWS EBS-диск на 5Gi типа gp3 и связывает. Никакого ручного PV.

# Использование в Pod
PVC монтируется в Pod как volume:

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /var/lib/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-claim
```

`/var/lib/data` внутри контейнера = реальный диск. Что записали — переживёт перезапуск.

# Полная картина
```
StorageClass (шаблон) ──создаёт──> PV (реальный диск)
                                      ↕ binding
                                    PVC (заявка) ──монтируется──> Pod
```

# Access Modes: кто и как может читать/писать
У хранилища есть режим доступа (access mode). Он определяет сколько Pod'ов могут одновременно использовать том и как.

Три режима:
**ReadWriteOnce (RWO)**
Том может быть смонтирован на запись только одной нодой одновременно. Pod'ы на этой ноде могут писать, но не Pod'ы на другой.

Это режим облачных блочных дисков (AWS EBS, GCP PD). Блочный диск физически подключается к одной виртуалке.

Когда: база данных (один Pod пишет в свой диск). 99% случаев — RWO.

**ReadOnlyMany (ROX)**
Том может быть смонтирован на чтение многими нодами одновременно. Никто не пишет, все читают.

Когда: общие read-only данные (статика, ML-модели которые много Pod'ов читают).

**ReadWriteMany (RWX)**
Том может быть смонтирован на запись многими нодами одновременно. Несколько Pod'ов пишут в один том.

Когда: общее файловое хранилище (загрузки пользователей, shared cache). Требует сетевую файловую систему (NFS, CephFS, EFS) — обычный блочный диск так не умеет.

Важное ограничение. Не все хранилища поддерживают все режимы:
- AWS EBS, GCP PD (блочные диски): только RWO.
- NFS, AWS EFS, CephFS (сетевые ФС): поддерживают RWX.

Если попросите RWX от EBS — PVC не свяжется, Pod зависнет в Pending.

Частая ошибка с RWO и масштабированием:
```yaml
# Deployment с 3 репликами + RWO PVC:
spec:
  replicas: 3
  template:
    spec:
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: shared-claim   # RWO!
```

Проблема: RWO = только одна нода пишет. Если 3 реплики раскиданы по разным нодам — только Pod'ы на одной ноде получат доступ, остальные зависнут.

Решения:  
- Для БД — не Deployment, а StatefulSet (каждый Pod свой PVC). Про это следующий урок.  
- Для shared-данных — RWX (NFS/EFS).

Если видите Pod, навечно зависший в Pending с упоминанием volume — почти наверняка кто-то попросил ReadWriteMany от облачного блочного диска, который так не умеет. Это классика.

# Dynamic provisioning: StorageClass создаёт PV сам
Создавать PV вручную для каждого приложения неудобно. Dynamic provisioning создаёт PV по запросу.

Без динамики (статика). Раньше админ заранее создавал PV:
```yaml
kind: PersistentVolume
metadata:
  name: pv-1
spec:
  capacity: { storage: 10Gi }
  # ...
```

Разработчик писал PVC, K8s искал подходящий PV среди заранее созданных. Если нет подходящего — PVC висит Pending. Админ должен создать вручную. Медленно.

С динамикой (StorageClass).
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
allowVolumeExpansion: true
```

Теперь когда разработчик пишет PVC:
```yaml
spec:
  storageClassName: standard     # или не указывает — берётся default
  resources:
    requests:
      storage: 5Gi
```

K8s на лету:  
1. Видит PVC без подходящего PV.  
2. Смотрит StorageClass `standard`.  
3. Вызывает provisioner (`ebs.csi.aws.com`) → создаёт реальный AWS EBS на 5Gi.  
4. Создаёт PV для этого диска.  
5. Связывает PV с PVC.

Всё за секунды, без человека.

Аннотация `is-default-class: "true"` делает StorageClass дефолтным. PVC без явного `storageClassName` использует его. Удобно — разработчику не надо знать названия классов.

Разные классы под разные нужды:
```
fast-ssd    → NVMe SSD (БД, нужна скорость)
standard    → обычный SSD (большинство)
cheap-hdd   → HDD (бэкапы, логи, не нужна скорость)
```

Разработчик выбирает класс под задачу. Цена/скорость на его усмотрение.

`allowVolumeExpansion: true` позволяет **увеличивать** размер тома потом:

```bash
kubectl edit pvc my-claim   # меняешь 5Gi → 20Gi
```

Диск расширится без пересоздания (если провайдер поддерживает).

# Reclaim Policy
Когда вы удаляете PVC, что происходит с реальными данными на диске? Зависит от Reclaim Policy у PV.

**Delete (по умолчанию для динамики)**
```yaml
spec:
  persistentVolumeReclaimPolicy: Delete
```

Удалил PVC → K8s удаляет PV → облачный провайдер удаляет реальный диск. Данные исчезают навсегда.

Плюс: не копятся брошенные диски (за которые вы платите).
Минус: случайно удалил PVC БД → потерял данные.

Это дефолт для динамически созданных PV. Будь осторожен с production-БД.

**Retain (для важных данных)**

```yaml
spec:
  persistentVolumeReclaimPolicy: Retain
```

Удалил PVC → PV остаётся (в статусе `Released`) → реальный диск сохраняется. Данные на месте.

Плюс: защита от случайной потери данных.  
Минус: надо вручную чистить освободившиеся PV (иначе копятся).

Когда: production-базы данных, критичные данные. Лучше потом руками удалить лишний диск, чем потерять данные навсегда.

Задать для StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: db-storage
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain        # ← для БД
```

Все PVC через этот класс получат Retain.

После удаления PVC при Retain, PV переходит в статус `Released`. Он не может автоматически связаться с новым PVC (хранит ссылку на старый). Чтобы переиспользовать:
1. Либо удалить PV (`kubectl delete pv`) и реальный диск.
2. Либо вручную «освободить»: убрать `claimRef` из PV (`kubectl edit pv`) — тогда он снова `Available`.

**На практике**
Для production-БД:  
- StorageClass с `reclaimPolicy: Retain`.  
- Регулярные бэкапы (Reclaim Policy не заменяет бэкапы!).  
- Осторожность с `kubectl delete pvc`.

И да — Reclaim Policy это не бэкап. Retain спасёт от случайного `kubectl delete pvc`, но не от того, что диск физически сдох. Бэкапы делай отдельно, всегда.