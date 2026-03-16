```
apt install auditd
```

![](https://ucarecdn.stepik.net/1162b6bf-e211-45c3-a720-f2dfb026f7d8/)

2. Подготовим или скачаем готовый файл конфигурации:

```
# wget https://raw.githubusercontent.com/Neo23x0/auditd/master/audit.rules
```

3. Скопируем файл с правилами в каталог `/etc/audit/rules.d`:

```
# mv audit.rules /etc/audit/rules.d/audit.rules
```

![](https://ucarecdn.stepik.net/90d660c7-426a-4368-a1ee-78c9280183d6/)

4. Загрузим правила командой:

```
augenrules --load
```

5. Убедимся, что правила добавились с помощью команды: 

```
auditctl -l
```

![](https://ucarecdn.stepik.net/5a9fa5ae-320f-4051-b4f6-7760cd325931/)

6. И проверим, что демон auditd записывает события:

```
tail /var/log/audit/audit.log
```