Статический анализ — это проверка кода без его выполнения. Его запускают до тестов. Статический анализ работает за секунды и даёт быструю обратную связь: разработчик узнаёт о проблеме раньше, чем тяжёлый тестовый прогон займёт несколько минут.

Пример: запуск в GitHub Actions
Допустим, проект на Python. Шаг линтинга в workflow выглядит так:
```
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - name: Install flake8
        run: pip install flake8
      - name: Run flake8
        run: flake8 src/ --max-line-length=88 --statistics
```

Если в коде есть строка длиннее 88 символов или неиспользуемый импорт, шаг завершится с ненулевым кодом выхода, и пайплайн остановится. Вывод будет примерно таким:
```
src/app.py:14:1: F401 'os' imported but unused
src/utils.py:32:89: E501 line too long (102 > 88 characters)
2 errors found
```