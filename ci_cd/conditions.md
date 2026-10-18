По умолчанию каждый job в GitLab CI запускается при каждом пуше в любую ветку. GitLab предоставляет два механизма: устаревший, но широко встречающийся only/except и современный rules.

only задаёт список условий, при которых job запускается. except — при которых job пропускается. Оба принимают имена веток, теги и специальные ключевые слова.

```
deploy_prod:
  stage: deploy
  script:
    - ./deploy.sh production
  only:
    - main
    - tags
```

Здесь job `deploy_prod` запустится только при пуше в ветку `main` или при создании git-тега.

`except` работает зеркально — исключает перечисленные условия.

Типичная ошибка: использовать `only` и `except` одновременно в одном job. GitLab применяет их через логическое И, что легко приводит к неожиданному поведению. Лучше выбирать что-то одно.

rules — замена only/except, появившаяся в GitLab 12.3. Она позволяет задавать несколько условий с разными действиями и поддерживает переменные окружения.

Каждое правило проверяется сверху вниз. Первое совпавшее определяет поведение job. Если ни одно не совпало — job не запускается.

```
build_image:
  stage: build
  script:
    - docker build -t myapp .
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: manual
    - when: never
```

Разбор примера:  
- Если пуш в `main` — job запускается автоматически.  
- Если pipeline запущен из merge request — job появляется в интерфейсе, но требует ручного подтверждения (`when: manual`).  
- Во всех остальных случаях — job не создаётся (`when: never`).

`rules` работает с предопределёнными переменными GitLab. Наиболее полезные:
- $CI_COMMIT_BRANCH: Имя текущей ветки
- $CI_COMMIT_TAG: Имя тега, если pipeline запущен по тегу
- $CI_PIPELINE_SOURCE: Источник: `push`, `merge_request_event`, `schedule`
- $CI_MERGE_REQUEST_TARGET_BRANCH_NAME: Целевая ветка merge request

Условие в `if` — это строковое сравнение или проверка на существование переменной. Если переменная не определена, она равна пустой строке.

```
notify_slack:
  stage: notify
  script:
    - ./send_notification.sh
  rules:
    - if: '$CI_COMMIT_TAG != null'
      when: always
```

Этот job отправит уведомление только при создании тега — удобно для оповещений о релизах.

