Вот неприятный факт про K8s который многие не знают: по умолчанию любой Pod может обратиться к любому другому Pod'у в кластере. Никаких ограничений.

**Чем это плохо**
Представьте кластер:  
- Frontend-под (публичный, доступен из интернета).  
- API-под.  
- Database-под с пользовательскими данными.

По умолчанию frontend может напрямую подключиться к базе данных. И любой другой Pod тоже. Если frontend взломали — атакующий сразу лезет в БД, минуя API.

Это нарушает принцип defense in depth (защита эшелонами). База данных должна принимать соединения только от API, ни от кого больше.

**Решение: NetworkPolicy**
NetworkPolicy — это «firewall на уровне Pod'ов». Описывает кто с кем может общаться.

Логика:
- Какие Pod'ы могут приходить к этому Pod'у (ingress).
- Куда этот Pod может обращаться (egress).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-api
spec:
  podSelector:
    matchLabels:
      app: database     # эта политика про Pod'ы БД
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api      # принимать только от Pod'ов с app: api
```

Теперь к БД может обращаться только API. Frontend — нет, другие Pod'ы — нет.

NetworkPolicy работает только если CNI-плагин её поддерживает. Calico, Cilium — поддерживают. Стандартный flannel — нет. В managed-кластерах (EKS, GKE) обычно есть. Про это — отдельный шаг.

NetworkPolicy описывает трафик в двух направлениях: ingress (входящий к Pod'у) и egress (исходящий от Pod'а).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-policy
spec:
  podSelector:          # к каким Pod'ам применяется
    matchLabels:
      app: database
  policyTypes:
  - Ingress             # регулируем входящий
  - Egress              # и исходящий
  ingress:
  - from: [...]         # откуда можно приходить
  egress:
  - to: [...]           # куда можно ходить
```

`podSelector` — к кому применяется:

```yaml
podSelector:
  matchLabels:
    app: database
```

Эта политика влияет на Pod'ы с меткой `app: database`. Если `podSelector: {}` (пустой) — на все Pod'ы namespace.

`ingress` — кто может приходить:

```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: api
  ports:
  - protocol: TCP
    port: 5432
```

«К Pod'ам database могут приходить Pod'ы с `app: api` на порт 5432 по TCP». Всё остальное — блокируется.

`egress` — куда может ходить:

```yaml
egress:
- to:
  - podSelector:
      matchLabels:
        app: cache
  ports:
  - port: 6379
```

«Pod'ы database могут обращаться к Pod'ам cache на порт 6379». Всё остальное исходящее — блокируется.

Важный нюанс: как только есть политика — действует «запрет по умолчанию». Запомните правило:
- Нет ни одной NetworkPolicy на Pod → разрешено всё.
- Есть хотя бы одна NetworkPolicy с `policyTypes: Ingress` → разрешено только то что явно указано в ingress, остальное запрещено.

То есть NetworkPolicy не «добавляет запреты», а «переключает в режим белого списка». Создал политику с одним правилом ingress — всё остальное входящее автоматически закрыто.

**Паттерн deny-all + allow-specific**
Лучшая практика безопасности: сначала запрети всё, потом разреши нужное. Это называется «default deny». Разберём как это делается.

Шаг 1: запрет всего трафика в namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # ← все Pod'ы namespace
  policyTypes:
  - Ingress
  - Egress
  # нет ни ingress, ни egress правил = всё запрещено
```

`podSelector: {}` — применяется ко всем Pod'ам. Нет правил ingress/egress — значит весь трафик заблокирован. Pod'ы не могут ни принимать, ни отправлять.

После этого вообще ничего не работает. Это база — «нулевое доверие».

Шаг 2: точечно разрешаем нужное. Теперь добавляем политики которые открывают конкретные пути:

```yaml
# API может принимать от Ingress Controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-ingress
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
```

```yaml
# API может ходить в БД
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-db
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - port: 5432
```

И так для каждого нужного канала связи.

Частая ошибка: сделал default-deny egress — и сломал DNS. Pod'ы не могут резолвить имена (CoreDNS заблокирован). Нужно явно разрешить:

```yaml
egress:
- to:
  - namespaceSelector: {}
    podSelector:
      matchLabels:
        k8s-app: kube-dns
  ports:
  - protocol: UDP
    port: 53
```

После default-deny + точечные allow:
- Frontend ↔ Ingress: разрешено.
- API ↔ Frontend: разрешено.
- API ↔ Database: разрешено.
- Frontend ↔ Database: запрещено (нет правила).
- Любой ↔ что попало: запрещено.

Минимальная поверхность атаки.

В правилах NetworkPolicy (`from`/`to`) можно указывать источник/назначение тремя способами. Разберём.

`podSelector` — Pod'ы по метке:

```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: api
```

«Разрешить от Pod'ов с меткой `app: api` в том же namespace».

Важно: `podSelector` работает внутри того же namespace где сама политика. Pod'ы из другого namespace не подойдут под это правило, даже если у них метка `app: api`.

`namespaceSelector` — Pod'ы из определённых namespace:

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: frontend
```

«Разрешить от всех Pod'ов из namespace с меткой `name: frontend`».

Это для cross-namespace: например API в `backend` принимает от всего что в `frontend`.

Комбинация `podSelector + namespaceSelector`:

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: frontend
    podSelector:
      matchLabels:
        app: web
```

Обратите внимание: оба селектора в одном элементе списка (один `-`). Это значит И: «Pod'ы с `app: web` ИЗ namespace `frontend`». Самое точное правило.

Тонкость с дефисами:

```yaml
# ЭТО "И" (Pod web из namespace frontend):
- from:
  - namespaceSelector: { matchLabels: { name: frontend }}
    podSelector: { matchLabels: { app: web }}

# А ЭТО "ИЛИ" (любой Pod из frontend ИЛИ любой web в текущем ns):
- from:
  - namespaceSelector: { matchLabels: { name: frontend }}
  - podSelector: { matchLabels: { app: web }}
```

Разница в одном дефисе! Два дефиса = два отдельных условия (ИЛИ). Один дефис с двумя селекторами = одно условие (И). Это частый источник ошибок.

- `podSelector` — связь внутри одного namespace (API ↔ БД в том же ns).
- `namespaceSelector` — связь между namespace (принять от всего frontend-namespace).
- Оба вместе — точечно: конкретный Pod из конкретного namespace.