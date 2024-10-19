# Домашнее задание №1

## Установка Angie
### Цель: Провести установку актуальной версии Angie на виртуальную машину различными способами.

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

Проверим работу динамического модуля [Image Filter](https://angie.software/angie/docs/configuration/modules/http/http_image_filter/). Установим модуль. Модуль зависит от библиотеки `libgd`, которая будет установлена автоматически.

```sh
asor@test1:~$ sudo apt install angie-module-image-filter
```

Скачаем рандомную картинку и положим её туда же, где лежат статические страницы.

```sh
asor@test1:~$ asor@test1:~$ cd /usr/share/angie/html
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
