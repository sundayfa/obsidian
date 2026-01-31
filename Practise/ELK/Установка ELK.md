**3.** Команды для установки elastic, которые я вводил в терминале ВМ (Ubuntu):  
    3.1 `wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg`  
    3.2 `sudo apt-get install apt-transport-https`  
    3.3 `echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://mirror.yandex.ru/mirrors/elastic/8/ stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list` _# доступ  из России заблокирован, пользуемся зеркалом с Yandex_  
    3.4 `sudo apt-get update && sudo apt-get install elasticsearch` _# здесь после установки в терминале отобразится ряд полезной информации, я рекомендую найти строку "The generated password for the built-in elastic superuser is: 674lXks81PXoTvml0JiX" и скопировать ключ он еще пригодится, а также команду для генерации токена доступа для Kibana:_ `/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana`  
    3.5 `sudo systemctl daemon-reload`  
    3.6 `sudo systemctl enable elasticsearch.service`  
    3.7 `sudo systemctl start elasticsearch.service`  
    3.8 `sudo systemctl status elasticsearch.service` _# проверяем статус запуска, пример результата на скрине ниже:_  
    3.9 проверим теперь, что elasticsearch действительно нормально работает, отправим c HOSTa из IDLE Python 3 (устанавливается автоматически вместе с Python 3) запрос на наш сервер (Ubuntu), пример результата на скрине ниже:
**4.** Команды для настройки elastic, которые я вводил в терминале ВМ (Ubuntu):  
    4.1 `sudo -s`  
    4.2 `nano /etc/elasticsearch/elasticsearch.yml` _# открываем конфиг, ищем следующие строчки и правим их:_   
        4.2.1 `network.host: 127.0.0.1` _# Elasticsearch будет доступен на локальном интерфейсе_  
        4.2.2 `discovery.seed_hosts: ["127.0.0.1", "[::1]"]` _# это нужно чтобы избежать "ошибки" при запуске службы_  
        4.2.3 `http.host: 0.0.0.0` _# это значит разрешить подключения по протоколу HTTP API из любого места_  
    4.3 сохраняем результаты правки "Ctrl + O" и выходим "Ctrl + X"  
    4.4 смотрим, что получилось следующей командой `ss -tulnp | grep 9200`, пример результата на скрине ниже:
**5.** Команды для установки kibana, которые я вводил в терминале ВМ (Ubuntu):  
    5.1 `sudo apt install kibana`  
    5.2 `sudo systemctl daemon-reload`  
    5.3 `sudo systemctl enable kibana.service`  
    5.4 `sudo systemctl start kibana.service`  
    5.5 `sudo systemctl status kibana.service`, пример результата на скрине ниже:
    5.6 `sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system` _# создадим пароль для встроенного пользователя kibana_system и сохраним себе он нам понадобится позже для аутентификации в elasticsearch_

**6.** Команды для настройки kibana, которые я вводил в терминале ВМ (Ubuntu):  
    6.1 `sudo cp -R /etc/elasticsearch/certs /etc/kibana` _# копируем папку с сертификатами из директории "elasticsearch" в директорию "kibana"_  
    6.2 `sudo chown -R root:kibana /etc/kibana/certs` _# передаем root:kibana права на папку "certs"_  
    6.3 `sudo -s`  
    6.4 `nano /etc/kibana/kibana.yml` _# открываем конфиг, ищем следующие строчки и правим их:_  
        6.4.1 `server.host: "192.168.1.203"` _# указываем IP-адрес нашей ВМ (Ubuntu)_  
        6.4.2 `elasticsearch.username: "kibana_system"`  
        6.4.3 `elasticsearch.password: "XXXXXXXXXXXXXXXXXXXX"` _# указываем пароль из п.п 5.6 (в нашем примере это - 674lXks81PXoTvml0JiX)_  
        6.4.4 `elasticsearch.ssl.certificateAuthorities: [ "/etc/kibana/certs/http_ca.crt" ]` _# добавляем сертификат_  
        6.4.5 `elasticsearch.hosts: [https://192.168.1.203:9200]` _# указываем подключение к elasticsearch по HTTPS_  
    6.5 сохраняем результаты правки "Ctrl + O" и выходим "Ctrl + X"

    6.6 смотрим, что получилось следующей командой `ss -tulnp | grep 5601`, пример результата на скрине ниже:  
    6.7 `sudo systemctl restart kibana.service` _# перегружаем Kibana_  
    6.8 вводим в браузере `http://192.168.1.203:5601` (лучше Edge, у меня Yandex не сработал сразу) и попадаем на приветственную страницу web-интерфеса Elasticsearch  
    6.9 вводим - "elastic"  
    6.10 вводим пароль, который сгенерировали в п.п 3.4 (в нашем примере это - 674lXks81PXoTvml0JiX)  
    6.11 далее у нас потребую токен, для этого в терминале ВМ (Ubuntu) вводим: `/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana`  и копируем значение токена из терминала в форму  web-интерфеса Elasticsearch  
    6.12 после токена потребуется Verification code, для этого в терминале ВМ (Ubuntu) вводим: `sudo sudo /usr/share/kibana/bin/kibana-verification-code` и копируем значение кода из терминала в форму web-интерфеса Elasticsearch.

Для установки Fleet Server необходимо перейти в Management -> Fleet -> нажать синюю кнопку `Add Fleet Server`, перейти в раздел Advanced. Если хост, который будет выполнять роль Fleet Server, тот же, где установлен ELK, то можно указать localhost, иначе адрес сервера, куда будем ставить Fleet Server. Далее генерируем токен. В пункте 5 указаны репозитории, которые недоступны в России, можно использовать [заркало](https://mirror.yandex.ru/mirrors/elastic/8/pool/main/e/elastic-agent/) и установить нужную версию. После чего выполнить команды:
```
sudo dpkg -i elastic-agent-UR_VERSION.deb 
sudo systemctl enable elastic-agent 
sudo systemctl start elastic-agent
```

И еще одна команда для enroll'а (тут такая же логика с localhost):
```
sudo elastic-agent enroll \ --fleet-server-es=https://<localhost_OR_IP>:9200 \ --fleet-server-service-token=<TOKEN_FROM_4_PUNCT> \ --fleet-server-policy=fleet-server-policy \ --fleet-server-port=8220 \ --fleet-server-es-insecure \ --insecure
```

После этого Fleet Server должен отобразиться во Fleet -> Agents.