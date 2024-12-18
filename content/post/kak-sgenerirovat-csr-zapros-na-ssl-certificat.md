+++
Categories = ["man"]
Description = "Небольшой how-to про генерацию csr для получения ssl сертификата"
Tags = ["web","ssl"]
date = "2016-08-05T19:08:25+03:00"
title = "Как сгенерировать CSR заявку на получение SSL сертификата"
Banner = "/img/ssl-csr.png"
+++

Чтобы получить SSL-сертификат на домен, необходимо отправить провайдеру SSL специально сформированный запрос — CSR (Certificate Signing Request).
Сначала генерируется приватный ключ, потом на основе этого ключа генерируется CSR.
Ключ может быть шифрованный и нешифрованный. Для шифрования нужна passphrase, и в последствии при каждом использовании сертификата нужно будет вводить пароль.
<!--more-->

Генерация ключа с паролем:
{{< highlight console >}}
$openssl genrsa -des3 -out private.key 2048
{{< /highlight >}}

Генерация ключа БЕЗ пароля:
{{< highlight console >}}
$openssl genrsa -out private.key 2048
{{< /highlight >}}

Теперь на основе ключа генерируем CSR:
{{< highlight console >}}
$openssl req -new -key private.key -out request.csr
{{< /highlight >}}

После указания параметров (страна, организация, домен и проч.), появится файл с CSR. Он отправляется провайдеру SSL для получения сертификата.
