+++
Categories = ["linux"]
Description = "Как подключить камеру к Raspberry Pi Zero W"
Tags = ["raspberry_pi"]
date = "2018-02-20T16:32:44+03:00"
title = "Подключение камеры к Raspberry Pi Zero W"
Banner = "/img/rpi_camera_0.jpg"
+++


Приехала камера с Aliexpress для подключения к Raspberry Pi через разъем [DSI](https://ru.wikipedia.org/wiki/Display_Serial_Interface). Пока подключал прошел по некоторым "граблям". Расскажу подробней.

<!--more-->

## Подключение и настройка

Камера из вот этого [набора](https://ru.aliexpress.com/item/7-in-1-Raspberry-Pi-Zero-Camera-Holder-Acrylic-Case-Heat-Sink-Mini-HDMI-Adapter-GPIO/32803652515.html?spm=a2g0s.9042311.0.0.5hwqJs).


Подключил камеру к DSI разъему и пошел в утилиту ```raspi-config```. В ней, в разделе ```Interfaces options``` есть пункт ```P1 Camera``` которая включает этот разъем в системе.

![raspi-config](/img/rpi_camera_1.png)

После включения можно попробовать воспользоваться утилитой ```raspistill```. Это простая консольная программа для снятия изображения с камеры. Для видео также есть ```raspivid```. В Raspbian она есть уже "изкоробки", для других дистрибутивов, возможно, потребуется установить:
{{< highlight console >}}
# apt-get install libraspberrypi-bin
{{< /highlight >}}

Однако с ходу получить изображение с камеры не получилось. Ругается:
{{< highlight console >}}
# raspistill -o img1.jpg
mmal: Cannot read camera info, keeping the defaults for OV5647
mmal: mmal_vc_component_create: failed to create component 'vc.ril.camera' (1:ENOMEM)
mmal: mmal_component_create_core: could not create component 'vc.ril.camera' (1)
mmal: Failed to create camera component
mmal: main: Failed to create camera component
mmal: Camera is not enabled in this build. Try running "sudo raspi-config" and ensure that "camera" has been enabled
{{< /highlight >}}

Смотрим определяется ли камера в системе:
{{< highlight console >}}
# vcgencmd get_camera
supported=0 detected=0
{{< /highlight >}}

Если нет, то скорее всего должно помочь обновление firmware. Обновляю:
{{< highlight console >}}
#  rpi-update
 *** Raspberry Pi firmware updater by Hexxeh, enhanced by AndrewS and Dom
 *** Performing self-update
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 13403  100 13403    0     0  30595      0 --:--:-- --:--:-- --:--:-- 30670
 *** Relaunching after update
 *** Raspberry Pi firmware updater by Hexxeh, enhanced by AndrewS and Dom
 *** We're running for the first time
 *** Backing up files (this will take a few minutes)
 *** Backing up firmware
 *** Backing up modules 4.9.41+
#############################################################
This update bumps to rpi-4.9.y linux tree
Be aware there could be compatibility issues with some drivers
Discussion here:
https://www.raspberrypi.org/forums/viewtopic.php?f=29&t=167934
##############################################################
 *** Downloading specific firmware revision (this will take a few minutes)
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   168    0   168    0     0    196      0 --:--:-- --:--:-- --:--:--   196
100 54.1M  100 54.1M    0     0   750k      0  0:01:13  0:01:13 --:--:-- 1308k
 *** Updating firmware
 *** Updating kernel modules
 *** depmod 4.9.75-v7+
 *** depmod 4.9.75+
 *** Updating VideoCore libraries
 *** Using HardFP libraries
 *** Updating SDK
 *** Running ldconfig
 *** Storing current firmware revision
 *** Deleting downloaded files
 *** Syncing changes to disk
 *** If no errors appeared, your firmware was successfully updated to 9f5eea78c4776fe82511284754887187a22d78c7
 *** A reboot is needed to activate the new firmware
{{< /highlight >}}

После обновления - ребут, и камера начинает работать:
{{< highlight console >}}
# vcgencmd get_camera
supported=1 detected=1

# raspistill -o img1.jpg
{{< /highlight >}}

Изображение с камеры выводится на экран:
![raspberry pi camera image](/img/rpi_camera_2.jpg)



Здесь подробная документация про raspistill:
[https://www.raspberrypi.org/app/uploads/2013/07/RaspiCam-Documentation.pdf](https://www.raspberrypi.org/app/uploads/2013/07/RaspiCam-Documentation.pdf)


Также можно пользоваться python-библиотекой ```picamera``` для работы с Rpi камерой:
[http://picamera.readthedocs.io/en/release-1.12/install.html](http://picamera.readthedocs.io/en/release-1.12/install.html)


