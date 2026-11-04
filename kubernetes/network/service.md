`Service` — это стабильная «точка входа» к группе Pod'ов. У него постоянный IP и постоянное DNS-имя, которые не меняются пока Service существует.

```
API → Service "database" (стабильный) → Pod БД (любой, меняется)
```

Service сам следит за тем какие Pod'ы сейчас живы (по их меткам) и направляет трафик на здоровые. Pod упал, появился новый — Service автоматически перенаправит. Вам ничего настраивать не надо.

Service находит Pod'ы через labels и selectors.

```yaml
kind: Service
spec:
  selector:
    app: database     # ← найди все Pod'ы с этой меткой
```

Service постоянно держит актуальный список Pod'ов с `app: database`. Любой запрос к Service балансируется на один из них.

# Типы Service
Service бывает нескольких типов. Самый базовый и частый — `ClusterIP`. Он создаётся по умолчанию если тип не указан.

**ClusterIP**
`ClusterIP` даёт Service IP-адрес доступный только внутри кластера. Снаружи (из интернета, с вашего ноутбука) к нему не достучаться. Только Pod'ы внутри кластера могут обращаться.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: ClusterIP        # можно не писать — это default
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
```

Теперь любой Pod внутри кластера может обратиться к `database:5432` — и попасть на один из Pod'ов БД.

Когда использовать: для внутренней связи между сервисами. Всё что не должно торчать наружу. А наружу обычно торчит только один-два сервиса (например frontend), остальное — внутреннее.

Принцип безопасности: минимум публичных точек входа. Ваша база данных не должна быть доступна из интернета — никогда. Только ваш API должен с ней общаться, а API внутри кластера.

Поэтому БД, кэши, внутренние сервисы — всегда `ClusterIP`. Они невидимы снаружи, защищены самим фактом изоляции.

Обратиться к сервису изнутри кластера — по имени Service:

```
postgresql://database:5432/mydb
redis://cache:6379
http://backend-api:8080
```

K8s через DNS превратит имя в ClusterIP, ClusterIP направит на живой Pod.

**NodePort**
ClusterIP невидим снаружи. Но иногда нужен внешний доступ и для этого есть `NodePort`.

`NodePort` открывает порт на каждой ноде кластера. Запрос на `<IP-любой-ноды>:<этот-порт>` попадёт в Service, а оттуда — на Pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080      # порт на нодах (диапазон 30000-32767)
```

Теперь `http://<IP-ноды>:30080` открывает ваш сервис снаружи.

Порт можно выбрать только из диапазона 30000-32767. Это сделано чтобы не конфликтовать с системными портами. Если не указать `nodePort` — K8s выберет случайный из этого диапазона.

NodePort удобен для теста, но плох для production:
- Некрасивые порты. `myapp.com:30080` вместо `myapp.com`. Пользователи привыкли к 80/443.
- Нужно знать IP ноды: если нода умерла, надо вручную переключаться на другую.
- Нет балансировки между нодами: запрос идёт на конкретную ноду.

Для production вместо `NodePort` используют `LoadBalancer` (в облаке) или `Ingress`.

# `port` и `targetPort`: две стороны `Service`
В манифесте Service вы часто видите два поля про порты: `port` и `targetPort`.

```yaml
spec:
  selector:
    app: web
  ports:
  - port: 80           # ← порт Service
    targetPort: 8080   # ← порт контейнера
```

`targetPort` — это порт на котором слушает ваше приложение внутри контейнера. Если Go-сервер слушает на 8080 — `targetPort: 8080`.

`port` — это порт самого Service. По нему обращаются другие Pod'ы. Может быть любым.

`Другой Pod → Service:80 → перенаправление → Pod:8080`

1. Клиент стучится в `web:80` (port Service).
2. Service перенаправляет на `targetPort: 8080` одного из Pod'ов.
3. Ваше приложение на 8080 получает запрос.

**Зачем разделять**
Часто их делают одинаковыми (`port: 8080, targetPort: 8080`) — это нормально. Но разделение даёт гибкость:  
- Снаружи все обращаются к привычному `:80`, а внутри приложение может слушать `:8080` (без root-прав).  
- Можете поменять порт приложения, не меняя как к нему обращаются.

Удобный приём — назвать порт в Pod и ссылаться по имени:

```yaml
# В Deployment:
ports:
- name: http
  containerPort: 8080

# В Service:
ports:
- port: 80
  targetPort: http     # ← по имени, не числу
```

Тогда меняете порт в одном месте (Deployment) — Service подхватит.

Если Service типа NodePort, появляется третий порт:

```yaml
ports:
- port: 80           # порт Service
  targetPort: 8080   # порт контейнера
  nodePort: 30080    # порт на ноде
```

# DNS-имена
Когда вы обращаетесь к Service в том же namespace — достаточно короткого имени `database`. Но если Service в другом namespace — нужно полное имя.

Полный вариант: `database.production.svc.cluster.local`.

Это значит: «Service `database` в namespace `production`».

- `database` — имя Service.
- `production` — namespace где он живёт.
- `svc` — означает «это Service» (а не Pod).
- `cluster.local` — суффикс кластера (можно настроить, но обычно такой).

K8s позволяет сокращать в зависимости от того откуда вы обращаетесь:

Из того же namespace:

`database                    # просто имя`

Из другого namespace:

```
database.production         # имя + namespace
database.production.svc     # + svc
database.production.svc.cluster.local   # полное
```

Все варианты выше работают одинаково. Чем больше частей — тем явнее.

Зачем знать полное имя:
1. Cross-namespace связь: API в `frontend`, БД в `data` — нужно полное имя.
2. Явность в production-конфигах: лучше написать `database.production` чем полагаться на «текущий namespace».
3. Отладка: понимаете что значит длинное имя в логах и ошибках DNS.

Кстати, длинный суффикс `.svc.cluster.local` вы будете часто видеть в логах ошибок подключения. Теперь, увидев его, сразу поймёте: кто-то не смог достучаться до Service по полному имени — и сможете проверить namespace.

**LoadBalancer**
В production нужен красивый внешний доступ: `myapp.com` на порту 443, с балансировкой и отказоустойчивостью. Для этого есть тип `LoadBalancer`.

Когда вы создаёте Service типа `LoadBalancer` в облаке (AWS, GCP, Yandex Cloud), облако автоматически создаёт настоящий балансировщик (AWS ELB, GCP Load Balancer) и привязывает к нему внешний IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

Через минуту:
```bash
kubectl get svc web
# NAME   TYPE           EXTERNAL-IP      PORT(S)
# web    LoadBalancer   34.120.45.67     80:31234/TCP
```

`EXTERNAL-IP` — реальный публичный IP. Привязываете к нему DNS (`myapp.com → 34.120.45.67`) и всё работает.

Что облако делает за вас:
- Создаёт балансировщик нагрузки.
- Раздаёт трафик между всеми нодами.
- Если нода умерла — перестаёт слать на неё.
- Иногда — TLS termination, DDoS-защиту, health checks.

Каждый LoadBalancer Service = отдельный облачный балансировщик = отдельный счёт (несколько долларов в месяц минимум).

Если у вас 20 сервисов которым нужен внешний доступ — это 20 балансировщиков, 20 внешних IP, 20 счетов. Дорого и расточительно.

Вместо 20 LoadBalancer'ов делают один LoadBalancer + Ingress, который по правилам направляет:  
- `myapp.com` → сервис A  
- `api.myapp.com` → сервис B  
- `myapp.com/admin` → сервис C

Один балансировщик, один IP, много сервисов.

**Ingress**
Мы выяснили: LoadBalancer Service удобен, но каждый стоит денег. Если у вас 20 сервисов которым нужен внешний HTTPS — 20 балансировщиков, дорого.

Решение — `Ingress`. Это «умный reverse proxy» на входе в кластер. Один публичный IP, а внутри он раздаёт запросы по правилам.

Ingress в K8s — это две вещи:
1. Ingress Controller — программа (Pod) которая реально обрабатывает трафик. Это nginx, traefik, или подобное. Её надо установить в кластер.
2. Ingress Resource — YAML с правилами «какой запрос куда направить». Их вы пишете сами.

Ingres Controller: один вход для всего кластера. K8s не включает Ingress Controller по умолчанию. Вы выбираете какой поставить:  
- nginx-ingress — самый популярный, проверенный.  
- traefik — современный, удобная автонастройка TLS.  
- Облачные: AWS ALB Controller, GCP Ingress.

Установка обычно через Helm: `helm install ingress-nginx ingress-nginx/ingress-nginx`.

После установки Controller получает один LoadBalancer (один публичный IP). Весь внешний HTTP/HTTPS-трафик идёт через него.

Как работает в целом:
```
Интернет
   ↓
Один LoadBalancer (один IP)
   ↓
Ingress Controller (nginx)
   ↓ (по правилам Ingress Resource)
   ├─ myapp.com       → web-service
   ├─ api.myapp.com   → api-service
   └─ myapp.com/admin → admin-service
```

Один балансировщик, один IP — десятки сервисов за ним.

Ingress Resource: правила маршрутизации. Controller установлен. Теперь вы пишете правила — какой запрос куда направить. Это и есть Ingress Resource (YAML).

Минимальный пример:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

Читается так: «запросы на `myapp.com` с путём `/` направляй в Service `web-service` на порт 80».

Несколько правил:
```yaml
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port: { number: 80 }
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port: { number: 8080 }
```

Теперь:  
- `myapp.com/` → web-service  
- `myapp.com/api` → api-service

Один домен, разные пути — разные сервисы.

pathType:
- `Prefix` — совпадение по началу пути. `/api` ловит `/api`, `/api/users`, `/api/orders`.
- `Exact` — точное совпадение. `/api` ловит только `/api`.

Обычно используют `Prefix`.

Важная деталь: в `backend` вы указываете Service, не Pod. Ingress → Service → Pod. Service делает балансировку как обычно. Ingress только маршрутизирует по host/path.

`Ingress (маршрут по URL) → Service (балансировка) → Pod (обработка)`

Применение:
```bash
kubectl apply -f ingress.yaml
kubectl get ingress
# NAME         HOSTS        ADDRESS         PORTS
# my-ingress   myapp.com    34.120.45.67    80
```

Привязываете DNS `myapp.com → 34.120.45.67` и всё работает.

Ingress может разделять трафик двумя способами: по домену (host) или по пути (path). Часто комбинируют.

Path-based: один домен, разные пути:
```yaml
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        backend: { service: { name: frontend, port: { number: 80 }}}
      - path: /api
        backend: { service: { name: api, port: { number: 8080 }}}
      - path: /admin
        backend: { service: { name: admin, port: { number: 80 }}}
```

Один домен `myapp.com`:  
- `/` → frontend  
- `/api` → api  
- `/admin` → admin

Когда: монолитный домен, логически разные части (UI, API, админка).

Host-based: разные поддомены:
```yaml
spec:
  rules:
  - host: app.mycompany.com
    http:
      paths:
      - path: /
        backend: { service: { name: app, port: { number: 80 }}}
  - host: api.mycompany.com
    http:
      paths:
      - path: /
        backend: { service: { name: api, port: { number: 8080 }}}
  - host: grafana.mycompany.com
    http:
      paths:
      - path: /
        backend: { service: { name: grafana, port: { number: 3000 }}}
```

Разные поддомены:  
- `app.mycompany.com` → app  
- `api.mycompany.com` → api  
- `grafana.mycompany.com` → grafana

Когда: разные сервисы/продукты, у каждого свой поддомен.

Можно вместе:
```
api.mycompany.com/v1   → api-v1
api.mycompany.com/v2   → api-v2
app.mycompany.com/     → frontend
```

Для host-based нужно настроить DNS — все поддомены указывают на IP Ingress:
```
app.mycompany.com.     A   34.120.45.67
api.mycompany.com.     A   34.120.45.67
grafana.mycompany.com. A   34.120.45.67
```

Или wildcard:
```
*.mycompany.com.       A   34.120.45.67
```

Все поддомены → один IP Ingress, а он разрулит.

В production сайт должен работать по HTTPS. Это значит — нужен TLS-сертификат. Получать и обновлять его вручную — боль (сертификаты Let's Encrypt живут 90 дней). В K8s это автоматизируют через cert-manager.

«Termination» = расшифровка TLS происходит на Ingress. Снаружи — зашифрованный HTTPS, внутри кластера — обычный HTTP.

```
Браузер --HTTPS--> Ingress (расшифровал) --HTTP--> Service --> Pod
```

Вашему приложению не надо возиться с сертификатами — Ingress всё делает. Приложение просто слушает HTTP.

cert-manager — это оператор который:  
1. Автоматически получает сертификаты от Let's Encrypt (бесплатно).  
2. Автоматически обновляет их до истечения.  
3. Хранит как K8s Secret.

Установка: `helm install cert-manager jetstack/cert-manager --set installCRDs=true`

Настраиваете ClusterIssuer (источник сертификатов):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

Теперь в Ingress добавляете TLS-секцию и аннотацию:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt   # ← cert-manager увидит
spec:
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-tls         # ← сюда положит сертификат
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: { name: web, port: { number: 80 }}
```

Что произойдёт:  
1. cert-manager видит аннотацию.  
2. Запрашивает сертификат у Let's Encrypt для `myapp.com`.  
3. Кладёт в Secret `myapp-tls`.  
4. Ingress использует этот сертификат для HTTPS.  
5. Каждые 60 дней cert-manager обновляет.

Полностью автоматически. Один раз настроил — забыл.


**Gateway API: замена Ingress в новых версиях**
Ingress существует давно и работает. Но у него есть проблемы, и K8s-сообщество разработало замену — `Gateway API`. Вам полезно знать про неё, потому что новые проекты постепенно переходят.

Проблемы `Ingress`:
1. Мало возможностей в стандарте. Базовый Ingress умеет только host/path → service. Всё остальное (редиректы, заголовки, веса для canary) — через **аннотации**, которые у каждого Controller'а свои. Получается непортируемо: конфиг для nginx-ingress не работает с traefik.
2. Один объект на всё. Ingress смешивает заботы: и инфраструктуру (где TLS), и маршрутизацию (какой path куда). В больших командах это создаёт конфликты — кто владеет Ingress-объектом.

Gateway API предлагает разделение на роли:
- GatewayClass — тип шлюза (настраивает админ кластера).
- Gateway — точка входа, порты, TLS (настраивает платформенная команда).
- HTTPRoute — правила маршрутизации (настраивают команды приложений).

```yaml
# Платформенная команда:
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    port: 443
    protocol: HTTPS

---
# Команда приложения:
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
spec:
  parentRefs:
  - name: prod-gateway
  hostnames:
  - myapp.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 8080
```

Преимущества:
- Стандартизировано: канареечные деплои, заголовки, редиректы — в стандарте, не в аннотациях. Портируемо между Controller'ами.
- Разделение ролей: админ владеет Gateway, разработчик — HTTPRoute. Меньше конфликтов.
- Более выразительно: traffic splitting (90% v1, 10% v2), фильтры заголовков из коробки.