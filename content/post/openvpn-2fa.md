+++
Categories = ["server"]
Description = "How-to: настройка OpenVPN-сервера с двухфакторной аутентификацией"
Tags = ["openvpn","linux","server","docker"]
date = "2023-01-23T02:25:24+03:00"
title = "OpenVPN и двухфакторная аутентификация"
Banner = "/img/openvpn-2fa.png"
+++

Ранее я писал как поднять OpenVPN сервер и подключаться к нему используя только клиентский сертификат из .ovpn файла. А сейчас покажу как запустить OpenVPN с двухфакторной аутентификацией пользователей. Использовать буду тот же готовый docker-образ `kylemanna/docker-openvpn`.

<!--more-->

## Как это работает

Общая схема такая: в докер-контейнере работает OpenVPN-сервер и PAM-плагин [google-authenticator](https://github.com/google/google-authenticator). Клиент подключается и для аутентификации предоставляет клиентский сертификат из конфига .ovpn (первый фактор) и одноразовый пароль, генерируемый приложением в телефоне (второй фактор). Приложение называется [Google Authentificator](https://ru.wikipedia.org/wiki/Google_Authenticator), хотя подойдет любое приложение реализующее TOTP-алгоритм для генерации одноразовых паролей.

![Схема работы](/img/openvpn-2fa-1.png)

## Запускаем сервер

Предполагаю, что на сервере уже работает Docker, поэтому для создания OpenVPN-сервера потребуется всего пара команд. 

Создадим директорию для конфига:
{{< highlight console >}}
mkdir -p /etc/openvpn
{{< /highlight >}}

Генерируем конфиг:
{{< highlight console >}}
docker run -v /etc/openvpn:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://vpn.example.com -2 -C AES-256-CBC 
{{< /highlight >}}

Здесь:
* вместо `vpn.example.com` подставьте свой домен или IP-адрес сервера
* параметр `-2` указывает, что мы хотим сгенерировать конфиг с 2FA
* шифр AES-256-CBC

Создаем корневые сертификаты. Потербуется ввести пассфразу. Она понадобится в дальнейшем при создании пользователей:
{{< highlight console >}}
docker run -v /etc/openvpn:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
{{< /highlight >}}

Запускаем сервер:
{{< highlight console >}}
docker run -v /etc/openvpn:/etc/openvpn -d --name openvpn --restart=always -p 1194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn
{{< /highlight >}}

Готово. Сервер работает. Теперь создадим пользователя. Вот готовая копипаста, в которой выполняется несколько команд последовательно. В ней нужно заменить несколько параметров:
{{< highlight console >}}
export username="user1" &&
docker exec -it openvpn easyrsa build-client-full $username nopass &&
docker run -v /etc/openvpn:/etc/openvpn --rm -it kylemanna/openvpn ovpn_otp_user $username &&
docker exec -it openvpn google-authenticator --time-based --disallow-reuse --force --rate-limit=3 --rate-time=30 --window-size=3 -l "${username}@vpn.example.com" -s /etc/openvpn/otp/${username}.google_authenticator &&
docker run -v /etc/openvpn:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient $username > $username.ovpn
{{< /highlight >}}
 
Здесь:
* в параметре `username` задайте имя пользователя
* `vpn.example.com` замените на свой домен или IP-адрес

Командой выше мы генерируем клиентский сертификат (первый фактор) и настраиваем TOTP-алгоритм (второй фактор). Во время выполнения на консоль будет дважды выведен QR-код, наc интересует второй. Выполнение остановится и потребует код из приложения:
{{< highlight console >}}
Your new secret key is: YD********************
Enter code from app (-1 to skip):
{{< /highlight >}}

Вводим `-1` и пропускаем этот шаг. Если все успешно, в текущем каталоге появится файл конфигурции клиента `user1.ovpn`. Клиенту для подключения понадобится этот файл и QR-код (второй). Ссылку на QR-код можно найти в выводе команды.

ВАЖНО. Нельзя шарить QR-код ни с кем кроме клиента. Иначе это скомпрометирует второй фактор.

## Настройка клиента

Клиент берет QR-код, который мы ему передали, и добавляет его в свое мобильное приложение Google Authentificator. После этого у него появится OTP-пароль, который явлется вторым фактором для подключения и обновляется каждые 30 секунд.
![Google Authentificator](/img/openvpn-2fa-3.png)


Далее в [OpenVPN клиенте](https://openvpn.net/vpn-client/) импортирует конфиг .ovpn. Важно при импорте указать Username:
![Импорт конфига](/img/openvpn-2fa-2.png)

После нажатия на кнопку Connect потребуется ввести пароль, тот самый OTP из мобильного приложения Google Authentificator. Если пароль верный, то соединение пройдет успешно.

P.S.

1. Вот [здесь](https://github.com/kylemanna/docker-openvpn/blob/master/docs/otp.md) есть доп. информация о 2FA в данном докер образе.
2. [Здесь](/post/install-openvpn-docker/) я делал OpenVPN-сервер с некоторыми кастомизациями (не дефолтный пароль и возможность одному клиенту подключаться много раз одновременно). Там есть готовые копипастные команды. Возможно тоже будет полезно
3. Удаление пользователя:
{{< highlight console >}}
docker run --rm -it -v /etc/openvpn:/etc/openvpn kylemanna/openvpn ovpn_revokeclient user1 remove
{{< /highlight >}}
