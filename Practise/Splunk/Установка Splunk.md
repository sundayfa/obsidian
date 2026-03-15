Чтобы запустить Splunk в контейнере нужно создать директорию, а в ней два файла:
compose-file:
```
version: '3.8'

services:
  splunk:
    image: splunk/splunk:latest
    container_name: splunk
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_GENERAL_TERMS=--accept-sgt-current-at-splunk-com
      - SPLUNK_PASSWORD=${SPLUNK_PASSWORD}
      - SPLUNK_ENABLE_LISTEN=9997
    ports:
      - "8000:8000"   # Web UI
      - "9997:9997"   # Indexer port (TCP)
    volumes:
      - splunk-data:/opt/splunk/var:Z
    networks:
      - splunknet

  forwarder:
    image: splunk/universalforwarder:latest
    container_name: splunk-uf
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_PASSWORD=${SPLUNK_PASSWORD}
      - SPLUNK_ADD=monitor /logs/app.log
      - SPLUNK_FORWARD_SERVER=splunk:9997
    volumes:
      - ./logs:/logs:Z
    depends_on:
      - splunk
    networks:
      - splunknet

volumes:
  splunk-data:

networks:
  splunknet:
    driver: bridge
```

.env:
```
SPLUNK_PASSWORD=<your_password> #min 8 symbols
```

Далее используя Docker Compose или Podman Compose запускаем контейнеры:
```
podman-compose up -d
```

Итого получаем работающий Splunk Enterprise и UFW в контейнерах. Для того, чтобы попасть внутрь контейнера:
```
podman exec -it <container_id> /bin/bash
```

В /etc/ лежат все конфигурационные файлы приложенек и тд.