
+++
Categories = ["server"]
Description = "Обход блокировки hub.docker.com на скачивание образов из России"
Tags = ["server"]
date = "2024-05-30T23:42:42+03:00"
title = "Собственное зеркало Docker Hub"
Banner = "/img/docker-registry.jpg"
+++

Сегодня Docker заблокировал доступ к своему репозиторию контейнеров hub.docker.com для пользователей из России. Поэтому я поднял себе уютное приватное прокси-зеркало в странах не столь отдаленных. Покажу как это быстро сделать.
<!--more-->
## Проблема

Ошибка выглядит как 403 в ответ на docker pull любого образа (веб-морда hub.docker.com тоже недоступна):
{{< highlight console >}}
#docker pull nginx:latest
Error response from daemon: pull access denied for nginx, repository does not exist or may require 'docker login': denied: <html><body><h1>403 Forbidden</h1>
Since Docker is a US company, we must comply with US export control regulations. In an effort to comply with these, we now block all IP addresses that are located in Cuba, Iran, North Korea, Republic of Crimea, Sudan, and Syria. If you are not in one of these cities, countries, or regions and are blocked, please reach out to https://hub.docker.com/support/contact/
</body></html>
{{< /highlight >}}

## Быстрофикс

Я перепробовал кучу зеркал из интернета, вот это единственное, которое вызывает доверие и работает надежно - https://dockerhub.timeweb.cloud. Если не хочется заморачиваться со своим, то можно использовать его. Для этого в файл `/etc/docker/daemon.json` (на десктопе - в настройки docker engine - см.скриншот ниже) надо добавить:

{{< highlight json >}}
"registry-mirrors": [ "https://dockerhub.timeweb.cloud"]
{{< /highlight >}}

Вот так выглядит мой полный daemon.json:
{{< highlight json >}}
{
  "registry-mirrors": ["https://dockerhub.timeweb.cloud"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
{{< /highlight >}}

И перезапустить докер: `service docker restart`. После этого можно пуллить образы.


## Решение

Чтобы организовать свое приватное зеркало поднадобится виртуалка за границей. Сегодня уже многие российские VPC-провайдеры предоставляют возможность поднимать виртуалки в соседних странах, что очень удобно для обхода блокировок.

На созданной виртуалке я использовал [Docker Registry](https://distribution.github.io/distribution/) от самого докера. Сейчас он называется Distribution — это программа, которая позволяет создать свой приватный container registry. В ней есть функция проксирования+кэш внешних репозиториев. Она и понадобится.

Создаю папки:
{{< highlight console >}}
#mkdir /opt/registry
#mkdir /opt/registry/data
{{< /highlight >}}

Создаю docker-compose.yml:
{{< highlight yaml >}}
version: '3.3'

services:
  registry:
    image: registry:2
    ports:
      - "0.0.0.0:5000:5000"
    volumes:
      - "/opt/registry/config.yml:/etc/docker/registry/config.yml"
      - "/opt/registry/data:/var/lib/registry"
    restart: always
{{< /highlight >}}

И создаю конфиг `/opt/registry/config.yml`:
{{< highlight yaml >}}
version: 0.1
storage:
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true
http:
  addr: 0.0.0.0:5000
proxy:
  remoteurl: https://registry-1.docker.io
  ttl: 1d
{{< /highlight >}}

Что важно:
* проксирование в *registry-1.docker.io*
* ttl для кэша в *1 день*
* если `delete:true`, то будет очищать данные кэша старше 1-го дня, чтобы много места не съедало. Тут можно тюнить, если место на диске позволяет.

Теперь в папке `/opt/registry` выполняю:
{{< highlight console >}}
#docker-compose up -d
{{< /highlight >}}

Запускается container registry, который уже можно прописывать себе в настройки докер демона `/etc/docker/daemon.json` (чтобы демон подхватил настройки придётся его перезапустить):
{{< highlight json >}}
"registry-mirrors": [
    "http://my-private-registry.example.com:5000"
]
{{< /highlight >}}

И в настройки docker на десктопе:
![registry-mirrors в MacOS](/img/docker-registry2.jpg)

## Опционально

Для секурности можно спрятать это за nginx:

{{< highlight nginx >}}
server {
    server_name my-private-registry.example.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
    }

    listen 80; 
}
{{< /highlight >}}

Попросить certbot выдать сертификат с редиректом, и получить красивый секурный конфиг:
{{< highlight nginx >}}
server {
    server_name my-private-registry.example.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/my-private-registry.example.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/my-private-registry.example.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}server {
    if ($host = my-private-registry.example.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;
    server_name my-private-registry.example.com;
    return 404; # managed by Certbot
}
{{< /highlight >}}

И теперь в настройки докера добавить:
{{< highlight json >}}
"registry-mirrors": [
    "https://my-private-registry.example.com"
]
{{< /highlight >}}

P.S. И не забыть заменить *0.0.0.0* на *127.0.0.1* в docker-compose.yml