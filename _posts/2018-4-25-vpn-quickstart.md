---
layout: post
title: Запустить за 60 секунд. Экспресс OpenVPN
disqus: true
category: Admin
tags: [vpn, openvpn, telegram, блокировки, admin,]
---

Учитывая, что каждый день рандомно падают десятки сервисов, иметь VPN становится необходимостью. В интернете есть предложения от компаний, предлагающих услугу под ключ за разумные деньги и с хорошей производительностью, если вы совсем ничего не хотите делать. Например https://tgvpn.com. Для среднестатистического юзера это неплохой выбор по цена/качество, однако мы тут собрались, чтобы сделать все своими руками. 

В этом посте я опишу, как быстро поднять свой VPN сервер и подключиться к нему. Let the hacking begin (c).

### 1. Арендуем сервер.

Подробно вдаваться в аренду VPS сервера не буду, т.к. данный вопрос уже раскрывался в статье про создание Telegram бота. (http://abdulmadzhidov.ru/blog//Telegram-bot-in-30-min/) Другие хостинги в плане алгоритма не отличаются от Digital Ocean.

### 2. Устанавливаем нужные пакеты

Первым делом  поставим утилиты, которые нужны для работы apt через https. Возможно они уже будут на вашей машине и тогда команда сообщит вам об этом. Сначала мы обновим список пакетов, которые доступны в репозиториях. Второй командой установим нужные программы.

```shell
sudo apt update

sudo apt install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```

Я предпочитаю поднимать сервисы в докер контейнерах, так что далее установим docker.

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo apt-key fingerprint 0EBFCD88
```

Командами выше достаем ключи для проверки подписи на тех пакетах, что мы будем ставить из официального репозитория с докером и удостоверяемся, что они успешно загрузились. 

Если все хорошо, мы должны получить подобный ответ с информацией о том, кем выдан ключ, его отпечатком...

```shell
pub   4096R/0EBFCD88 2017-02-22
      Key fingerprint = 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid                  Docker Release (CE deb) <docker@docker.com>
sub   4096R/F273FCD8 2017-02-22
```

Добавляем сам репозиторий и снова обновляем список пакетов.

```shell
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt update
``` 

После этого можем установить docker.

```shell
sudo apt install docker-ce
```

Для проверки, что все прошло успешно, мы можем запустить тестовый контейнер, который поприветствует нас

```shell
sudo docker run hello-world
```

Установку пакетов закончили. 

P.S. Иногда бывают проблемы, что докер доступен только от рута. Для решения создаем группу docker и добавляем своего пользователя в нее.

```sh
sudo groupadd docker
sudo usermod -aG docker $USER
```
После этого релогинимся на машину под своей учеткой.

### 3. Настраиваем наш OpenVPN

Создаем папку, в которую будут сложены наши конфиги и ключи

```sh
mdkir ~/openvpn_data
```

и запускаем докер контейнер с openvpn с командой genconfig.

```shell
docker run -v ~/opevpn_data:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://<ip-адрес-вашей-машины-здесь>

docker run -v ~/openvpn_data:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
```

Вы можете ввести пароль для защиты приватного ключа. Поле Common Name можно оставить пустым. 

На этом моменте у нас в папке ~/openvpn_data должны хранится конфиги для нашего VPN сервера.

### 4. Запускаем сервер

```sh
docker run -v ~/openvpn_data:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn
```

Готово, сервер запущен и работает. Осталось сгенерировать конфигурационные файлы и сертификаты для клиентов.

### 5. Создаем клиентов

```sh
docker run -v ~/openvpn_data:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full <имя_клиента> nopass

docker run -v ~/openvpn_data:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient <имя_клиента> > <имя_клиента>.ovpn
```


Первая команда создаст сертификат для клиента с заданным именем, а вторая сгенерирует конфигурационный файл .ovpn, с влкюченным туда сертификатом. В процессе вас попросят ввести ваш пароль, который мы задали выше. 

### 6. Подключение клиентов

В итоге в папке ~/openvpn_data мы получаем .ovpn файл для каждого клиента, который необходимо отправить уже на устройство. 

Это можно сделать разными путями, но я просто поднимал в папке openvpn_data веб сервер на питоне и скачивал файлы с клиентской машины.

```sh
cd ~/openvpn_data

python -m SimpleHTTPServer
```

Пока команда запущеня, все содержимое папки будет доступно на порту 8000.

Заходим с клиента на http://<ip_адрес_вашей_машины>:8000 и скачиваем нужный .ovpn файл.

Подключение зависит от OpenVPN клиента, которым вы пользуетесь. На MacOS есть TunnelBlink с удобным  интерфейсом. На Ubuntu я предпочитаю консольный openvpn клиент.

```sh
sudo apt install openvpn

sudo openvpn --config <путь_до_конфиг_файла> --daemon
```

На этом все.


