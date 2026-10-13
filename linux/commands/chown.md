chown (change owner) меняет владельца и группу:

```
chown alice deploy.sh            # сменить владельца
chown alice:devops deploy.sh     # сменить владельца и группу
chown -R alice:devops /var/app   # рекурсивно для каталога
```

Флаг `-R` применяет изменения ко всем файлам и подкаталогам.