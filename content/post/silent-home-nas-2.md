+++
Categories = ["server"]
Description = "Продолжение истории про домашний NAS сервер. Как я заменил Seafile на Nextcloud"
Tags = ["server", "linux"]
date = "2022-06-06T22:22:14+03:00"
title = "Бесшумный NAS для дома и собственное файловое облако на Linux. Часть 2"
Banner = "/img/nas2-qnap.jpg"
draft=false
+++

Вот уже три года я использую в своём цифровом быту QNAP в качестве файлохранилки. Про выбор железки и первоначальную установку я писал в [первой части](/post/silent-home-nas) этой статьи. За это время в моем сетапе произошли изменения, и здесь поделюсь своим опытом за прошедшее время. Теперь из названия статьи можно убрать слово "файловое".

<!--more-->

## Состояние дисков

За три года ни одной ошибки в SMART-е. Диски в зеркальном RAID-е вот такие — ```WDC WDS100T2B0A-00SM50```

![Диск WD](/img/nas2-disk.png)

## Kodi, но без HDMI

Сама коробка QNAP у меня переехала в кладовку, и кабель HDMI до туда не дотянуть. Поэтому на телевизор с AndroidTV был поставлен Kodi, и в качестве источника данных указываю свой сервер, на котором тоже стоит Kodi. Телевизор подключается по Wi-Fi и тянет данные оттуда. 5 гигагерцового канала хватает, чтобы пробить пару стен и стримить FullHD видео без задержек. 

## Nextcloud vs Seafile

Прожив с Seafile три года, я понял, что мне стало мало того функционала, который он дает. А дает он очень мало возможностей, по сути только одну — хранить файлы. И никаких тебе плагинов и расширений. Превьювить файлы в браузере умеет очень ограниченно, поиск тоже хромает. 

Кроме этого, у Seafile очень бедное приложение для iOS: ни тебе галереи нормальной, ни фоновой загрузки фотографий сделанных на телефон (тут iphone правда сам виноват, потому что режет фоновую работу приложений). Под Android проблемы с фоновой загрузкой нет, но в остальном то же самое. 

А после 24 февраля, когда в России начали отключать разные облачные сервисы, стало понятно, что лучше искать self-hosted замену, а с Seafile построить какую-то собственную "экосистему" довольно сложно. Тогда я решил попробовать Nextcloud.

На тот момент в Seafile у меня было было примерно 25к файлов и 150ГБ.

## Установка Nextcloud

Я запустил Nextcloud с помощью docker-compose. Потребовалось некоторое время, чтобы подобрать параметры запуска, но в итоге финальная версия моей конфигурации выглядит так:
{{< highlight yaml >}}
version: '2'

services:
  db:
    image: mariadb
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    restart: always
    volumes:
      - /opt/nextcloud-mysql:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=<rootpassword>
      - MYSQL_PASSWORD=<dbpassword>
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    logging:
      options:
        max-file: "3"
        max-size: "50m"

  app:
    image: nextcloud:23.0.4-fpm
    links:
      - db
    ports:
      - 127.0.0.1:10000:9000
    environment:
      - NEXTCLOUD_IPADDRESS=<ipaddress>
      - NEXTCLOUD_FQDN=<hostname>
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=<dbpassword>
      - MYSQL_HOST=db
#      - NEXTCLOUD_ADMIN_USER=admin
#      - NEXTCLOUD_ADMIN_PASSWORD=<adminpassword>
    volumes:
      - /var/www/html:/var/www/html
    restart: always
    logging:
      options:
        max-file: "3"
        max-size: "50m"

  cron:
    image: nextcloud:23.0.4-fpm
    restart: always
    links:
      - db
    volumes:
      - /var/www/html:/var/www/html
    entrypoint: /cron.sh
    logging:
      options:
        max-file: "3"
        max-size: "50m"
{{< /highlight >}}

В app контейнере работает само приложение, в cron - регулярные таски которые запускаются в фоне (разметка фотографий на карте, image recognition и проч.)

В качестве входной точки на сервере я использую nginx. Вот так прокидываю запросы внутрь docker-контейнра, и использую Let'sEncrypt для HTTPS:
{{< highlight console>}}
upstream php-handler {
        server localhost:10000;
}

server {
    server_name		<hostname>;
    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/<hostname>/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/<hostname>/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    add_header Strict-Transport-Security max-age=15768000;

    root /var/www/html;

    client_max_body_size 1024m;
    # Enable gzip but do not remove ETag headers
    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

    add_header Referrer-Policy                      "no-referrer"   always;
    add_header X-Content-Type-Options               "nosniff"       always;
    add_header X-Download-Options                   "noopen"        always;
    add_header X-Frame-Options                      "SAMEORIGIN"    always;
    add_header X-Permitted-Cross-Domain-Policies    "none"          always;
    add_header X-Robots-Tag                         "none"          always;
    add_header X-XSS-Protection                     "1; mode=block" always;
    # Remove X-Powered-By, which is an information leak
    fastcgi_hide_header X-Powered-By;

    index index.php index.html /index.php$request_uri;

    location = / {
          if ( $http_user_agent ~ ^DavClnt ) {
              return 302 /remote.php/webdav/$is_args$args;
          }
      }

    location = /robots.txt {
          allow all;
          log_not_found off;
          access_log off;
      }

    location ^~ /.well-known {
          # The rules in this block are an adaptation of the rules
          # in `.htaccess` that concern `/.well-known`.
         location = /.well-known/carddav { return 301 /remote.php/dav/; }
          location = /.well-known/caldav  { return 301 /remote.php/dav/; }
         location /.well-known/acme-challenge    { try_files $uri $uri/ =404; }
          location /.well-known/pki-validation    { try_files $uri $uri/ =404; }
         # Let Nextcloud's API for `/.well-known` URIs handle all other
          # requests by passing them to the front-end controller.
          return 301 /index.php$request_uri;
      }

    location ~ /(ocm-provider|ocs-provider)/ {
        return 301 $scheme://$host/$1/;
    }

       # Rules borrowed from `.htaccess` to hide certain paths from clients
    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)  { return 404; }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console)                { return 404; }

    location ~ \.php(?:$|/) {
            # Required for legacy support
            rewrite ^/(?!index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|oc[ms]-provider\/.+|.+\/richdocumentscode\/proxy) /index.php$request_uri;

            fastcgi_split_path_info ^(.+?\.php)(/.*)$;
            set $path_info $fastcgi_path_info;

            try_files $fastcgi_script_name =404;

            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $path_info;
            #fastcgi_param HTTPS on;

            fastcgi_param modHeadersAvailable true;         # Avoid sending the security headers twice
            fastcgi_param front_controller_active true;     # Enable pretty urls
            fastcgi_pass php-handler;

            fastcgi_intercept_errors on;
            fastcgi_request_buffering off;
    }


    # Rule borrowed from `.htaccess`
    location /remote {
        return 301 /remote.php$request_uri;
    }

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }
}
{{< /highlight >}}

Дальше я поставил у себя на десктопе клиент Nextcloud и попробовал загрузить свои 25к файлов. Клиент кряхтел и пыхтил, но таки за ночь это осилил. А теперь, когда вся масса файлов загружена, добавлять по несколько десятков новых никакой проблемы не составляет.

## После установки

Конечно, отзывчивость веб-интерфейса Nextcloud после Seafile бросается в глаза, но в целом, скорость работы стала приемлемой. Три года назад было намного хуже. Nextcloud написан на PHP, так что задумчивость у него в крови :smile: . Обратной стороной этого является легкость написания плагинов.

Сразу после установки я пошел смотреть плагины, которые тут есть. И вот что я для себя выбрал:

### **Calendar**

Ну тут все понятно. CalDAV календарь, который отлично подключается в любой android/iphone календарь. Заменил им iCloud календарь.

### **Contacts**

CardDAV записаная книжка. Перенес контакты из iCloud практически без боли. Один-в-один не переносится, ломаются метки номеров "сотовый", "домашний", но это не критично.

### **Preview Generator**

Плагин, который генерирует превью для галереи. Крайне полезная штука, с ней плитки картинок в галерее загружаются значительно быстрее. Для того чтобы появлялись превьюшки для видео, надо поставить ffmpeg внутрь контейнера Nextcloud (или подключить с хоста)

### **Bookmarks**

Облачное хранилище закладок из браузера. Я пользуюсь Firefox, и для него есть плагин floccus, который позволяет загружать загладки в Nextcloud. Сценарий такой: у меня есть ноутбук и ПК, добавил я закладку на ноубуке, потом запустил ПК, а она уже и там есть. Удобно. А для мобилки есть отдельное приложение.

### **GpxEdit / GpxMotion / GpxPod**

После ухода из России Strava, для беговых/вело тернировок я начал писать gpx-трек в [OpenGPXTracker](https://apps.apple.com/ru/app/open-gpx-tracker/id984503772), и складывать в отдельную папку в Nextcloud-е. Конечно, полноценной замены Strava не получается — нет самой главной фишки — социальной сети и общения, но для ведения дневника тренировок вполне подходит. И еще нельзя в один клик создать фоточку с треком для инстаграма :smile: (идея для плагина!). Btw, из Strav-ы можно выгрузить все свои треки в gpx-формате (см. [Bulk Export](https://support.strava.com/hc/en-us/articles/216918437-Exporting-your-Data-and-Bulk-Export), но теперь только через VPN открывается)

### **Maps**

Плагин для работы с картами. Рисует на карте всю инфу до которой дотянется. Самое полезное — рисует фотки на карте по координатам в EXIF.

![Maps плагин Nextcloud](/img/nas2-maps.png)

### **Tasks**

Замена для приложения Напоминания в iPhone

### **Recognize**

Это image recognition в Nextcloud, чтобы в поиске по фото написать "собака", а он тебе покажет все фотографии на которых есть собака. Долго мучался с этим расширением. Пытался заставить его в фоне просмотреть все мои 20к фоток, ибо жрет много памяти и падает в OOM на моем хиленьком QNAP-е. С такой-то матерью удалось, но результат не оправдал ожидания — смотрит плохо, метки только на английском, часто ошибается. По итогу выглядит примерно так:


![Recognition плагин Nextcloud](/img/nas2-recognition.png)


В итоге отказался.

## Self-hosted экосистема

После установки этих плагинов, очень быстро ощутил разницу с Seafile. Там у меня была простая файловая хранилка, а тут настоящая "экосистема" с кучей сервисов. И в iOS-приложении nextcloud смогли починить проблему фоновой загрузки фото. Для этого надо приложению дать права на геолокацию, тогда оно в фоне сможет загружать фото. У приложения Seafile это почему-то работает через раз. 

В общем, выжать полезности из Nextcloud-а можно гораздо больше, чем из Seafile. Поэтому остаюсь пока на нём.