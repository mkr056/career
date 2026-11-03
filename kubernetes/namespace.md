В кластере могут жить много окружений и команд. Если все ресурсы вперемешку, наступит хаос: dev-pod рядом с prod-pod'ом, имена пересекаются, кто-то случайно удалит чужое.

Решение — `namespaces`. Это логическое разделение ресурсов внутри одного кластера.

**Стандартные namespaces**
В свежем кластере уже есть несколько:
- `default` — где живёт всё что вы создали не указав namespace.
- `kube-system` — системные компоненты K8s (kube-proxy, coredns).
- `kube-public` — общедоступные ресурсы (редко используется).

**Зачем создавать свои**
Разделение окружений:  
- `dev` — для разработки.  
- `staging` — для тестов.  
- `production` — для продакшна.

В одном кластере. Не надо разворачивать три кластера — namespace дешевле.

Разделение команд:  
- `team-payments` — сервисы команды платежей.  
- `team-users` — сервисы команды пользователей.

Каждой команде можно дать права только на её namespace через RBAC.

**Создание**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

Или одной командой: `kubectl create namespace dev`.

**Использование**
Указываете namespace при создании ресурса:
```yaml
metadata:
  name: my-app
  namespace: dev
```

Или при команде: `kubectl apply -f deploy.yaml -n dev`.

**Лимиты ресурсов на namespace**
Можно ограничить сколько CPU/памяти все Pod'ы в namespace могут съесть в сумме:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
```

Команда не съест больше выделенного — даже если задеплоит миллион Pod'ов.

