Вы не можете развернуть полноценный кластер на работе для экспериментов. Нужно где-то учиться. Есть несколько вариантов.

**minikube — самый популярный для обучения**
```bash
brew install minikube      # mac
# или
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

minikube start             # запускает однонодовый кластер в Docker или VM
kubectl get nodes          # видишь свой кластер
```

minikube запускает Linux-VM (или Docker-контейнер) на вашем компьютере и поднимает там полноценный Kubernetes. Все компоненты — настоящие.

Плюс: ближе всего к реальному K8s, есть addons (ingress, metrics).
Минус: тяжеловат (4 ГБ RAM хочет).

**kind — Kubernetes in Docker**
```bash
brew install kind
kind create cluster
```

То же самое что minikube, но кластер живёт прямо в Docker-контейнерах (без отдельной VM). Легче, быстрее.

Подходит для CI: можно стартовать кластер за 30 секунд, прогнать тесты, удалить.

**k3s — лёгкий K8s**
Полноценный K8s, но облегчённый — занимает 50-100 МБ. Подходит для:  
- IoT / edge (запустить на Raspberry Pi).  
- Маленьких production-кластеров.  
- Когда полный K8s избыточен.

**Облачные кластеры (для production)**
Когда дойдёте до production, будете использовать managed-кластер:
- EKS — Amazon.
- GKE — Google.
- AKS — Microsoft Azure.
- Yandex Managed Kubernetes — Yandex Cloud.

Облако само управляет Control Plane (etcd, API Server и т.д.), вы только настраиваете свои Worker Nodes и Pod'ы. Удобно.