С помощью политик Fleet мы можем управлять конфигурацией агентов elastic. Давайте настроим наши агенты на сбор системных логов. Перейдем в раздел Agent policies. Тут перечислены политики, которые применяются к агентам, например, какие логи стоит собирать. В разделе **"Add Integrations"** можно добавить в политику модули для сбора данных. 

Интеграции(Integrations) это модули для сбора данных из различных источников. С актуальным списком доступных интеграций можно ознакомиться по адресу [https://www.elastic.co/integrations](https://www.elastic.co/integrations)

Однако можно локально поднять Docker контейнер со всеми интеграциями, так как сайт недоступен:
```
 sudo docker pull docker.elastic.co/package-registry/distribution:8.15.0
```
Ожидаем пока что всё скачается и потом:
```
sudo docker run -d \
  --name elastic-package-registry \
  --restart always \
  -p 8080:8080 \
  docker.elastic.co/package-registry/distribution:8.15.0

```
Тут снова ожидаем пока что станет тяжелый образ. Команды для дебага:
```
 sudo docker ps - статус должен быть  Up About an $time$ (healthy)
 curl -I http://localhost:8080 - можно дернуть его курлом
```
Теперь нужно указать кибане, где ей искать интеграции:
```
sudo nano /etc/kibana/kibana.yml
```
В конец файла добавляем строчку:
```
xpack.fleet.registryUrl: "http://localhost:8080"
```
Далее перезапускаем:
```
 sudo systemctl restart kibana
```
Вуаля! Все интеграции появились!
