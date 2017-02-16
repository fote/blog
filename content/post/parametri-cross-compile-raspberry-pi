+++
Categories = ["linux"]
Description = "Параметры кросс-компиляции Golang для Raspberry Pi"
Tags = ["go","linux","golang","raspberry_pi"]
date = "2017-02-16T23:31:44+03:00"
title = "Параметры кросс-компиляции Golang для Raspberry Pi"

+++


Чтобы скомпилировать исходники на go и потом запустить полученный бинарник на raspberry pi нужно выставить определенные env-переменные перед компиляцией.


<!--more-->

Для разных версий Raspberry Pi нужно указывать разные версии ARM. Узнать какой именно нужно можно в ```/proc/cpuinfo```:
```
Processor	: ARMv6-compatible processor rev 7 (v6l)
BogoMIPS	: 697.95
Features	: swp half thumb fastmult vfp edsp java tls
CPU implementer	: 0x41
CPU architecture: 7
CPU variant	: 0x0
CPU part	: 0xb76
CPU revision	: 7

Hardware	: BCM2708
Revision	: 000f
Serial		: 00000000aa0b36f1
```

В данном случае процессор ARM версии 6. И параметры компиляции будут такие:
```
GOOS=linux GOARCH=arm GOARM=6 go build
```

