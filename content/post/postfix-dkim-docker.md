+++
Categories = ["server"]
Description = "Мануал как поднять MTA с нуля в docker-контейнере и настроить DKIM PTR SPF DMARC чтобы не попадать в спам"
Tags = ["postfix", "linux"]
date = "2018-06-10T18:15:24+03:00"
title = "Поднимаем почтовый сервер для рассылок: POSTFIX + DKIM + Docker"
Banner = "/img/postfix_3.png"
+++

Тут опишу как поднять почтовый сервер на базе Postfix-а для рассылки почты. Также расскажу про DKIM, SPF, DMARC. Поднимать буду в docker-контейнере, потому что это просто и быстро.

<!--more-->

Общая схема будет выглядеть так:
![Схема работы](/img/postfix_2.png)


## PTR-запись

Основное что нужно прописать, чтобы почта не попадала в спам-листы - это обратная запись PTR. *Она должна указывать на сервер с которого будут отправляться письма*. То есть, если мы отправляем письма с сервера mydomain.ru с IP-адресом 11.22.33.44, то PTR должна быть:

```
44.33.22.11.in-addr.arpa. IN PTR mydomain.ru.
```

PTR-записи прописывается тем, кому приндалежит этот IP-адрес. Если это какой-то облачный провайдер, то в админке обычно есть редактор PTR-записей. В DigitalOcean создается автоматически по имени инстанса - имя должно быть FQDN:

![PTR-запись в админке DigitalOcean](/img/postfix_1.png)


## SPF-запись

Это TXT-запись в DNS зоне домена в которой указываются адреса серверов с которых происходит отпарвка почты у данного домена. Когда почтовый сервер приемника получает письмо с адресом отправителя ```example@mydomain.ru``` он идет в DNS и проверяет SPF запись у домена example.com. Пример такой записи:
```
mydomain.ru. IN TXT "v=spf1 a mx ip4:11.22.33.0/24 ip4:2.2.2.2 ip6:2a03:b0c0:0:1010::424:8001/64 -all"
```

В данном примере:

 - ```v=spf1``` - версия SPF
 - ```a``` - разрешить прием писем с адреса указанного в A записи example.com
 - ```mx``` - разрешить прием писем с адреса указанного в MX записи example.com
 - ```ip4:1.1.1.0/24 ip4:2.2.2.2``` - разрешить прием писем из сети 1.1.1.0/24; разрешить прием писем с адреса 2.2.2.2
 - ```ip6:2a03:b0c0:0:1010::424:8001/64``` - то же самое что ip4, но для IPv6
 - ```-all``` - отбросить письмо, пришедшее не от указанных адресов. Можно вместо -all указать ~all, что значит - письмо не отбрасывать, но проверить остальными способами.

 Все поля опциональные, не обязательно указывать все. Здесь есть более подробное описание синтаксиса SPF-записи: [http://www.openspf.org/SPF_Record_Syntax](http://www.openspf.org/SPF_Record_Syntax)


## DKIM

Механизм подписи всех писем цифровой подписьмю и проверки валидности подписи приемником. В общем виде работает так:

1. Генерируем пару приватный/публичный ключ
2. Кладем публичный ключ в TXT запись домена
3. Приватный ключ отдаем почтовому серверу postfix (ниже описано как настроить), и настраиваем чтобы он подписывал все письма этим ключом
4. Когда почтовый сервер получает письмо, он проверяет что подписано оно ключом из TXT записи домена

Настроим.
Создаем пару ключей:
{{< highlight console >}}
# openssl genrsa -out private_dkim_key.pem 1024
# openssl rsa -pubout -in private_dkim_key.pem -out public_dkim_key.pem 
{{< /highlight >}}

Добавляем TXT запись 
```
myselector._domainkey.mydomain.ru. TXT "v=DKIM1\; k=rsa\; t=s\; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDQmO9AuWRbWPgljzDPQodrLfFLFqYY"
```

Здесь:
- ```v=DKIM1``` - версия DKIM, всегда DKIM1
- ```myselector``` - селектор в самом домене. Может быть произвольным. Нужен чтобы на один домен можно было повесить несколько ключей. Запоминаем его - он понадобится при настройке Postfix
- ```k=rsa``` - тип ключа; обычно - rsa
- ```t=s``` - указывает на то, что письма не отправляются с субдоменов этого домена
- ```p={YOUR_PUBLIC_KEY}``` - публичная часть ключа, которую генерировали выше

## DMARC

DMARC - это тоже TXT запись в которой указывается политика для принимающего сервера, что делать с письмами не прошедшими проверку SPF и DKIM. Вот пример DMARC записи:

```
_dmarc.mydomain.ru TXT "v=DMARC1; fo=0; p=none; rua=mailto:dmarc_reports@mydomain.ru"
```

Здесь:

- ```v=DMARC1``` - версия DMARC. Всегда DMARC1
- ```p=none``` - что делать с письмами не прошедшими проверку. Может быть none - ничего не делать, только слать отчет, quarantine - добавлять письмо в спам, reject - отклонять письмо.
- ```fo=0``` - политика отправки отчетов; 0 - отправлять отчет только если письмо не прошло и DKIM и SPF; 1 - если не прошло или DKIM или SPF; s - не прошло SPF; d - не прошло DKIM
- ```rua=mailto:dmarc_reports@mydomain.ru``` - адрес на который будут слаться отчеты от почтовых серверов в случае обнаружения, не прошедших проверку, писем


## Установка Postfix в docker-контейнере

Запустим контейнер с Postfix-ом и открытым 25 портом в который будет ходить наше приложение для отправки почты.
Предлагаю воспользоваться уже собранным образом вот из этого репозитория - [https://github.com/MarvAmBass/docker-versatile-postfix](https://github.com/MarvAmBass/docker-versatile-postfix)

Создадим на хост машине папки для DKIM, почтовой очерди и логов. Потом эти папки подмонтируем в контейнер:
{{< highlight console >}}
# mkdir -p /etc/postfix/dkim
# mkdir -p /var/spool/postfix
# mkdir -p /var/log/postfix
{{< /highlight >}}

DKIM приватный ключ (публичный в TXT записи), который генерировали выше положим сюда (имя обязательно такое): 
```
/etc/postfix/dkim/dkim.key
```

Запускаем контейнер:
{{< highlight console >}}
# docker run -d --name=postfix \
 -p 127.0.0.1:25:25 \
 -e POSTFIX_RAW_CONFIG_SMTPD_USE_TLS=no \
 -v /var/spool/postfix:/var/spool/postfix \
 -v /var/log/postfix:/var/log \
 -v /etc/postfix/dkim:/etc/postfix/dkim/ \
 marvambass/versatile-postfix mydomain.ru no-reply@N0replyuserP@ssw0rd
{{< /highlight >}}

По умолчанию в этом образе включен TLS, я его отключил в переменной POSTFIX_RAW_CONFIG_SMTPD_USE_TLS, потому что ходить в этот Postfix будут соседние контейнеры на этом же хосте. При необходимости можно опубликовать 25 порт контейнера на интерфейс локальной сети (но не в интернет!), чтобы приложение с соседних хостов могло посылать почту. 

Теперь можно отправлять почту с хост машины, или прилинковывать к этому контейнеру соседние, чтобы они тоже получали доступ в 25 порт.

Проверить работоспособность можно маленькой утилиткой mail-checker [http://github.com/fote/mail-checker](http://github.com/fote/mail-checker). Она отправляет тестовое письмо на указанный адрес через только что поднятый MTA:

{{< highlight console >}}
# wget -O mail-checker https://github.com/fote/mail-checker/releases/download/1.0/mail-checker.linux.amd64 && chmod +x mail-checker

# ./mail-checker -mailHost 127.0.0.1 -mailPort 25 -username no-reply -password N0replyuserP@ssw0rd -to mymail@gmail.com -from no-reply@mydomain.ru
{{< /highlight >}}


Или по старинке через nc / telnet:
{{< highlight console >}}
# echo -ne '\no-reply\0N0replyuserP@ssw0rd' | openssl enc -base64
Cm8tcmVwbHkATjByZXBseXVzZXJQQHNzdzByZA==


# nc -v 127.0.0.1 25
172.17.0.4 (172.17.0.4:25) open
220 mydomain.ru ESMTP
ehlo localhost
...
...
auth plain Cm8tcmVwbHkATjByZXBseXVzZXJQQHNzdzByZA==
235 2.7.0 Authentication successful

mail from: no-reply@mydomain.ru
250 2.1.0 Ok

rcpt to: mymail@yandex.ru
250 2.1.5 Ok

data
354 End data with <CR><LF>.<CR><LF>

TestMessage
Ignore
.
250 2.0.0 Ok: queued as 45F6F487AB

quit
221 2.0.0 Bye
{{< /highlight >}}

Если все было настроено правильно, то письмо не должно попасть в спам, а если заглянуть в свойства, там будет примерно следующее:

![DKIM pass spf pass dmarc pass](/img/postfix_4.png)





