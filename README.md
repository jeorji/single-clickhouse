# single-clickhouse

```bash
helm upgrade --install clickhouse ./single-clickhouse \
  -f single-clickhouse/values.yaml \
  -f single-clickhouse/values-secret.yaml
```

Минимальный Helm-чарт, который разворачивает одиночный ClickHouse:

- `StatefulSet` с 1 репликой, PVC для данных и `users.xml`/`profiles.xml` монтированными в `/etc/clickhouse-server/users.d`.
- Пользователи описываются (профили/квоты/сети) в `values.yaml`, а пароли выносятся в отдельный файл (`values-secret.yaml`).

## Требования к кластеру

- Kubernetes >=1.20 с рабочим StorageClass по умолчанию.
- Доступ к образу `clickhouse/clickhouse-server:<tag>` из кластера.

## Настройка пользователей и профилей

1. В `values.yaml` правим блок `users`: имена, профили, квоты, сети.
2. Копируем пример секретов:
   ```bash
   cp clickhouse/values-secret.example.yaml clickhouse/values-secret.yaml
   ```
   и вписываем реальные пароли (`userCredentials.<user>`). Этот файл не коммитим.
3. (Опционально) включаем пользовательские профили:
   ```yaml
   profiles:
     enabled: true
     items:
       - name: analytics
         settings:
           max_memory_usage: 4000000000
           max_threads: 8
   ```
   После этого пользователю можно назначить `profile: analytics`.

## Проверка

1. Убедиться, что Pod поднялся: `kubectl get pods -l app=clickhouse`.
2. Пробросить порт и выполнить HTTP-запрос:
   ```bash
   kubectl port-forward svc/clickhouse 8123:8123 &
   curl -u app:app_password --data-binary 'SELECT 1' http://localhost:8123/
   ```
3. Проверить TCP-клиентом внутри кластера:
   ```bash
   kubectl run -it --rm ch-client --image=clickhouse/clickhouse-server --restart=Never -- \
     clickhouse-client --host clickhouse --user app --password 'app_password' --query 'SELECT version();'
   ```
