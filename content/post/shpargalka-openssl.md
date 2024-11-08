+++
Categories = ["server"]
Description = "Список основных команд OpenSSL для использования в быту"
Tags = ["server", "cheatsheet","man"]
date = "2023-10-31T14:21:14+03:00"
title = "Шпаргалка OpenSSL"
Banner = "/img/shpargalka-openssl.jpg"
+++

Cобрал команды, которые часто использую в быту, для быстрой копипасты.
<!--more-->

Узнать версию OpenSSL:
{{< highlight console >}}
$openssl version
{{< /highlight >}}

## Просмотр сертификатов

Просмотреть сертификат из файла:
{{< highlight console >}}
$openssl x509 -noout -text -in 'certfile.cer'
{{< /highlight >}}

Просмотреть сертификат на порту:
{{< highlight console >}}
$echo | openssl s_client -showcerts -connect 4te.me:443 </dev/null | openssl x509 -text | less
{{< /highlight >}}

Просмотреть сертификат на порту с использованием SNI (если на одном адресе висит сразу несколько SSL-сайтов, то перед установкой соедниения нужно передать домен):
{{< highlight console >}}
$echo | openssl s_client -showcerts -servername 4te.me -connect 4te.me:443 </dev/null | openssl x509 -text
{{< /highlight >}}

* параметр `-servername` указывает домен для SNI
* параметр `-connect` указывает адрес сервера, к которому нужно подключиться

## Генерация сертификата

Создать ключ и самоподписанный сертификат на 365 дней (домен указывается в интерактивном режиме в поле Common Name):
{{< highlight console >}}
$openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout privkeyfile.key -out certfile.cer
{{< /highlight >}}

Создать ключ и самоподписанный сертификат на 365 дней в **неинтерактивном** режиме:
{{< highlight console >}}
$openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout privkeyfile.key -out certfile.cer -subj "/C=RU/ST=State/L=City/O=CompanyName/OU=CompanySectionName/CN=domain.ru"
{{< /highlight >}}

Создание сертфиката на основе существующего ключа (для неинтерактивного режима добавить `-subj`; см.выше):
{{< highlight console >}}
$openssl req -x509 -key privkeyfile.key -new -days 365 -out certfile.cer
{{< /highlight >}}

## Получение сертификата

Для получение сертификата из центра сертификации (CA), нужно сгенерировать заявку на получение сертификата -- CSR. Она генерируется на основе приватного ключа.

Для начала генерируем ключ БЕЗ пароля:
{{< highlight console >}}
$openssl genrsa -out privkeyfile.key 2048
{{< /highlight >}}

Теперь на основе ключа генерируем CSR:
{{< highlight console >}}
$openssl req -new -key privkeyfile.key -out request.csr
{{< /highlight >}}
После этого файл *request.csr* отправляется в центр сертификации.

Прочитать CSR:
{{< highlight console >}}
$openssl req -in request.csr -text -noout
{{< /highlight >}}


## Просмотр ключей

Прочитать файл ключа:
{{< highlight console >}}
$openssl rsa -text -in privkeyfile.pem -noout
{{< /highlight >}}

Извлечь публичный ключ из файла:
{{< highlight console >}}
$openssl rsa -in privkeyfile.pem -pubout
{{< /highlight >}}
