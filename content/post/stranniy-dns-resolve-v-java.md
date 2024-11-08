+++
Categories = ["linux"]
Description = "Вечное кэширование у java-резолвера"
Tags = ["java","linux","server","man"]
date = "2016-11-30T19:55:29+03:00"
title = "Вечный кэш у DNS резолвера в Java"
Banner = "/img/terminal.jpg"
+++

Как известно, чтобы обеспечить кросс-платформенность, у JVM свой DNS-резолвер,
который работает отдельно от системного, и работает местами очень странно.
На днях столкнулся с интересным поведением некоторых java приложений, и долго ломал голову
в чем же дело. Оказалось, Java навечно кэширует DNS ответы.

<!--more-->
То есть если приложение ходит во внешний мир по доменному имени, например в github.com, то
IP-адрес который она получит при первом резолве останется в кэше НАВСЕГДА.
Даже если github.com изменит IP-адрес, Java об этом никогда не узнает. Спасет только рестарт приложения,
который обнулит DNS-кэш у JVM, и заставит приложение сходить еще раз разрезолвить уже новый IP-адрес.
Очень странное поведение.

Чтобы это исправить, надо запускать java-приложение с опцией:
```
-Dnetworkaddress.cache.ttl=60s
```
60s - время на которое кэшируется удачная попытка резолва.

Есть еще опция ```networkaddress.cache.negative.ttl``` которая кэширует неудачные
попытки резолва, но по умолчанию она выставлена в 10, и не закэширует неудачный ответ навсегда.

Здесь есть дополнительное описание этих опций:
[https://docs.oracle.com/javase/7/docs/technotes/guides/net/properties.html#nct](https://docs.oracle.com/javase/7/docs/technotes/guides/net/properties.html#nct)

Однако этого может быть недостаточно, и адреса все еще будут застревать в кэше. В этом случае надо добавить эту же опцию в security параметры JVM. По умолчанию файл с security-настройками лежит в ```JRE_HOME/lib/security/java.security```, чтобы добавить к этим настройкам свои, надо запускать java-процесс с опцией:
```
-Djava.security.properties=/etc/java.security.options
``` 

В ```/etc/java.security.options``` прописываем ```networkaddress.cache.ttl=60s```

Здесь есть статься с более подробным описанием настройки security:
[https://dzone.com/articles/how-override-java-security](https://dzone.com/articles/how-override-java-security)
