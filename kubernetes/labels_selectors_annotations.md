В K8s ресурсы не знают друг о друге напрямую. Service не знает про конкретный Pod, ReplicaSet не знает про конкретный Pod. Связь — через `labels` (метки) и `selectors` (что искать).

`Labels` — метки на ресурсах. Любому ресурсу можно навесить метки:
```yaml
metadata:
  name: my-web
  labels:
    app: web
    tier: frontend
    env: production
    version: v1.2.3
```

labels — это ключ-значение. Что угодно.

Флаг `-l` фильтрует ресурсы по меткам. Несколько меток через запятую означают «И». Пример: `kubectl get pods -l tier=backend,env=prod`.

`Selectors` — поиск по меткам. В Service вы не указываете конкретные Pod'ы. Вы пишете критерий:

```yaml
kind: Service
spec:
  selector:
    app: web
    env: production
```

Service автоматически найдёт все Pod'ы у которых одновременно `app=web` и `env=production`. Если новый Pod появился с такими метками — Service автоматически начнёт направлять на него трафик.

Так работает много чего в K8s
- Service находит Pod'ы по selector.
- ReplicaSet считает свои Pod'ы по selector.
- NetworkPolicy разрешает трафик к Pod'ам с такими-то метками.
- HorizontalPodAutoscaler масштабирует Deployment'ы с такими-то метками.

Это очень гибкая модель. Хотите временно отключить Pod от Service — поменяли у него метку, Service его «забыл». Без перезапуска, без удаления.

Стандартный набор который рекомендует K8s:
```yaml
labels:
  app.kubernetes.io/name: my-api
  app.kubernetes.io/version: "1.2.3"
  app.kubernetes.io/component: backend
  app.kubernetes.io/part-of: my-platform
  app.kubernetes.io/managed-by: helm
```

Длинно, но систематично. Многие инструменты (Helm, Prometheus) знают про эти стандартные метки.

В простых проектах ограничиваются коротким:

```yaml
labels:
  app: my-api
  env: prod
```

Главное — единообразно во всём проекте.

`Annotations` — метаданные без участия в селекции. Похожи на labels — тоже ключ-значение. Но используются по-другому.

`Labels` — для поиска и фильтрации через selectors. Должны быть короткими, осмысленными.
`Annotations` — для хранения произвольных метаданных. Длинные строки, JSON, что угодно. Не участвуют в selectors.

```yaml
metadata:
  name: my-pod
  labels:
    app: web              # для поиска через selector
    env: prod
  annotations:
    deployed-by: "alice@example.com"
    git-commit: "abc123def456"
    description: "Главный API-сервис обрабатывающий заказы"
    config-checksum: "sha256:..."
```

**Когда что использовать**

`Label` если: нужно по этому искать.  
- `app: web` — Service ищет по этому label.  
- `env: prod` — отдельная политика для prod.

`Annotation` если: это информация, не для поиска.  
- Кто задеплоил, когда, какой commit.  
- Конфигурационные хеши для контроллеров.  
- Описание ресурса в свободной форме.

**Annotations используют операторы и контроллеры**
Многие инструменты K8s читают annotations для своих целей:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"     # ← Prometheus сканирует эту аннотацию
    prometheus.io/port: "9090"
    nginx.ingress.kubernetes.io/rewrite-target: /
```

Это позволяет настраивать поведение операторов через манифесты ресурсов, не меняя сам оператор.