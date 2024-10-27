## Администрирование Nginx/Angie

- [Домашнее задание №1](#домашнее-задание-1)
- [Домашнее задание №2](#домашнее-задание-2)

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
