На Windows скачали установочные файлы Sysmon с офф сайта Microsoft - https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

Далее нам нужен конфиг, я взял прям с сайта Wazuh - https://wazuh.com/resources/blog/emulation-of-attack-techniques-and-detection-with-wazuh/sysmonconfig.xml

Для установки Sysmon перейти в директорию, в которую была распакована зипка, после чего ввести команду, которая применяет пользовательский конфиг (в данном варианте файл конфига в той же директории, что и распакованная зипка):
```
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
```

Логи Sysmon хранятся в `Applications and Services Logs > Microsoft > Windows > Sysmon > Operational`

Теперь необходимо забирать логи агентом SIEM. Для этого нужно отредактировать конфиг агента или загружать уже готовый отредактированный агент на устройства. Путь до конфига агента:
```
C:\Program Files (x86)\ossec-agent\ossec.conf
```
далее с помощью любого редактора добавляем в конец файла (до *</ossec_config>*) источник логов:
```
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Перезапуск агента:
```
Restart-Service WazuhSvc
```

Не всё логируется Sysmon'ом. Конфиг определяет, какие процессы и события будут записываться в журнал. Для того, чтобы добавить (include) логирование события в тот или иной event_id, нужно изменить конфиг, например:
```
<ProcessCreate onmatch="include">
  <!-- New dedicated rule for Cipher -->
  <Rule name="DataWiping_Tools" groupRelation="or">
    <Image condition="end with">cipher.exe</Image>
  </Rule>
</ProcessCreate>
```

Данная строчка включает логирование события создания процесса (event_id=1), если поле image кончается словами `cipher.exe`. 

Кроме того, можно исключать (exclude) события, которые создают шум:
```
<ProcessCreate onmatch="exclude">
  <Rule groupRelation="and">
    <Image condition="end with">AcroRd32.exe</Image>
    <CommandLine condition="contains any">/CR;channel=</CommandLine>
  </Rule>
</ProcessCreate>
```

Проще всего смотреть в Event Viewer на событие (Details -> XML View) и искать, в каком поле будет значение, которое хотим отловить.