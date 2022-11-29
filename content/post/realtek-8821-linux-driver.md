+++
Categories = ["desktop", "linux"]
Description = "Драйвер для RTL8811CU RTL8821CU RTL8731AU 0bda:c811"
Tags = ["linux", "desktop"]
date = "2022-11-30T01:40:42+03:00"
title = "Установка Wi-Fi драйвера Realtek в Linux"
Banner = "/img/rt8821_2.png"
+++

Установка драйвера для Wi-Fi адаптера от Realtek под Linux может стать небольшой проблемой. Есть несколько версий драйверов на github, но не все подходят для ядра линукс версий 5.+ . Здесь покажу рабочий вариант

<!--more-->

У меня USB адаптер, и вот так он выглядит в *lsusb*. ID: **0bda:c811**
{{< highlight console >}}
Bus 001 Device 003: ID 0bda:c811 Realtek Semiconductor Corp. 802.11ac NIC
{{< /highlight >}}

## Установка

Вот этот драйвер подойдет к адаптерам на базе **RTL8811CU** **RTL8821CU** **RTL8731AU**, и к Linux kernel версий **4.19-6.1**:
[https://github.com/morrownr/8821cu-20210118](https://github.com/morrownr/8821cu-20210118)

Устанавливаю на Ubuntu 22.10:
{{< highlight console >}}
sudo apt update && sudo apt install build-essential git dkms bc
git clone https://github.com/morrownr/8821cu-20210118.git
cd 8821cu-20210118
sudo ./install-driver.sh
{{< /highlight >}}

Если все прошло успешно, скрипт спросит "Do you want to edit the driver options file now?" - отказываемся. На предложение перезагрузки соглашаемся :)

После ребута получаем рабочий Wi-Fi адаптер:
![iwconfig](/img/rt8821.png)
