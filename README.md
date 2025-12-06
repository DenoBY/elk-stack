# ELK Stack

## Установка на VPS

```bash
# Настроить систему для Elasticsearch
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

# Создать файл с паролями
cp .env.example .env
nano .env  # Установить свои пароли

# Запуск
docker-compose up -d

# Проверка
docker-compose ps
```

## Настройка паролей (после первого запуска)

```bash
# 1. Установить пароль для kibana_system пользователя
docker exec -it elasticsearch bin/elasticsearch-reset-password -u kibana_system -i

# Ввести пароль который указали в .env как KIBANA_PASSWORD

# 2. Перезапустить Kibana
docker-compose restart kibana
```

## Вход в Kibana

- URL: `http://IP_СЕРВЕРА:5601`
- Логин: `elastic`
- Пароль: значение `ELASTIC_PASSWORD` из `.env`

## Открытые порты

| Порт | Сервис   | Назначение               |
|------|----------|--------------------------|
| 5044 | Logstash | Приём логов от Filebeat  |
| 5601 | Kibana   | Веб-интерфейс            |

Kibana: `http://IP_СЕРВЕРА:5601`

## Установка Filebeat на серверах с логами

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.12.0-amd64.deb
sudo dpkg -i filebeat-8.12.0-amd64.deb
sudo nano /etc/filebeat/filebeat.yml
```

Содержимое filebeat.yml:

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
