Запекать конфигурацию в образ — плохо. Тогда для каждого окружения нужен свой образ. Это противоречит идее «один образ — везде».

ConfigMap — это объект K8s для хранения конфигурации отдельно от кода и образа. Один образ + разные ConfigMap = разное поведение в разных окружениях.

Это просто пары ключ-значение. Хранятся в etcd, доступны любому Pod'у, которому вы его явно подключите. ConfigMap всегда привязан к конкретному namespace.

ConfigMap хранит данные в открытом виде в etcd. Для чувствительного есть Secret.

# Создание ConfigMap через манифест
Самый распространённый способ — описать ConfigMap в YAML-файле и применить его через `kubectl apply`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  LOG_LEVEL: "info"
  DB_HOST: "postgres.default.svc.cluster.local"
  DB_PORT: "5432"
  MAX_CONNECTIONS: "100"
```

Поле `data` содержит пары ключ-значение. Все значения — строки, даже числа нужно оборачивать в кавычки, иначе `kubectl apply` вернёт ошибку. Преобразование типов выполняет само приложение.

Применяем манифест: `kubectl apply -f app-config.yaml`.

Проверяем, что объект создан:

```
kubectl get configmap app-config
kubectl describe configmap app-config
```

Вывод `describe` покажет все ключи и их значения — это удобно для отладки.

# Создание ConfigMap через kubectl create
Для быстрого создания без YAML-файла используют флаги `--from-literal` и `--from-file`.

Из литеральных значений:

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=DB_HOST=postgres.default.svc.cluster.local
```

Из файла — Kubernetes возьмёт имя файла как ключ, а его содержимое как значение:

```bash
kubectl create configmap app-config --from-file=config.properties
```

# Два способа подключить ConfigMap
ConfigMap можно подать в Pod как переменные окружения или как файлы. Выбор зависит от того как приложение читает конфиг. Если под ссылается на ConfigMap с определенным именем, которого не существует в класетере и при этом параметр `optional` не указан, Под не запустится и останется в статусе Pending с ошибкой `CreateContainerConfigError`

**Способ 1: переменные окружения**
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app
        envFrom:
        - configMapRef:
            name: app-config
```

`envFrom` загружает все ключи ConfigMap как переменные окружения. Если в ConfigMap `LOG_LEVEL: info` — внутри контейнера будет `$LOG_LEVEL=info`.

`env.valueFrom` позволяет взять конкретный ключ из ConfigMap и присвоить его переменной с любым именем:
```yaml
env:
- name: LOG_LEVEL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: LOG_LEVEL
```


**Способ 2: монтирование как файлы**
```yaml
spec:
  containers:
  - name: app
    image: my-app
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: app-config
```

Каждый ключ ConfigMap станет отдельным файлом в `/etc/config/`, а значение — его содержимым:

```
/etc/config/LOG_LEVEL     → содержит "info"
/etc/config/DB_HOST       → содержит "postgres"
```

**Целый конфиг-файл**
ConfigMap может хранить весь файл целиком:
```yaml
data:
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend;
      }
    }
```

Смонтируете — в `/etc/nginx/nginx.conf` будет этот файл.

# Обновление ConfigMap
Вы обновили ConfigMap:

```bash
kubectl edit configmap app-config   # поменял LOG_LEVEL: info → debug
```

Ждёте, что приложение подхватит новый LOG_LEVEL. А оно — нет.

Если ConfigMap подключён как переменные окружения — изменения не подхватятся. Переменные окружения задаются при старте контейнера и не меняются в работающем процессе. Это фундаментальное ограничение Linux, не K8s.

Чтобы новый `LOG_LEVEL` применился — нужно перезапустить Pod'ы:

```bash
kubectl rollout restart deployment/my-app
```

Если ConfigMap подключён как файлы (volumeMount) — K8s обновит файлы в `/etc/config/` автоматически. Но с задержкой (до ~60 секунд — kubelet синхронизирует периодически).

Однако! Само приложение должно уметь перечитывать файл. Если оно прочитало конфиг один раз при старте и держит в памяти — обновление файла ничего не даст. Приложение должно следить за файлом (file watch) и перечитывать.

Практический вывод: в большинстве случаев после изменения ConfigMap делают `kubectl rollout restart deployment/my-app`. Это запускает rolling update — Pod'ы пересоздаются с новым ConfigMap. Безопасно (zero-downtime), предсказуемо.

Для редких случаев где нужен hot-reload без рестарта — есть специальные инструменты (Reloader operator).

# Автоматический reload конфига
Иногда хочется чтобы Pod автоматически перезапускался при изменении ConfigMap. Например, поменял настройку → сервис сам подхватил. Без этого легко забыть перезапустить Pod после изменения ConfigMap. Поменяли настройку, думаете «применилось», а старые Pod'ы работают со старым конфигом. Hash/Reloader защищают от этой ошибки.

Есть два подхода.

**Подход 1: hash-аннотация (ручной, но надёжный)**
Идея: добавить в Pod аннотацию с **хешем** ConfigMap. Когда ConfigMap меняется — меняется хеш — меняется Pod-template — K8s делает rolling update.

```yaml
spec:
  template:
    metadata:
      annotations:
        configmap-hash: "abc123def"   # хеш содержимого ConfigMap
```

В CI/CD вы вычисляете хеш ConfigMap и подставляете в аннотацию. Helm это делает автоматически:

```yaml
annotations:
  checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

При изменении ConfigMap хеш меняется → Deployment видит изменение template → rolling update. Чисто, предсказуемо.

**Подход 2: Reloader operator (автоматический)**
[Reloader](https://github.com/stakater/Reloader) — это оператор который следит за ConfigMap/Secret и автоматически перезапускает связанные Deployment'ы.

Установка: `helm install reloader stakater/reloader`

Помечаете Deployment аннотацией:

```yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
```

Теперь любое изменение ConfigMap или Secret который использует этот Deployment → Reloader делает `rollout restart`. Полностью автоматически.

# поле `immutable`
По умолчанию ConfigMap и Secret можно изменить в любой момент: достаточно выполнить `kubectl edit` или применить обновлённый манифест. Начиная с Kubernetes 1.21 оба ресурса поддерживают поле `immutable` (неизменяемый). Поле `immutable` стоит на верхнем уровне манифеста. Когда оно выставлено в `true`, Kubernetes запрещает менять поля `data` и `binaryData`, а также вернуть `immutable` обратно в `false`. Метки и аннотации менять можно.

Если попытаться обновить данные уже зафиксированного ресурса, API-сервер вернёт ошибку. Единственный способ «изменить» такой ресурс — удалить его и создать заново. Именно поэтому принято включать версию в имя: `app-config-v1`, `app-config-v2`. При обновлении конфигурации создаётся новый ресурс, а манифест Deployment обновляется так, чтобы ссылаться на новое имя.

Снижение нагрузки на API-сервер. Kubernetes периодически опрашивает ConfigMaps и Secrets, которые смонтированы как volumes, чтобы обнаружить изменения. Для неизменяемых ресурсов этот watch-механизм отключается — кластер с тысячами подов заметно выигрывает в производительности.