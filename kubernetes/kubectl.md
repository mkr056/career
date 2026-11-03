kubectl — это ваш главный интерфейс к Kubernetes.

# get — посмотреть список
```bash
kubectl get pods                       # все Pod'ы в текущем namespace
kubectl get pods -n production         # в конкретном namespace
kubectl get pods -A                    # во всех namespace
kubectl get pods -o wide               # с дополнительной инфой (IP, нода)
kubectl get pods -o yaml               # полный YAML ресурса
kubectl get pods -o json               # полный JSON ресурса
kubectl get pods --show-labels         # колонка с labels
kubectl get deployments,services       # несколько типов сразу
```

Универсальная команда «покажи мне ресурсы».

# describe — посмотреть детали
`kubectl describe pod my-pod`

Подробный вывод про один ресурс: события, контейнеры, volumes, что произошло за время жизни. Главный инструмент диагностики: «почему Pod не запускается?» → `describe`, читаете events внизу.

# logs — логи контейнера
```bash
kubectl logs my-pod                    # логи последнего запуска
kubectl logs my-pod --previous         # логи предыдущего запуска (если упал)
kubectl logs -f my-pod                 # follow (как tail -f)
kubectl logs my-pod -c sidecar         # конкретный контейнер если их несколько
```

Никакого «зайти в Pod и посмотреть» — логи централизованы.

# exec — выполнить команду в Pod
```bash
kubectl exec my-pod -- ls /app         # выполнить ls внутри
kubectl exec -it my-pod -- /bin/sh     # интерактивный shell
```

Войти и потыкать — для отладки.

# port-forward — пробросить порт локально
`kubectl port-forward pod/my-pod 8080:80`

Порт 80 в Pod'е теперь доступен на `localhost:8080` вашего компьютера. Можно открыть в браузере или приконнектиться psql'ом — даже если Pod внутри кластера без публичного доступа.

# apply — применить YAML
```bash
kubectl apply -f manifest.yaml
kubectl apply -f ./manifests/         # все файлы в папке
```

Декларативный подход: «вот желаемое состояние, реализуй». Если ресурса нет — создаст. Если есть — обновит.

# delete — удалить
```bash
kubectl delete pod my-pod
kubectl delete -f manifest.yaml       # удалить всё что описано в файле
```

# Контексты: переключение между кластерами
В реальной жизни у вас обычно несколько кластеров:
- Локальный minikube для экспериментов.
- Dev-кластер компании.
- Staging.
- Production.

kubectl надо знать в какой кластер сейчас обращаться. Это решается через контексты.

Контекст — это «набор настроек: кластер + пользователь + namespace». Все они хранятся в файле `~/.kube/config`.

```yaml
contexts:
- name: minikube
  context:
    cluster: minikube
    user: minikube
- name: production
  context:
    cluster: prod-cluster
    user: alice
    namespace: backend
```

Когда вы пишете `kubectl get pods`, kubectl смотрит **текущий контекст** и шлёт запрос в этот кластер.

**Просмотр**
```bash
kubectl config get-contexts            # все контексты
kubectl config current-context         # какой сейчас активен
```

Звёздочка слева — текущий контекст.

**Переключение**
`kubectl config use-context production`: Теперь все команды будут идти в production.

**Утилиты**
Голый `kubectl config use-context` — длинно. Большинство ставит утилиту [`kubectx`](https://github.com/ahmetb/kubectx):

```bash
kubectx                  # список контекстов
kubectx production       # переключить
kubectx -                # вернуться к предыдущему
```

И ещё `kubens` от того же автора:
```bash
kubens                   # список namespace
kubens dev               # сменить default namespace в текущем контексте
```

**Безопасность**
Когда у вас production-контекст активен, легко случайно выполнить опасную команду. Распространённая практика — настроить bash/zsh prompt чтобы было видно текущий контекст:

```
~/projects/my-app [production] $ kubectl delete deployment/payment-service
```

Видите красную надпись `production` → задумаетесь прежде чем нажать Enter. Это спасло многих от инцидентов.

# apply vs create
В kubectl есть два способа создавать ресурсы. Они работают по-разному.

`kubectl create` — императивный.

`kubectl create -f pod.yaml`: Говорит: «создать этот ресурс». Если ресурса нет — создаст. Если уже есть — ошибка: «AlreadyExists».

Можно создать ресурс прямо в командной строке без YAML: `kubectl create deployment my-web --image=nginx:1.25`.

`kubectl apply` — декларативный. 

`kubectl apply -f pod.yaml`: Говорит: «привести кластер к этому состоянию». Если ресурса нет — создаст. Если есть — обновит (применит только различия).

Изменил YAML, поменял `replicas: 3` → `replicas: 5`, `kubectl apply` — кластер обновит число реплик. Никакой ошибки.

**Разница**
Императивно (`create`, `delete`, `set`, `edit`):  
- Вы говорите «сделай X».  
- Шаг-за-шагом.  
- Кластер не знает «как должно быть» — только «что я делал».

Декларативно (`apply`):  
- Вы описываете желаемое состояние в YAML.  
- Кластер сам разбирается «как привести реальность к описанию».  
- YAML можно хранить в git. История изменений = история коммитов.


**Выбор**
В production — всегда `apply`. Причины:  
1. YAML коммитится в git → понятно кто и что менял.  
2. Идемпотентно — применил один раз, применил десять раз, результат одинаковый.  
3. Diff'ы между версиями понятны.  
4. GitOps-инструменты (ArgoCD, FluxCD) основаны на `apply`.

Императивные команды (`create`, `set image`) — для быстрых правок локально, в обучении, в одноразовых задачах. Не для prod.

