В современных K8s-кластерах админ обычно настраивает Pod Security Standards:

```yaml
# В namespace включается:
labels:
  pod-security.kubernetes.io/enforce: restricted
```

После этого K8s откажет запускать Pod если:
- `runAsNonRoot: false` (или не задано)
- Используются privileged-возможности
- Контейнер пытается смонтировать host paths

Ваш PodSpec должен явно сказать «работаю не от root»:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: my-app
```

И сам образ должен уметь запускаться не от root — для этого нужен `USER` в Dockerfile.