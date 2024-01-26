+++
Categories = ["server"]
Description = "Инструкция как запустить Selenium в Docker"
Tags = ["server","linux","selenium"]
date = "2020-01-09T17:15:14+03:00"
title = "Поднимаем Selenium в Docker за 2 минуты"
Banner = "/img/selenium-docker.png"
+++

Короткая инструкция как быстро поднять кластер для Selenium-тестов при помощи Docker.
<!--more-->


## Введение

Мне известны два способа поднятия Selenium в docker-контейнерах. Можно взять стандартные Selenium node и hub (https://github.com/SeleniumHQ/docker-selenium) и поднимать их, но они довольно нестабильные, и приходится перезагружать их переодически, иначе зависают. А можно использовать [Selenoid](https://aerokube.com/selenoid). Это демон, который работает с Docker API и поднимает браузеры в контейнерах. 

Когда тест отправляет запрос в Selenoid, в этот момент он поднимает контейнер с запрашиваемым браузером и проксирует дальнейшие запросы внутрь. Поднятие браузера занимает несколько секунд (в зависимости от браузера). Схематично это выглядит вот так:
![Схема работы Selenoid](https://github.com/aerokube/selenoid/raw/master/docs/img/selenoid-animation.gif)


Здесь расскажу как организовать Selenium кластер на основе Selenoid.

## Установка

Буду использовать Linux, но это будет работать и для MacOS(только Intel процессоры)/Windows, и вообще везде где работает Docker. 

Запустить демон Selenoid можно одной командой с помощью утилиты [cm](https://aerokube.com/cm/latest/). Качаем ее:
{{< highlight console >}}
$curl -s https://aerokube.com/cm/bash | bash
{{< /highlight >}}

И запускаем:
{{< highlight console >}}
$./cm selenoid start --vnc
> Using Docker
- Your Docker API version is 1.40
> Selenoid is already downloaded
> Configuring Selenoid...
> Processing browser "firefox"...
- Fetching tags for image selenoid/firefox
registry.ping url=https://registry.hub.docker.com/v2/
registry.tags url=https://registry.hub.docker.com/v2/selenoid/firefox/tags/list repository=selenoid/firefox
- Requested to download VNC images...
- Pulling image selenoid/vnc_firefox:71.0
- Pulling image selenoid/vnc_firefox:70.0
> Processing browser "chrome"...
- Fetching tags for image selenoid/chrome
registry.tags url=https://registry.hub.docker.com/v2/selenoid/chrome/tags/list repository=selenoid/chrome
- Requested to download VNC images...
- Pulling image selenoid/vnc_chrome:79.0
- Pulling image selenoid/vnc_chrome:78.0
> Processing browser "opera"...
- Fetching tags for image selenoid/opera
registry.tags url=https://registry.hub.docker.com/v2/selenoid/opera/tags/list repository=selenoid/opera
- Requested to download VNC images...
- Pulling image selenoid/vnc_opera:65.0
- Pulling image selenoid/vnc_opera:64.0
> Pulling video recorder image...
- Pulling image selenoid/video-recorder:latest-release
> Configuration saved to /home/fote/.aerokube/selenoid/browsers.json
> Selenoid is already running
{{< /highlight >}}

По дефолту она скачивает образы докер-контейнеров для двух последних версий Firefox, Chrome, Opera. Мы указали флаг ```--vnc```, чтобы скачать образы к которым потом можно будет подключиться по VNC и прогонять ручные тесты. Если ручных тестов нет, то можно запускать без этого флага, тогда он скачает образы без VNC.

Когда образы скачаются, запустится Selenoid и повесится на порт 4444. Можно туда нацеливать свои тесты.

{{< highlight console >}}
$docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED              STATUS              PORTS                    NAMES
1f5068e6845b        aerokube/selenoid:latest   "/usr/bin/selenoid -…"   About a minute ago   Up About a minute   0.0.0.0:4444->4444/tcp   selenoid
{{< /highlight >}}

Доступные браузеры и сколько сейчас запущено можно смотреть по урлу ```/status```:
{{< highlight console >}}
$curl http://localhost:4444/status
{"total":5,"used":0,"queued":0,"pending":0,"browsers":{"chrome":{"78.0":{},"79.0":{}},"firefox":{"70.0":{},"71.0":{}},"opera":{"64.0":{},"65.0":{}}}}
{{< /highlight >}}

## Пример Python теста

Вот у меня есть пример теста на python:
{{< highlight python >}}
from selenium import webdriver
from selenium.webdriver.common.keys import Keys

capabilities = {
    "browserName": "chrome",
    "version": "78.0",
    "platform": "LINUX"
}

driver = webdriver.Remote(
    command_executor='http://localhost:4444/wd/hub',
    desired_capabilities=capabilities
)

try:
    print 'Session ID is: %s' % driver.session_id
    print 'Opening the page...'
    driver.get('https://duckduckgo.com/')

    print 'Taking screenshot...'
    driver.get_screenshot_as_file(driver.session_id + '.png')
finally:
    driver.quit()
{{< /highlight >}}

Просто открывает duckduckgo.com и делает скриншот. Нацеливаю его на локально работающий selenoid - ```
http://localhost:4444/wd/hub```, указываю ```chrome 78.0``` в capabilities и запускаю:
{{< highlight console >}}
$time python test.py
Session ID is: 4ce050ba-fd1a-44df-a19b-e4e2da134232
Opening the page...
Taking screenshot...

real	0m8.634s
{{< /highlight >}}

8 секунд - хороший результат. Рядом с тестом появился скриншот:
![Скриншот Python теста](/img/selenium-docker4.png)


## Web-интерфейс

У selenoid есть и web-UI, он устанавливается отдельно. Это отдельный контейнер. Запустить можно так же с помощью ```cm```:
{{< highlight console >}}
$./cm selenoid-ui start
> Using Docker
- Your Docker API version is 1.40
> Downloading Selenoid UI...
- Fetching tags for image aerokube/selenoid-ui
registry.ping url=https://registry.hub.docker.com/v2/
registry.tags url=https://registry.hub.docker.com/v2/aerokube/selenoid-ui/tags/list repository=aerokube/selenoid-ui
- Pulling image aerokube/selenoid-ui:1.9.1
> Starting Selenoid UI...
> Successfully started Selenoid UI
{{< /highlight >}}

Теперь в браузере открываю ```http://localhost:8080/``` и вижу:

![Selenoid UI](/img/selenium-docker1.png)

На вкладке Capabilites есть примеры кода для разных языков, и можно запустить браузер вручную. Ручной запуск будет работать только если при старте selenoid был указан флаг ```--vnc```. 

Когда нажимаем ```Create Session```, Selenoid запускает докер контейнер с настоящим браузером и с помощью novnc клиента на веб-морде можно подключиться внутрь и посмотреть на браузер в контейнере. Он полностью рабочий и им можно манипулировать

![Ручной запуск браузеров в Selenoid UI](/img/selenium-docker2.png)

Получаются такие браузеры внутри браузера :)

![Selenium в Docker](/img/selenium-docker3.png)


## Заключение

У самого Selenoid есть огромное количество разных фичей, можно запускать Windows и Android браузеры, полная дока есть вот здесь - https://aerokube.com/selenoid/latest/

