## Краткое описание:
Управление параметрами пользователя осуществляется полностью в Active Directory (либо любой другой службе каталогов)
посредством задания атрибутов и членства в группах.

Сервис получает по протоколу RADIUS запрос на аутентификацию пользователя.
Производится поиск посльзователя в Active Directory
в случае успеха, проверяется его членство в группе, разрешающей подключение по VPN (параметр otp_group в секции ldap_setting файла settings.json)
если пользователь является членом этой группы, проверяются атрибуты:
Мобильный телефон (mobile)
Электронная почта (mail)
А так же Заметки на вкладке телефоны (info)
В поле Заметки можно указать предпочитаемый метод доставки одноразового пароля, либо otpmail либо otpsms
если в этом поле уже имеется текст, укажите метод доставки в конце текста, отделив его запятой или пробелом.

Для отправки SMS требуется промежуточный сервис (мы предоставляем интеграцию с СМС Дисконт)

### Параметры в файле settings.json

    Секция ldap_setting:
    fqdn:               FQDN или IP адрес LDAP сервера.
    fqdn2:              FQDN или IP адрес резервного LDAP сервера.
    ldap_port:          LDAP порт (обычно 389)
    ldaps_port:         LDAP over SSL порт (обычно 636)
    ldaps_enabled:      включение LDAP over SSL
    base_dn:            узел в дереве откуда начинать поиск пользователей
    bind_username_upn:  имя пользователя от имени которого будет производится обращение по LDAP к контроллеру домена
                        в формате UPN (username@domain)
    bind_password:      пароль пользователя
    otp_group:          имя группы, членам которой разрешен доступ в VPN (в формате CN=<Группа>,CN=<контейнер>........,DC=<домен>,DC=local)
    
    Секция radius_setting
    shared_secret       секректный ключ
    port                порт (обычно 1812)
    address             адрес на котором слушать Radius дейтаграммы (можно оставить пустым)

    Секция otp_params:
    valid_interval:     интервал в течении которого временный пароль действителен
    otp_len:            количество цифр в одноразовом пароле - 6
    
    Секция smtp_params:
    mail_from:          Пользователь, от которого будет производится отправка письма
    mail_from_name:     "PowerMF"
    smtpserver:         IP или FQDN адрес SMTP сервера
    SMTPPort:           порт SMTP сервера
    subject:            тема в письме
    message:            текст помимо пароля
    smtp_password:      пароль на SMTP соединение
    
    Секция sms_params:
    smsurl:             URL шлюза SMS - сейчас возможен только СМС Дисконт - "https://api.iqsms.ru/messages/v2/send.json"
    smslogin":          Имя пользователя для аутентификации на SMS шлюзе
    smspassword":       Пароль для аутентификации на SMS шлюзе
    smscert:            Сертификат для аутентификации на SMS шлюзе
    smskey:             Закрытый ключ
    json:               1 - Использовать формат json (для указанного выше URL это так)
    smsca:              корневой сертификат - не обязателен
    authbycert:         Если аутентификация по логину/паролю то 0, если по сертификату то 1 (для СМС Дисконт - 0)
    checkidentity:      1 - если проверять валидность сертификата сервера и 0 если не проверять

## Установка
Перейти в папку /opt

**cd /opt**

выполнить:
#### git clone https://github.com/OlegPowerC/powermf.git
перейти в папку powermf

**cd powermf**

сделать исполняемым файл init.sh

**chmod +x ./init.sh**

и выполнить его

**./init.sh**

Дождаться появления информации о лицензии и сохранить ее

Связаться с нами по e-mail: info@powerc.ru или по телефону и приобрести лицензию

После получения файла license.dat скопировать его в папку lic

Для запуска как сервис включить сервис

**systemctl enable /opt/powermf/powermf.service**

Запустить

**service powermf start**

Разрешить принимать дейтаграммы на нужный порт в файрволе

**firewall-cmd --zone=public --permanent --add-port=1812/udp**

**firewall-cmd --reload**

#### Журналирование
Журнал ведется в текстовый файл в папке ***logs***

### Утилита генерации секретного ключа для программного TOTP генератора
Утилиты лежат в папке **tools** под windows и linux

### Настройка шлюза VPN на примере Cisco ASA

    laaa-server ADLDAP protocol ldap
    aaa-server ADLDAP (inside) host 192.168.0.2
    server-port 389
    ldap-base-dn dc=EXAMPLE, dc=LOCAL
    ldap-scope subtree
    ldap-naming-attribute sAMAccountName
    ldap-login-password TestPass123
    ldap-login-dn cn=ASA, cn=Users, dc=EXAMPLE, dc=LOCAL
    server-type microsoft
    
    aaa-server RDTEST protocol radius
    aaa-server RDTEST (inside) host 192.168.0.5
    key radiuskeytest123
    authentication-port 1812
    
    tunnel-group TWTEST type remote-access
    tunnel-group TWTEST general-attributes
    authentication-server-group ADLDAP
    secondary-authentication-server-group RDTEST use-primary-username

В примере:
192.168.0.2 - это адрес LDAP сервера
192.168.0.5 - это адрес Linux хоста на котором запущен сервис PowerMF

###  Пример настройки Active Directory

![ad example image](https://github.com/OlegPowerC/powermf/blob/master/addata.png)


### Тест при использовании в качестве VPN шлюза, Cisco ASA

**test aaa-server authentication RDTEST host 192.168.0.5 username User password 123654**

В случае указывания реального пользователя - члена группы указанной в параметре otp_group
у которого есть токен (secretkey в поле info)
и пароля сгенерированного генератором OTP в логе будет сообщение:

INFO: Authentication Successful

В случае указания неверного пароля, в логе будет:

ERROR: Authentication Rejected: AAA failure
