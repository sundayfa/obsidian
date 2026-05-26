В рамках настройки стенда был создан домен lab.local:
- Контроллер домена с ОС Windows Server 2022
- Workstation с ОС Windows 11, которая находится в домене
- Межсетевой экран pfsense
- Сервер со Splunk на машине с ОС Ubuntu Server
- Машина с ОС Kali Linux вне домена

Сетевая конфигурация:
- Внутренняя сеть в Vbox с адресом сети 192.168.100.0/24
- DC01 - 192.168.100.10
- WS01 - 192.168.100.20
- pfsense - 192.168.100.1 и NAT на втором интерфейсе
- Ubuntu Server - 192.168.100.45
- Kali Linux - 192.168.100.67

Для демонстрирования атак были созданы следующие УЗ:
- j.smith
- a.lee
- t.user
- svc_webapp
- b.wilson c настройкой DoesNotRequirePreAuth
- serj

![[Pasted image 20260526203248.png]]


#### Атаки на УЗ в домене для получения пароля

Была скачана утилита kerbrute и [файл](https://github.com/danielmiessler/SecLists/blob/master/Usernames/xato-net-10-million-usernames.txt) со списком имен УЗ для enum'а:

![[Pasted image 20260526210900.png]]

Далее можно сделать спреинг паролей для найденных УЗ:

![[Pasted image 20260526211338.png]]

Или брутфорс пароля для УЗ с помощью [словаря](https://github.com/danielmiessler/SecLists/blob/master/Passwords/Common-Credentials/xato-net-10-million-passwords-1000000.txt):
![[Pasted image 20260526212648.png]]

![[Pasted image 20260526212801.png]]

По итогу получили пароли для 3-х УЗ в домене

#### Использованные атаки и инструменты

| Атака            | Инструмент | Протокол        | Цель                            |
| ---------------- | ---------- | --------------- | ------------------------------- |
| User Enumeration | kerbrute   | Kerberos AS-REQ | Получить список УЗ              |
| Password Spray   | kerbrute   | Kerberos        | Подбор одного пароля ко всем УЗ |
| Brute Force      | kerbrute   | Kerberos        | Перебор паролей одной УЗ        |
#### Обнаружение атак в Splunk
#### User enum:
```
index="wineventlog" EventCode=4768 AND NOT Client_Address  IN (::1, ::ff*)
| stats count(Account_Name) BY Client_Address Account_Name
```

![[Pasted image 20260526213925.png|601]]

Идея в том, чтобы искать большое кол-во событий 4768 (TGT Requested) от одного ip/hostname. Однако такой метод требует тестирования в каждом отдельно взятом домене, так как кол-во событий может отличаться. Итоговая задача - выставить нужный порог кол-ва событий за определенное время.

#### Password spray:
```
index="wineventlog" EventCode=4771 NOT Client_Address IN (::*)
| bin _time span=5m
| stats dc(Account_Name) as unique_users, count BY _time, Client_Address
```

![[Pasted image 20260526214227.png]]

Идея в том, чтобы искать большое кол-во событий 4771 (failed Kerberos pre-authentication attempt) от одного ip/hostname. Однако такой метод требует тестирования в каждом отдельно взятом домене, так как кол-во событий может отличаться. Итоговая задача - выставить нужный порог кол-ва событий за определенное время.

#### Bruteforce:
```
index="wineventlog" EventCode=4771 OR EventCode=4625 NOT Client_Address IN (::*)
| bin _time span=1m
| stats count BY _time, Client_Address Account_Name
```

![[Pasted image 20260526214937.png]]

Идея в том, чтобы искать превышения порога событий 4771 или 4625 (logon request fails) для каждого ip/hostname. В примере я брутил УЗ `Administrator` и пароль подобрался далеко не сразу, отсюда и столько событий в минуту.

#### Усложнение атак
Политика блокировки аккаунтов:

![[Pasted image 20260526215525.png]]

Более строгая политика паролей для администраторов:

```
New-ADFineGrainedPasswordPolicy `
  -Name "AdminPasswordPolicy" `
  -Precedence 1 `
  -MinPasswordLength 16 `
  -ComplexityEnabled $true `
  -LockoutThreshold 3 `
  -LockoutDuration "00:60:00" `
  -LockoutObservationWindow "00:30:00" `
  -PasswordHistoryCount 24 `
  -MaxPasswordAge "30.00:00:00"

Add-ADFineGrainedPasswordPolicySubject `
  -Identity "AdminPasswordPolicy" `
  -Subjects "Domain Admins"
```

Запуск brute force после применения политик блокировок УЗ:

![[Pasted image 20260526220104.png]]

Можно еще сделать детекшен, который будет искать ВПО или ПО для перечисления домена, брута УЗ или спреинга УЗ. Например, поиск по EventCode=4688 в WinEventLog:Security или eventid=1 в sysmon. При этом необходим лукап с названиями процессов, который можно матчить с path иди Cmdline в логах Windows.

#### Атака DCSync
Использовал утилиту nxc с ключом --ntds:

![[Pasted image 20260526221256.png]]

#### Ключевые GUIDs репликации

| GUID                                   | Право                                      |
| -------------------------------------- | ------------------------------------------ |
| `1131f6aa-9c07-11d1-f79f-00c04fc2dcd2` | DS-Replication-Get-Changes                 |
| `1131f6ad-9c07-11d1-f79f-00c04fc2dcd2` | DS-Replication-Get-Changes-All             |
| `89e95b76-444d-4c62-991a-0facbeda640c` | DS-Replication-Get-Changes-In-Filtered-Set |

```
index="wineventlog" host=DC01 EventCode=4662 "*1131f6aa*" OR "*1131f6ad*"
| table _time, Account_Name, Account_Domain, Properties
```

![[Pasted image 20260526230154.png]]

Идея серча - искать события 4662 выполнения действий над объектами домена с Properties = `1131f6aa-9c07-11d1-f79f-00c04fc2dcd2` OR Properties = `1131f6ad-9c07-11d1-f79f-00c04fc2dcd2` OR Properties = `89e95b76-444d-4c62-991a-0facbeda640c`. Чтобы добиться распаршенного поля Properies, можно добавить searchtime-parsing в файл props.conf приложения search & reporting в Splunk.

Для события 4662 нужно было настроить аудит на объекте домена:

```
dsa.msc → lab.local → Properties → Security
→ Advanced → Auditing → Everyone
→ Replicating Directory Changes ✅
→ Replicating Directory Changes All ✅
```

#### Дамп NTDS.dit
![[Pasted image 20260526224456.png]]

![[Pasted image 20260526224520.png]]

![[Pasted image 20260526224613.png]]

Дамп успешен. Детект:

```
index="wineventlog" host=DC01 EventCode=4663
| search Object_Name="*ntds*"
| table _time, Account_Name, Process_Name, host, Accesses
```

![[Pasted image 20260526225601.png]]

Ищем события доступа к файлу ntds. Тут важно фильтровать по процессу, который инициировал доступ, а также доступы были запрошены при доступе. 

Либо универсальный серч для поиска созданий теневых копий или использование ntdsutil в логах 4688 на рабочих станциях/серверах:

```
index="wineventlog" 
    (EventCode=4663 ObjectName="*ntds*") OR
    (EventCode=4688 (CommandLine="*ntdsutil*" OR 
                     CommandLine="*vssadmin*" OR 
                     CommandLine="*shadowcopy*"))
| eval attack_type=case(
    EventCode=4663, "File Access - ntds.dit",
    like(CommandLine,"%ntdsutil%"), "Process - ntdsutil",
    like(CommandLine,"%vssadmin%") OR like(CommandLine,"%shadowcopy%"), 
    "Process - VSS Shadow Copy"
  )
| table _time, attack_type, SubjectUserName, CommandLine, ObjectName, host
| sort - _time
```

#### Усложнение атаки DCSync
```
# Убрать права репликации у конкретного пользователя
$user = Get-ADUser -Identity "<username>"
$domainDN = (Get-ADDomain).DistinguishedName
$acl = Get-ACL "AD:\$domainDN"

$replicationGuids = @(
  [GUID]"1131f6aa-9c07-11d1-f79f-00c04fc2dcd2",
  [GUID]"1131f6ad-9c07-11d1-f79f-00c04fc2dcd2"
)

foreach ($guid in $replicationGuids) {
  $ace = $acl.Access | Where-Object {
    $_.ObjectType -eq $guid -and
    $_.IdentityReference -match $user.SamAccountName
  }
  if ($ace) { $acl.RemoveAccessRule($ace) }
}
Set-ACL "AD:\$domainDN" $acl
```