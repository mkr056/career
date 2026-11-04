Мы обращались к Service по имени: `database:5432`, `web:80`. Откуда K8s знает что `database` — это конкретный IP? Через встроенный DNS-сервер — CoreDNS.

CoreDNS — это сам по себе Pod (живёт в `kube-system`). Когда вы создаёте Service, K8s автоматически регистрирует для него DNS-запись.

```
Service "database" с ClusterIP 10.96.0.5
        ↓
CoreDNS запоминает: database → 10.96.0.5
```

Теперь любой Pod, обращаясь к `database`, спросит CoreDNS «какой IP у database?», получит `10.96.0.5`, подключится.

Ничего настраивать не надо: создал Service — DNS-запись появилась автоматически. Удалил Service — запись пропала. Это работает из коробки.

```yaml
kind: Service
metadata:
  name: my-api    # ← это имя сразу становится DNS-именем
```

Не нужно знать IP, не нужно его прописывать. Имя стабильно даже когда Pod'ы за Service пересоздаются.

Теперь полная картина:
1. Pod'ы эфемерные, IP меняются.
2. Service даёт стабильный ClusterIP.
3. CoreDNS даёт стабильное имя для этого ClusterIP.

Вы работаете с именами (`database`, `redis`, `api`), а вся машинерия с IP скрыта.

Проверить можно изнутри Pod'а:
```bash
kubectl exec -it some-pod -- nslookup database
# Server:    10.96.0.10
# Address:   10.96.0.10:53
# Name:      database.default.svc.cluster.local
# Address:   10.96.0.5
```

Видите — `database` резолвится в IP Service.