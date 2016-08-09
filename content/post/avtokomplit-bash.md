+++
Categories = ["linux"]
Description = "Некоторые советы при работе в bash"
Tags = ["bash"]
date = "2015-05-15 04:56:21 -0400"
title = "Автокомплит для ssh"

+++

Чтобы после ssh автоматом по нажатию [TAB][TAB] подставлялись хосты из know_hosts:

```bash
complete -W "$(echo `cat ~/.ssh/known_hosts | cut -f 1 -d ' ' | sed -e s/,.*//g | uniq | grep -v "\["`;)" ssh
```
