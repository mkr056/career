Триггер определяет, при каком событии система CI/CD должна начать выполнение цепочки этапов.

**Push**
Pipeline запускается каждый раз, когда разработчик отправляет коммит в репозиторий. Пример конфигурации в GitLab CI:

```
stages:
  - build
  - test

build-job:
  stage: build
  script:
    - echo "Building..."
  rules:
    - if: '$CI_PIPELINE_SOURCE == "push"'
```

На практике push-триггер ограничивают: например, только для веток `main`, `develop` или по маске `feature/*`.

**Merge Request**
Триггер по merge request (merge request trigger, в GitHub — pull request trigger) срабатывает, когда разработчик создаёт или обновляет запрос на слияние веток.

**Schedule**
Расписание (scheduled trigger, cron trigger) запускает pipeline по времени, независимо от активности разработчиков. Расписание задаётся в формате cron.

**Manual**
Ручной триггер требует явного действия человека — нажатия кнопки в интерфейсе или вызова через API. Pipeline не запустится автоматически, пока кто-то не даст разрешение. Это незаменимо для деплоя в продакшен.

```
deploy-production:
  stage: deploy
  script:
    - ./deploy.sh production
  when: manual
  only:
    - main
```

Ключевое слово `when: manual` говорит системе: не запускать этот шаг автоматически. В интерфейсе GitLab рядом с таким шагом появится кнопка «Play», которую нужно нажать вручную.