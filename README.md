## Администрирование Nginx/Angie

- [Домашнее задание №1](#домашнее-задание-1)
- [Домашнее задание №2](#домашнее-задание-2)
- [Домашнее задание №3](#домашнее-задание-3)

## Домашнее задание №1

### Установка Angie
#### Цель: Провести установку актуальной версии Angie на виртуальную машину различными способами.

Создадим виртуальную машину на Yandex Cloud. Установим Angie согласно [инструкции](https://angie.software/angie/docs/installation/oss_packages/#install-deb-oss).

```sh
asor@test1:~$ sudo curl -o /etc/apt/trusted.gpg.d/angie-signing.gpg https://angie.software/keys/angie-signing.gpg
asor@test1:~$ echo "deb https://download.angie.software/angie/$(. /etc/os-release && echo "$ID/$VERSION_ID $VERSION_CODENAME") main" \
    | sudo tee /etc/apt/sources.list.d/angie.list > /dev/null
asor@test1:~$ sudo apt-get update
asor@test1:~$ sudo apt-get install angie
```
Проверим работоспособность Angie с помощью браузера.

![screenshot 1](https://github.com/user-attachments/assets/db4d0ad7-2ad0-4082-bcb2-cbc070871bd8)

Проверим работу динамического модуля [Image Filter](https://angie.software/angie/docs/configuration/modules/http/http_image_filter/). Установим модуль.

```sh
asor@test1:~$ sudo apt install angie-module-image-filter
```

Модуль зависит от библиотеки `libgd`, которая будет установлена автоматически.
 
Скачаем рандомную картинку и положим её туда же, где лежат статические страницы.

```sh
asor@test1:~$ cd /usr/share/angie/html
asor@test1:/usr/share/angie/html$ sudo mkdir img && cd img
asor@test1:/usr/share/angie/html/img$ sudo wget https://www.google.com/images/branding/googlelogo/1x/googlelogo_color_272x92dp.png
```
Добавим следующие директивы конфигурации.

В `/etc/angie/angie.conf`:

```nginx

load_module modules/ngx_http_image_filter_module.so;

```

В `/etc/angie/http.d/default.conf`:

```nginx

server {
    location /img/ {
        root   /usr/share/angie/html;
        image_filter resize 150 100;  # изменим размер
        image_filter rotate 90;       # повернём изображение
    }
...
```

Перезагрузим конфигурацию Angie:

```sh
asor@test1:~$ sudo systemctl reload angie.service
```

Проверим работу модуля с помощью браузера.

![screenshot 2](https://github.com/user-attachments/assets/b87c927e-d6ed-4277-84e8-daceda3c8681)

Теперь запустим Angie из Docker-образа, согласно [инструкции](https://angie.software/angie/docs/installation/docker/). Выполняем на локальной машине. Файлы страниц и конфигурации находятся в текущей директории, контейнеру откроем к ним доступ на чтение. Порт 80 контейнера пробросим на порт 8080 хоста.

```sh
asor@desktop1:~/angie$ docker run --rm --name angie -v $(pwd):/usr/share/angie/html:ro \
    -v $(pwd)/angie.conf:/etc/angie/angie.conf:ro -p 8080:80 -d docker.angie.software/angie:1.7.0-alpine
```

Проверим работу Angie с помощью браузера с учётом того, что порт 80 контейнера переключен на порт 8080 хоста.

![screenshot 3](https://github.com/user-attachments/assets/0b6ea403-850b-4700-aac1-61e4f53f2c20)

Успешно произведена установка актуальной версии Angie различными способами.

## Домашнее задание №2

### Миграция из Nginx в Angie
#### Цель: Научиться переводить готовые конфигурации для Nginx в конфигурацию Angie.

На виртуальной машине Ubuntu 24.04 установлен Nginx. Копируем набор конфигов для Nginx в соответствующую директорию:

```sh
~$ sudo tar -xzvf nginx_conf.tar-252831-0242cd.gz -C /etc
```

Устанавливаем Angie согласно [инструкции](https://angie.software/angie/docs/installation/oss_packages/#install-deb-oss).

После установки Angie автоматически запущен. Останавливаем его:

```sh
~$ sudo systemctl stop angie
```

Копируем старый файл конфигурации `/etc/nginx/nginx.conf` в новый `/etc/angie/angie.conf`. Копируем файлы `static-avif.conf` и `static.conf` из `/etc/nginx` в новый `/etc/angie`. Копируем также директории `sites-enabled` и `sites-available` и их содержимое из `/etc/nginx` в `/etc/angie`. Создаём директорию `conf.d`. В файлах корректируем директории где необходимо:

```sh
~$ cd /etc/angie
/etc/angie$ sudo cp ../nginx/nginx.conf ./angie.conf
/etc/angie$ sudo cp ../nginx/{static-avif.conf,static.conf}  .
/etc/angie$ sudo cp -r ../nginx/{sites-enabled,sites-available,snippets}/ .
/etc/angie$ sudo mkdir conf.d
/etc/angie$ for f in angie.conf sites-available/* ; do sudo sed -i 's|/nginx|/angie|g' $f ; done
```

Меняем ссылки в директории `sites-enabled`:

```sh
/etc/angie$ cd sites-enabled
/etc/angie/sites-enabled$ for f in * ; do sudo ln -sf "/etc/angie/sites-available/$f" "$f" ; done
```

В Nginx присутствовал модуль `ngx_http_echo_module`. Устанавливаем соответствующий модуль Angie, а также модули `brotli`.

```sh
/etc/angie$ sudo apt install angie-module-echo angie-module-brotli
```

Меняем в `angie.conf` строку `include /etc/angie/modules-enabled/*.conf;` на `load_module modules/ngx_http_echo_module.so;`.

Для использования модулей `brotli` также добавляем директивы загрузки соответствующих модулей:

```nginx
load_module modules/ngx_http_brotli_filter_module.so;
load_module modules/ngx_http_brotli_static_module.so;
```

Проверяем работоспособность конфигурации:

```sh
/etc/angie$ sudo /usr/sbin/angie -t
angie: the configuration file /etc/angie/angie.conf syntax is ok
angie: configuration file /etc/angie/angie.conf test is successful
```

Успешно завершена миграция с Nginx на Angie. 

## Домашнее задание №3

### Запуск статического сайта
#### Цель: Создать базовую конфигурацию для работы статического сайта с разделением location.

Angie установлен на виртуальной машине с IP [158.160.74.39](http://158.160.74.39). Поскольку у нас нет доменного имени, мы будем обращаться к сайту по IP.

Копируем содержимое архива сайта `static-site.zip` в `/etc/angie/html`.

Создаём конфигурацию сайта. Копируем в `/etc/angie` файл `angie.conf` следующего содержания.

```nginx
# Количество рабочих процессов выбирается автоматически (по количеству ядер).
worker_processes auto;

events {
}

http {

    # Определение переменных с помощью директивы map. Задаём режим
    # кеширования статического контента с учётом браузера.

    map $msie $cache_control {

        # Ресурс актуален в течение 1 года, может быть закеширован в любом
        # кеше, не должны применяться никакие преобразования, не требует
        # обновлений.

        default "max-age=31536000, public, no-transform, immutable";

        # Для MS IE ответ предназначен для одного пользователя и не должен
        # помещаться в разделяемый кеш.

        "1"     "max-age=31536000, private, no-transform, immutable";
    }

    server {

        # Изначально сайт тестировался на localhost, но поскольку на виртуальной
        # машине Yandex Cloud у нас нет доменного имени, мы будем обращаться к
        # сайту по IP.

        #server_name localhost;

        # Редирект. При обращении к главной странице мы будем перенаправлены на порт 8000.

        listen 8000;

        error_page 404 /error/index.html;

        # location c регулярным выражением для отдачи картинок

        location ~* \.(ico|jpeg|jpg|png)$ {
            add_header Cache-Control $cache_control;
        }

        # location для каждой директории со своими настройками.
        # Поскольку в этих директориях находятся css и скрипты, задаём для них
        # режим кеширования аналогичный картинкам. Того же эффекта мы могли бы
        # добиться, добавив соответствующие расширения файлов в предыдущий
        # location.

        location /assets {
            add_header Cache-Control $cache_control;

            # Последовательный перебор путей обработки запроса. Эта директива
            # перенаправит пользователя на нашу страницу ошибки 404, если он
            # попытается зайти на .../assets/. Без неё он получит стандартную
            # страницу ошибки 403 Forbidden.

            try_files $uri =404;
        }

        location /error {
            add_header Cache-Control $cache_control;
            try_files $uri =404;
        }

        location /images {
            # Сюда мы попадём только если если формат картинки не перечислен
            # в location c регулярным выражением.

            add_header Cache-Control $cache_control;
            try_files $uri =404;
        }

        location / {
            # Сюда мы приходим по умолчанию, если никакой другой location
            # не подошёл.

            root   html;
            index  index.html;

            # Внутренний редирект. На именованный location @stub перейдём, если
            # отсутствует файл index.html.

            try_files /index.html @stub;
        }

        location @stub {
            return 200 "This is a stub";
        }
    }

    server {
        # Внешний редирект. Директива return перенаправит на домен, если
        # приходим по IP или без заголовка Host в запросе.

        listen 80 default_server;
        server_name '';
        return 302 http://158.160.74.39:8000$request_uri;
    }
}
```

Перезагружаем Angie:

```sh
root@test1:~# systemctl reload angie
```

Пробуем зайти на сайт через браузер:

![Screenshot 3](https://github.com/user-attachments/assets/1ef709d1-cbf8-46f0-8975-6b7ea73eb215)

Пробуем пройти по ссылкам:

![Screenshot 4](https://github.com/user-attachments/assets/e3f9777f-856b-4b7d-98d1-9bb721abe24e)

Кеш картинок работает в соответствии с настройками:

![Screenshot 5](https://github.com/user-attachments/assets/c06e655d-a222-484b-b460-f22b808de271)

Базовая конфигурация для работы статического сайта создана.
