---
layout: post
title: Telegram бот в доспехах или разработка ТГ бота с нуля.
disqus: true
category: Dev
tags: [python, flask, telegram-sdk, bot, nginx, https]
---

Всем мир! Сегодня я вам на практике покажу, как с 0 написать несложного Telegram бота. 

Для начала придумаем тестовое задание. 

У меня есть сервер Counter Strike 1.6 на котором я временами играю с друзьями. Заходить в игру, чтобы проверить наличие там игроков, лень. Напишем бота, который будет доставать для нас информацию о сервере и список игроков со счетом и выводить в удобном формате.  

![Telegram logo]({{ site.baseurl }}/images/telegram-bot.png)

*Для того, чтобы проследить за действиями, достаточно базовых знаний linux, web и python.*

## Постановка задачи

Для работы телеграм бот, который крутится на нашей машине, должен получать обновления от сервера Telegram. Это можно сделать двумя методами:
* Long polling. Метод, когда ваша программа в определенный промежуток времени опрашивает сервер об обновлениях.
* Web hook. Тут вы поднимаете веб сервер, на который телеграм бот шлет обновления, если они есть.

Второй метод популярнее и считается надежнее, поэтому остановимся на нем.

Писать будем на Python с использованием Flask. В роли веб сервера используем nginx. В работе с сервером CS нам поможет чудесный модуль [python-valve](https://github.com/Holiverh/python-valve/)

Выбор сервера для хостинга нашего бота роли не играет, я буду работать с Digital Ocean.

Сам бот будет довольно простым: одна кнопка Обновить, нажатие на которую, как и любое сообщение, будет обновлять информацию о карте и игроках.

## Регистрация бота

Первым делом нам надо зарегистрировать своего бота у [@BotFather](https://t.me/BotFather). Он отвечает за управление информацией о боте и получение и обновление токенов для работы с ними. От него мы получим такую строку 

![BotFather]({{ site.baseurl }}/images/botfather.png)

```
"Use this token to access the HTTP API:
344036403:AAE0JMeFjn97OiaTLzJy_oGKC4zEWuQuSYQ"
```
Это ключ по которому сервера телеграм будут знать, что вы владельцы этого бота. Если вы его потеряете, вы можете получить новый у того же @BotFather.

Первый шаг сделали. Мы уже сейчас можем найти нашего бота в ТГ и писать ему, но это пока только оболочка и отвечать вам не будет. Ему нужен код/призрак.

## Аренда выделенного сервера на DO

Как и любой сайт и серверсайд программа, наш код должен быть запущен на машине с белым ip и выходом в интернет. Для такого случая мы арендуем машину у [Ditial Ocean](digitalocean.com)

Показывать пошаговую регистрацию на сайте я не буду, покажу только как получить и настроить сам сервер. Кстати, если вы студент, можете попробовать получить 50$ на хостинг бесплатно.

![Digital ocean]({{ site.baseurl }}/images/do1.png)

Нажимаем на большую копку Create droplet.

![Digital ocean]({{ site.baseurl }}/images/do2.png)

Выбираем желаемую ОС. Я буду работать с Ubuntu. 

Дальше можно выбрать самый дешевый сервер за 5 баксов в месяц или 0.007 в час.
С выбором региона чуть тяжелее. Для телеграм бота особой разницы нет, но если будете хостить игры тут, следует подумать о пингах. Тут можно сравнивать состояние от вас, до разных регионов DO.
[http://speedtest-sfo1.digitalocean.com/](http://speedtest-sfo1.digitalocean.com/)

Больше нам ничего менять тут не надо, нажимаем Create.

В течении пары минут мы получим ip адрес и root пароль от сервера на нашу почту. Все, машина работает. Можем коннектиться к ней по ssh.

```shell
ssh root@%ваш_ip_адрес
```

Вводим пароль и получаем полный контроль над сервером. Возможно вас попросят сменить пароль, поставьте посложнее.

Первым делом нам сразу надо создать нового пользователя и входить на сервер из под него, чтобы не стать легкой мишенью для хакеров.

```shell
adduser gorec # Добавляем пользователя. Вводим данные, которые нас запросит система
sudo usermod -aG sudo gorec # Даем юзеру возможность исполнять команды от имени админа
```
Теперь необходимо перезайти на сервер под новым пользователем

```shell
ssh gorec@ваш_ip_адрес
```

## Установка нужных сервисов

```shell
sudo apt update
sudo apt install python-pip python-dev nginx
sudo service nginx start
```
Первая команда обновит локальные репозитории, чтобы не было проблем с поиском нужных пакетов. Вторая поставит PIP (систему управления пакетами python), веб сервер nginx и dev пакет python. Третья команда запустит веб сервер.

Если вы сейчас введете ip адрес вашего сервера в браузере любого компьютера подключенного к сети интернет, вы получите страницу приветствия nginx.
![Nginx hello world]({{ site.baseurl }}/images/nginx.png)

Выключим на пока наш веб сервер и займемся python.

```shell
sudo service nginx stop
```

## Python и virtualenv

По хорошему, нам надо создавать окружения для каждого из наших python проектов, чтобы их зависимости не начали конфликтовать между собой. Для этого воспользуемся virtualenv.

```shell
sudo pip install virtualenv
```

Создаем папку проекта в домашней и стартуем в ней новое изолированное окружение.

```shell
cd ~
mkdir CSBot
cd CSBot
virtualenv csbot
source csbot/bin/activate
```
Теперь у нас есть локальная копия python, pip и все python модули установленные внутри останутся в этом окружении. 

## Устанавливем Flask и UWSGI

Бота будем писать на микрофреймворке flask, который позволит нам сократит время на разработку. Для перенаправления запросов с nginx на flask будем пользоваться uwsgi.

```shell
pip install uwsgi flask
```

Для начала запустим простую страничку в Flask и попробуем достучаться к ней через nginx.

В папке с проектом создадим наш главный файл в котором будет хранится код.

```python
# file - ~/CSBot/csbot.py
from flask import Flask # Импортируем модули
app = Flask(__name__) # Создаем приложение

@app.route("/") # Говорим Flask, что за этот адрес отвечает эта функция
def hello_world():
    return "It's working"
```

Так же напишем скрипт, который будет запускать наше Flask приложение.

```python
# file - ~/CSBot/wsgi.py
from csbot import app # Импортируем наше приложение

if __name__ == "__main__":
    app.run() # запускаем его
```
Оба файла находятся в папке с проектом.

Запускам uwsgi командой:

```shell
cd ~/CSBot
uwsgi --socket 0.0.0.0:5000 --protocol=http -w wsgi:app
```
Если все прошло успешно, вы, перейдя в своем браузере по адресу вашего сервера и порт 5000 должны увидеть It's working.

![Uwsgi]({{ site.baseurl }}/images/uwsgi.png)

Теперь напишем конфиг файл, чтобы Flask приложение поднималось само, даже если сервер перезагрузится.
Сначала выйдем из нашего окружения.

```shell
deactivate
```

Потом создадим .ini конфиг для uwsgi в папке с ботом по пути```~/CSBot/csbot.ini```.
Конфиг задает количество процессов, имя сокета, права и файл для логирования.

```shell
[uwsgi]
module = wsgi:app

master = true
processes = 5

socket = csbot.sock
chmod-socket = 660
vacuum = true

die-on-term = true
logto = /var/log/uwsgi/%n.log
```

Так же создадим папку для логов и сделаем ее владельцем себя.

```shell
sudo mkdir /var/log/uwsgi
sudo chown -R gorec:gorec /var/log/uwsgi
```
Создадим так же systemd unit файл, для атвоматизации запуска нашего бота.


```shell
sudo nano /etc/systemd/system/csbot.service
```

и сам файл

```shell
[Unit]
Description=Serving our csbot
After=network.target

[Service]
User=gorec
Group=www-data
WorkingDirectory=/home/gorec/CSBot
Environment="PATH=/home/gorec/CSBot/csbot/bin"
ExecStart=/home/gorec/CSBot/csbot/bin/uwsgi --ini csbot.ini
```

Теперь после ввода команды, мы должны увидеть сокет файл в нашей папке проекта.

```shell
sudo service csbot start
sudo systemctl enable csbot
```

Мы так же можем проверить работу с помощью просмотра логов в папке и статуса сервиса.

```shell
sudo service csbot status
less /var/log/uwsgi/csbot.log
```

## Настройка nginx

Дальше нам необходимо настроить веб сервер, чтобы свои запросы он направлял в сокет uwsgi.

```shell
sudo nano /etc/nginx/sites-available/csbot
```

```shell
server {
    listen 80;
    server_name ваш_айпи;

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/gorec/CSBot/csbot.sock;
    }
}

```

Теперь нам надо сделать ссылку на конфиг, в папке sites-enabled

```shell
sudo ln -s /etc/nginx/sites-available/csbot /etc/nginx/sites-enabled
```
Мы можем проверить наши конфиг файлы с помощью nginx.

```shell
sudo nginx -t
```

Если он сказал, что все ОК, перезапускаем веб сервер.

```shell
sudo service nginx restart
```

Теперь по ip адресу нашего сервера в браузере вы должны получить выдачу Flask приложения.

## Включаем https

Телеграм бот не станет работать по обычному протоколу http т.к. это ставит под угрозу переписку. У нас нет домена на сервере, будем работать с самоподписным сертификатом.

Сначала генерируем наш сертификат и ключ с помощью openssl.

```shell
cd /etc/ssl/
sudo mkdir gorec
cd gorec
sudo openssl genrsa -des3 -passout pass:xxxx -out server.pass.key 2048
sudo openssl rsa -passin pass:xxxx -in server.pass.key -out server.key
rm server.pass.key
sudo openssl req -new -key server.key -out server.csr
```

На последней команде openssl задаст вам несколько вопросов. Можно игнорировать все, кроме Common Name (FQDN). В нем следует указать ip вашего сервера.

Следующей командой мы получим сертификат из сгененрированного выше приватного ключа.

```shell
cd /etc/ssl/gorec
sudo openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt
```

Так-с, серты сгенерировали. Идем к настройке nginx. 

Открываем конфиги нашего сайта в nginx:

```shell
sudo nano /etc/nginx/sites-available/csbot
```
Указвыаем новый порт взамен 80. Указвыаем ключи и сертификаты и протоколы, которые мы можем хендлить. 

```shell
server {
    listen 443 default ssl;
    server_name ваш_ип;
    keepalive_timeout   60;
    ssl_certificate /etc/ssl/server.crt;
    ssl_certificate_key  /etc/ssl/server.key;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers  "HIGH:!RC4:!aNULL:!MD5:!kEDH";
    add_header Strict-Transport-Security 'max-age=604800';
    access_log /var/log/nginx_access.log;
    error_log /var/log/ngingx_error.log;
    location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/gorec/CSBot/csbot.sock;

    }
}

```

Проверяем конфиги и перезапускаем веб сервер

```shell
sudo nginx -t
sudo service nginx restart
```

Теперь в браузере по https://ваш_ип, вы должны увидеть вашу страницу flask приложения. Браузер будет ругаться на сертификат, игнорируйте.

## Пишем код для бота

Дошли до самого интересного.

Сначала установим модуль для работы с Telegram - python-telegram-bot

```shell
cd ~/CSBot
source csbot/bin/activate
sudo pip install python-telegram-bot
```
Теперь напишем код бота (```~/CSBot/csbot.py```), который всегда отвечает словом hello на любое наше сообщение.

```python
# -*- coding: utf-8 -*-
from __future__ import unicode_literals 
from flask import Flask, request
import telegram

app = Flask(__name__)
app.debug = True

TOKEN = "ваш_токен"

global bot 
bot = telegram.Bot(token=TOKEN)

URL = 'ваш_ip' 

#WebHook
@app.route('/HOOK', methods=['POST', 'GET']) 
def webhook_handler():
    if request.method == "POST": 
        update = telegram.Update.de_json(request.get_json(force=True), bot)
        try:
            chat_id = update.message.chat.id 
            text = update.message.text
            userid = update.message.from_user.id
            username = update.message.from_user.username
            bot.send_message(chat_id=chat_id, text="hello")
        except Exception, e:
            print e
    return 'ok' 

#Set_webhook 
@app.route('/set_webhook', methods=['GET', 'POST']) 
def set_webhook(): 
    s = bot.setWebhook('https://%s:443/HOOK' % URL, certificate=open('/etc/ssl/server.crt', 'rb')) 
    if s:
        print(s)
        return "webhook setup ok" 
    else: 
        return "webhook setup failed" 

@app.route('/') 
def index(): 
    return '<h1>Hello</h1>' 

```

Разберем код.

В начале мы импортируем нужные модули flask, request, telegram.
Дальше мы создаем наше приложение, включаем режим отладки (надо присвоить False, когда закончим разработку).

После мы задаем наш token полученный при создании бота в самом начале, адрес сервера и создаем бота.

У нас есть два главных метода:

*set_webhook - отвечает за создание вебхука с сервером Telegram и вызывается один раз.

*webhook_handler - слушает на /HOOK и занимается обработкой сообщений с серверов телеграма.

В хендлере мы получаем данные в Post запросе, достаем нужную информацию и исходя из этого делаем какие-то действия и шлем сообщение юзеру методом bot.send_message.

Перезапускаем бота и заходим на https://ваш_ип/set_webhook. Если в ответе вы получили webhook setup ok, можно пойти тестить бота. 

![Bot start]({{ site.baseurl }}/images/bot.png)

Давайте чуть усложним его и добавим кнопку "Обновить".

Для начала импортируем ReplyKeyboardMarkup для создания клавиатуры и после отправим его в сообщении параметром reply_markup.


```python
# -*- coding: utf-8 -*-
from __future__ import unicode_literals 
from flask import Flask, request
from telegram.replykeyboardmarkup import ReplyKeyboardMarkup
import telegram

app = Flask(__name__)
app.debug = True

TOKEN = "ваш_токен"

global bot 
bot = telegram.Bot(token=TOKEN)

URL = 'ваш_ip' 

#WebHook
@app.route('/HOOK', methods=['POST', 'GET']) 
def webhook_handler():
    if request.method == "POST": 
        update = telegram.Update.de_json(request.get_json(force=True), bot)
        try:
        	kb = ReplyKeyboardMarkup([["Обновить"]])
            chat_id = update.message.chat.id 
            text = update.message.text
            userid = update.message.from_user.id
            username = update.message.from_user.username
            bot.send_message(chat_id=chat_id, text="hello", reply_markup=kb)
        except Exception, e:
            print e
    return 'ok' 

#Set_webhook 
@app.route('/set_webhook', methods=['GET', 'POST']) 
def set_webhook(): 
    s = bot.setWebhook('https://%s:443/HOOK' % URL, certificate=open('/etc/ssl/server.crt', 'rb')) 
    if s:
        print(s)
        return "webhook setup ok" 
    else: 
        return "webhook setup failed" 

@app.route('/') 
def index(): 
    return '<h1>Hello</h1>' 

```

![Bot start]({{ site.baseurl }}/images/bot2.png)

## Немного функционала

Напишем функцию, которая будет опрашивать наш CS сервер и выдавать информацию по нему.

Для начала установим наш модуль для работы с valve серверами. Не забудьте перейти в окружение перед установкой. 

```shell
cd ~/CSBot
source csbot/bin/activate
pip install python-valve
```

Теперь в коде определим новую функцию, которая будет генерить строку ответ и возвращать ее нам, для отправки пользователю.

```shell
server_address = (адрес_сервера, 27015)
server = valve.source.a2s.ServerQuerier(server_address)

def get_info():
    info = server.get_info()
    players = server.get_players()

    answer = "Players: {player_count}/{max_players} \nServer name: {server_name}\nGame: {game}\n".format(**info)
    for player in sorted(players["players"],
                         key=lambda p: p["score"], reverse=True):
        answer += "{name} {score} Dur: {duration}\n".format(**player)

    return answer
```


Код сам по себе не сложен, но давайте разберем.
Сначала задаем адрес и порт сервера и создаем объект.

В функции достаем общую информацию и инфу об игроках. Заносим данные в строку answer и проходимся по массиву пользователей добавляя в новую строку данные о солдатах с количеством очков и временем проведенным на сервере. Возвращаем строку ответ. 
Названия полей (player_count, max_players и остальные можно посмотреть в коде модуля на github https://github.com/Holiverh/python-valve/blob/master/valve/source/a2s.py)

Осталось отправлять полученную строку сообщение пользователю. Для этого правим строку bot.send_message

```python
...
bot.send_message(chat_id=chat_id, text=get_info(), reply_markup=kb)
...
```

Перезапускаем нашего бота.

```shell
sudo service csbot restart
```

![Bot showing info]({{ site.baseurl }}/images/csbot.png)

## Логи

В случае ошибок мы можем узнать что не так благодаря двум логам:
* Логи веб сервера находятся в /var/log/nginx/
* Логи нашего приложения и uwsgi в /var/log/uwsgi/

Код бота можно найти тут: [https://github.com/mrZizik/CSBot](https://github.com/mrZizik/CSBot) \\
Написать ему можно тут: [@csinfobot](https://t.me/csinfobot)