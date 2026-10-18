Весь пайплайн GitLab CI/CD описывается в одном файле — .gitlab-ci.yml. Он лежит в корне репозитория, и GitLab читает его автоматически при каждом пуше.

Базовая структура файла
```
stages:
  - build
  - test
  - deploy

build-app:
  stage: build
  script:
    - echo "Сборка приложения"
    - npm install
    - npm run build

run-tests:
  stage: test
  script:
    - echo "Запуск тестов"
    - npm test

deploy-prod:
  stage: deploy
  script:
    - echo "Деплой на сервер"
    - ./deploy.sh
```

Ключ stages задаёт глобальный порядок стадий. Каждый job ссылается на свою стадию через ключ stage. Если несколько jobs принадлежат одной стадии — они выполняются параллельно.

Script — список shell-команд внутри job.

Кроме stage и script, job поддерживает несколько важных ключей:
- image: Docker-образ, в котором запускается job
- before_script: Команды перед основным script
- after_script: Команды после script (выполняются даже при ошибке)
- variables: Переменные окружения для конкретного job
- tags: Выбор раннера по метке

Если в script указана команда, которой нет в образе или окружении раннера, job упадёт с ошибкой command not found. Всегда проверяйте, что нужные утилиты доступны в выбранном image или установлены в before_script.
