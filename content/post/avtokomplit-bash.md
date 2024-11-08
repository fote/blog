+++
Categories = ["linux"]
Description = "Некоторые советы при работе в bash"
Tags = ["bash","man"]
date = "2015-05-15 04:56:21 -0400"
title = "Автокомплит из know_hosts для ssh"
Banner = "/img/terminal.jpg"
+++


Чтобы после ssh автоматом по нажатию [TAB][TAB] подставлялись хосты из know_hosts:
<!--more-->

{{< highlight console >}}
#complete -W "$(echo `cat ~/.ssh/known_hosts | cut -f 1 -d ' ' | sed -e s/,.*//g | uniq | grep -v "\["`;)" ssh
{{< /highlight >}}