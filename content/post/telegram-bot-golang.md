+++
Categories = ["golang"]
Description = "Как написать telegram бота на golang. Простой пример Telegram bot-а на языке Go с использованием Telegram API"
Tags = ["golang"]
date = "2017-05-25T21:05:17+03:00"
title = "Как написать Telegram бота на Golang"

+++
![Telegram bot Golang](/img/telegram_bot_golang.png)

Telegram - очень удобный мессенджер и у него есть отличный инструмент, который можно использовать в своих самых разных целях - боты. Bot - это аккаунт в телеграме, который управляется программой, а не человеком.

<!--more-->

В двух словах - создается аккаунт с определенным именем, пишется скрипт/программа которая следит за сообщениями, которые приходят в этот аккаунт и реагирует на них запрограммированным способом.

Здесь я покажу пример написания telegram-бота для мониторинга состояния сайтов на языке Go. Бот будет обходить список урлов, который указан в конфиге, и, если этот урл не открывается или отдает не HTTP/200, то писать о падении в определенный чат.

Этот бот я использую для мониторинга состояния production-сайтов, если сайт упал - бот напишет мне об этом в телеграм чат.

Приступим к созданию. Сначала надо создать аккаунт в телеграме. Любые действия с аккаунтами ботов осуществляются с помощью ... бота ```@BotFather```. Это бот, который управляет ботами. Чтобы создать нового - надо написать ему в личку команду ```/new_bot```. После этого BotFather задаст вопрос про имя для нового бота и выдаст token для авторизации в API:

![Создание бота в чате с @BotFather](/img/telegram_bot_golang1.png)


Все, теперь существует аккаунт ```@simplesitemonitoringbot```, который можно найти в поиске и под которым будет авторизовываться наш бот:

![Новый бот](/img/telegram_bot_golang2.png)

Начнем кодировать. Для работы с API есть библиотека [telegram-bot-api](http://github.com/Syfaro/telegram-bot-api) ([GoDoc](https://godoc.org/github.com/go-telegram-bot-api/telegram-bot-api)).  Код ниже можно брать за основу при создании своего бота:

{{< highlight go >}}

package main

import (
	"flag"
	"github.com/Syfaro/telegram-bot-api"
	"log"
	"os"
)

var (
	// глобальная переменная в которой храним токен
	telegramBotToken string
)

func init() {
	// принимаем на входе флаг -telegrambottoken
	flag.StringVar(&telegramBotToken, "telegrambottoken", "", "Telegram Bot Token")
	flag.Parse()

	// без него не запускаемся
	if telegramBotToken == "" {
		log.Print("-telegrambottoken is required")
		os.Exit(1)
	}
}

func main() {
	// используя токен создаем новый инстанс бота
	bot, err := tgbotapi.NewBotAPI(telegramBotToken)
	if err != nil {
		log.Panic(err)
	}

	log.Printf("Authorized on account %s", bot.Self.UserName)

	// u - структура с конфигом для получения апдейтов
	u := tgbotapi.NewUpdate(0)
	u.Timeout = 60

	// используя конфиг u создаем канал в который будут прилетать новые сообщения
	updates, err := bot.GetUpdatesChan(u)

	// в канал updates прилетают структуры типа Update
	// вычитываем их и обрабатываем
	for update := range updates {
		// универсальный ответ на любое сообщение
		reply := "Не знаю что сказать"
		if update.Message == nil {
			continue
		}

		// логируем от кого какое сообщение пришло
		log.Printf("[%s] %s", update.Message.From.UserName, update.Message.Text)

		// свитч на обработку комманд
		// комманда - сообщение, начинающееся с "/"
		switch update.Message.Command() {
		case "start":
			reply = "Привет. Я телеграм-бот"
		case "hello":
			reply = "world"
		}

		// создаем ответное сообщение
		msg := tgbotapi.NewMessage(update.Message.Chat.ID, reply)
		// отправляем
		bot.Send(msg)
	}
}


{{< /highlight >}}

Компилируем и пробуем запускать. 
{{< highlight console >}}
$ go build
$ ./telegram-site-monitoring -telegrambottoken 3972____:____________03WRDsIU
2017/05/24 19:13:59 Authorized on account simplesitemonitoringbot
2017/05/24 19:14:01 [f___] asdf
2017/05/24 19:14:05 [f___] /start
2017/05/24 19:14:42 [f___] /start
2017/05/24 19:14:46 [f___] /hello
2017/05/24 19:15:02 [f___] asdfasdf
{{< /highlight >}}

![Новый бот в работе](/img/telegram_bot_golang3.png)

Бот работает и отвечает до тех пор, пока работает программа. Если завершить программу и писать ему в оффлайн, то сообщения будут складываться в очередь на стороне телеграма, а когда приложение вновь запустится - придут все разом.

# Добавление функции мониторинга


Добавим функцию обхода урлов в списке и нотификации в определенный чат, если что-то из списка не отвечает. Редактировать список урлов можно через команды бота прямо в чате.

Для начала создадим новую группу и добавим туда бота. В эту группу можно добавлять людей, которые хотят получать уведомления от мониторинга.

![Чат для нотификаций](/img/telegram_bot_golang4.png)

Теперь надо узнать chatid этой группы. Способ не совсем тривиальный, но другого я не нашел:

1. Когда бот находится в группе, пишем ему какое-нибудь сообщение типа: ```/test @simplesitemonitoringbot```

2. Заходим на урл ```https://api.telegram.org/bot3972______:__________3WRDsIU/getUpdates```

3. В полученном json-е ищем chatid:
{{< highlight json>}}
....
"chat":{"id":-147193640,"title":"Site monitoring Chat","type":"group","all_members_are_administrators":true}
....
{{< /highlight >}}
В данном случае chatid = -147193640  . Да, он может быть отрицательным.


Теперь можно добавить функцию мониторинга в код. Здесь приведен не весь код. Полный код  у меня в github - https://github.com/fote/telegram-site-monitoring:
{{< highlight go >}}
var (
	// здесь храним список проверяемых сайтов и их код состояния
	// Возможные коды состояния
	// 0 - еще не проверялся
	// 1 - таймаут соединения
	// 200 - ОК
	// Все остальные HTTP коды - crit
	SiteList   map[string]int

	// chatid чата для нотификаций
	chatID     int64
	
	// файл в котором будем хранить список сайтов с состояниями и зачитывать их при старте
	// чтобы при рестарте приложения не потерять список
	configFile string

	telegramBotToken string
)

func init() {
	SiteList = make(map[string]int)
	

	flag.StringVar(&configFile, "config", "config.json", "config file")
	flag.StringVar(&telegramBotToken, "telegrambottoken", "", "Telegram Bot Token")
	flag.Int64Var(&chatID, "chatid", 0, "chatId to send messages")

	flag.Parse()

	if telegramBotToken == "" {
		log.Print("-telegrambottoken is required")
		os.Exit(1)
	}

	if chatID == 0 {
		log.Print("-chatid is required")
		os.Exit(1)
	}

	// при старте зачитываем список из файла если он существует и сохраняем
	// в глобальную мапку SiteList
	// если файла не существует, стартуем с пустым списком
	load_list()
}

func send_notifications(bot *tgbotapi.BotAPI) {
	// обходим мапку с сайтами
	// если статус не 200 - шлем нотификацию в чат мониторинга
	for site, status := range SiteList {
		if status != 200 {
			alarm := fmt.Sprintf("CRIT - %s ; status: %v", site, status)
			bot.Send(tgbotapi.NewMessage(chatID, alarm))
		}
	}
}

func monitor(bot *tgbotapi.BotAPI) {
	// параметр для http клиента чтобы принимал все ssl сертификаты
	tr := &http.Transport{
		TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
	}

	// важно указать таймаут http соединения, иначе он вечно будет висеть и мы не увидим
	// проблемы если сервер просто упал
	var httpclient = &http.Client{
		Timeout: time.Second * 10,
		Transport: tr,
	}

	// в вечном цикле обходим список урлов раз в 5 мин и сохраняем статус в глобальный map SiteList 
	for {
		for site, _ := range SiteList {
			response, err := httpclient.Get(site)
			if err != nil {
				SiteList[site] = 1
				log.Printf("Status of %s: %s", site, "1 - Cannot connect to server")
			} else {
				log.Printf("Status of %s: %s", site, response.Status)
				SiteList[site] = response.StatusCode
			}
		}

		// шлем нотификации
		send_notifications(bot)
		// сохраняем текущий статус SiteList в файл configFile
		save_list()
		time.Sleep(time.Minute * 5)
	}
}

func main() {
	// создаем инстанс бота
	bot, err := tgbotapi.NewBotAPI(telegramBotToken)
	if err != nil {
		log.Panic(err)
	}

	log.Printf("Authorized on account %s", bot.Self.UserName)
	log.Printf("Config file: %s", configFile)
	log.Printf("ChatID: %v", chatID)
	log.Printf("Starting monitoring thread")

	// в отдельном потоке запускаем функцию мониторинга
	go monitor(bot)

	u := tgbotapi.NewUpdate(0)
	u.Timeout = 60

	bot.Send(tgbotapi.NewMessage(chatID, fmt.Sprint("Я живой; вот сайты которые буду мониторить: ", SiteList)))

	updates, err := bot.GetUpdatesChan(u)

	// здесь вычитываем присланные команды и реагируем на них
	for update := range updates {
		reply := ""
		if update.Message == nil {
			continue
		}

		log.Printf("[%s] %s", update.Message.From.UserName, update.Message.Text)

		switch update.Message.Command() {
		// выводим список сайтов в мониторинге
		case "site_list":
			sl, _ := json.Marshal(SiteList)
			reply = string(sl)

		// добавить урл в список мониторинга и присвоить статус 0 (еще не проверялся)
		case "site_add":
			SiteList[update.Message.CommandArguments()] = 0
			reply = "Site added to monitoring list"

		// удалить из списка
		case "site_del":
			delete(SiteList, update.Message.CommandArguments())
			reply = "Site deleted from monitoring list"
		case "help":
			reply = HelpMsg
		}

		msg := tgbotapi.NewMessage(update.Message.Chat.ID, reply)
		bot.Send(msg)
	}

{{< /highlight >}}

Компилируем и тестируем.

{{< highlight console >}}
$ ./telegram-site-monitoring -telegrambottoken 39720____:_______________3WRDsIU -chatid -147193640
2017/05/25 17:32:41 No such file - starting without config
2017/05/25 17:32:42 Authorized on account simplesitemonitoringbot
2017/05/25 17:32:42 Config file: config.json
2017/05/25 17:32:42 ChatID: -147193640
2017/05/25 17:32:42 Starting monitoring thread
2017/05/25 17:32:50 [f___] /site_add https://4te.me
2017/05/25 17:32:58 [f___] /site_add http://doesntexistsite.ru
2017/05/25 17:33:00 [f___] /site_list
2017/05/25 17:37:42 Status of https://4te.me: 200 OK
2017/05/25 17:37:42 Status of http://doesntexistsite.ru: 1 - Cannot connect to server
2017/05/25 17:42:43 Status of http://doesntexistsite.ru: 1 - Cannot connect to server
2017/05/25 17:42:43 Status of https://4te.me: 200 OK
{{< /highlight >}}

![Тест мониторинга](/img/telegram_bot_golang5.png)

Все работает! Можно запускать на сервере демоном или собрать docker-контейнер (https://github.com/fote/telegram-site-monitoring/blob/master/Dockerfile) и запускать с помощью docker-а и следить за падениями важных сайтов:
{{< highlight console>}}
# mkdir /opt/tsm
# docker run -d --restart=always --name=tsm -v /opt/tsm:/etc/tsm fote/tsm -telegrambottoken 3972_____:______________3WRDsIU -chatid -147193640
{{< /highlight >}}


Длинных аптаймов!
