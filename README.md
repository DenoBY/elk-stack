# ELK Stack

## Первый запуск

```bash
# 1. Настроить систему для Elasticsearch
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

# 2. Создать .env с паролем
echo "ELASTIC_PASSWORD=ваш_сложный_пароль" > .env

# 3. Запустить только Elasticsearch
docker-compose up -d elasticsearch

# 4. Подождать ~1 минуту пока запустится, проверить:
docker-compose logs elasticsearch | tail -5
# Должно быть "license ... valid" без ошибок

# 5. Создать токен для Kibana
docker exec -it elasticsearch bin/elasticsearch-service-tokens create elastic/kibana kibana-token

# 6. Скопировать токен (строка после SERVICE_TOKEN =) и добавить в .env
echo "KIBANA_SERVICE_TOKEN=AAEAAWVsYXN0aWMva2liYW5hL..." >> .env

# 7. Запустить остальные сервисы
docker-compose up -d

# 8. Проверить статус (все должны быть Up)
docker-compose ps
```

## Вход в Kibana

- URL: `http://IP_СЕРВЕРА:5601`
- Логин: `elastic`
- Пароль: значение `ELASTIC_PASSWORD` из `.env`

## Порты

| Порт | Сервис   | Назначение               |
|------|----------|--------------------------|
| 5044 | Logstash | Приём логов от Filebeat  |
| 5601 | Kibana   | Веб-интерфейс            |

## Установка Filebeat (на серверах с логами)

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.0-amd64.deb
sudo dpkg -i filebeat-8.12.0-amd64.deb
```

Настроить `/etc/filebeat/filebeat.yml`:

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/*.log
      - /var/log/*.log

output.logstash:
  hosts: ["IP_VPS_С_ELK:5044"]
```

```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

## Автоудаление логов старше 30 дней

В Kibana → **Dev Tools** выполнить:

```
PUT _ilm/policy/delete-30-days
{
  "policy": {
    "phases": {
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

Создать шаблон для автоприменения политики к новым индексам:

```
PUT _index_template/printegy-template
{
  "index_patterns": ["printegy-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "delete-30-days"
    }
  }
}
```

Применить политику к существующим индексам:

```
PUT printegy-*/_settings
{
  "index.lifecycle.name": "delete-30-days"
}
```

Проверить что политика применилась:

```
GET printegy-*/_settings?filter_path=*.settings.index.lifecycle
```
