+++
Categories = ["desktop"]
Description = "Как вернуть флаги в MacOS 12.4 Monteray"
Tags = ["macos", "desktop"]
date = "2020-05-28T01:44:42+03:00"
title = "Возвращаем флаги раскладок клавиатуры в трей MacOS 12.4"
Banner = "/img/flags-macos1.png"
+++

После обновления MacOS Monterey до 12.4, в трее изменилась иконка отображения текущей раскладки клавиатуры. Раньше были флаги, а теперь буквенное обозначение раскладки. Это выглядит красиво и лаканично, но в реальной жизни приходится всматриваться, чтобы понять какая сейчас раскладка включена. ~~Раньше было лучше, эпол верни всё взад~~. Покажу маленький костыль как вернуть обратно флаги в трей. 
<!--more-->

Этот способ я подсмотрел на [iphones.ru](https://www.iphones.ru/iNotes/kak-vernut-flagi-dlya-yazykov-klaviatury-v-status-bare-macos-124-i-novee-05-18-2022), однако там иконки сделаны неаккуратно, поэтому я заменил иконки на те, что более похожи на родные МакОСёвые.

## Возвращаем флаги

1. Скачиваем [ru-keyboard-with-icon.bundle.zip](/files/ru-keyboard-with-icon.bundle.zip) и распаковываем.
2. Кладем ```ru-keyboard-with-icon.bundle``` файл в ```/Library/Keyboard Layouts```. Для этого в Finder нажимаем ```Сommand + Shift + G``` и переходим в нужный каталог.
3. Перезагружаемся 
4. В настройках Клавиатуры добавляем источники ввода "США (Флаг)" и "Русская (Флаг):
![Настройки клавиатуры](/img/flags-macos.png)

5. Вы великолепны!

![Снова флаг](/img/flags-macos3.png)