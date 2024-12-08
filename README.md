## Администрирование Nginx/Angie

- [Домашнее задание №1](#домашнее-задание-1)
- [Домашнее задание №2](#домашнее-задание-2)
- [Домашнее задание №3](#домашнее-задание-3)
  - [Работа над ошибками №1](#работа-над-ошибками-1)
- [Домашнее задание №4](#домашнее-задание-4)
- [Домашнее задание №5](#домашнее-задание-5)
- [Домашнее задание №6](#домашнее-задание-6)

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

Angie установлен на виртуальной машине с IP [51.250.97.82](http://51.250.97.82). Поскольку у нас нет доменного имени, мы будем обращаться к сайту по IP.

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
        return 302 http://51.250.97.82:8000$request_uri;
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

### Работа над ошибками №1

В ходе проверки было выявлено некорректное отображение сайта в связи с тем, что был задан неверный MIME-type стиля CSS.

![css-error](https://github.com/user-attachments/assets/d67e2f43-ee1b-4945-8257-d7ef1a366033)

В блок `http` файла конфигурации были добавлены следующие директивы:

```nginx
    include       /etc/angie/mime.types;
    default_type  application/octet-stream;
```

После чего сайт стал отображаться корректно:

![css-fixed](https://github.com/user-attachments/assets/76186aff-54d3-47af-925b-7be0f937823f)


*Примечание: Поскольку виртуальная машина была сконфигурирована как [прерываемая](https://yandex.cloud/ru/docs/compute/concepts/preemptible-vm), её IP изменился на [158.160.74.39](http://158.160.74.39/).*

## Домашнее задание №4

### Запуск сайта с CMS Wordpress
#### Цель: Запустить рабочую систему управления сайтами (CMS). Изучить на практике Angie/Nginx как обратный прокси.

Angie изнчально установлен на виртуальной машине с IP [84.252.137.218](http://84.252.137.218). Поскольку у нас нет доменного имени, мы будем обращаться к сайту по IP.

Для работы Wordpress устанавливаем сервер MySQL и PHP с необходимыми расширениями:

```sh
asor@test1:~$ sudo apt install mysql-server-8.0
asor@test1:~$ sudo apt install php-fpm php-curl php-mysqli php-gd php-intl php-mbstring php-soap php-xml php-xmlrpc php-zip
```

Создаём БД и настраиваем MySQL. Аутентификация с использованием плагина `mysql_native_password` [не рекоммендуется](https://mariadb.com/kb/en/authentication-plugin-mysql_native_password/) для систем с высоким уровнем безопасности, но применима для наших целей.

```sh
asor@test1:~$ sudo mysql
mysql> CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '1234';
mysql> CREATE USER 'wordpress'@'%' IDENTIFIED WITH mysql_native_password BY '1234';
mysql> GRANT ALL ON wordpress.* TO 'wordpress'@'%';
```

Скачиваем и устанавливаем Wordpress:

```sh
asor@test1:~$ cd /tmp
asor@test1:/tmp$ curl -LO https://wordpress.org/latest.tar.gz
asor@test1:/tmp$ tar -xzf latest.tar.gz
asor@test1:/tmp$ cp wordpress/{wp-config-sample.php,wp-config.php}
asor@test1:/tmp$ sudo mkdir /var/www
asor@test1:/tmp$ sudo cp -a wordpress/. /var/www/wordpress
asor@test1:/tmp$ sudo chown -R www-data:www-data /var/www/wordpress
```

Редактируем файл `/var/www/wordpress/wp-config.php` в соответствии с настройками БД:

```php
// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** Database username */
define( 'DB_USER', 'wordpress' );

/** Database password */
define( 'DB_PASSWORD', '1234' );

/** Database hostname */
define( 'DB_HOST', 'localhost' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

/**#@+
 * Authentication unique keys and salts.
 *
 * Change these to different unique phrases! You can generate these using
 * the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}.
 *
 * You can change these at any point in time to invalidate all existing cookies.
 * This will force all users to have to log in again.
 *
 * @since 2.6.0
 */
define('AUTH_KEY',         '<P`yGe{e-%+h;qLnt-nhT|5&/BCOn#O|b?>V#IgQn7CyK~`MmsYgczJm;cf@gsJQ');
define('SECURE_AUTH_KEY',  'Fj Qnr`c]/_4w+>^sIK!h3+x`;L{>Vv(Ui$9Tf-sw->)cb)zWJ]u:uIy/hS?}jd1');
define('LOGGED_IN_KEY',    'H]-iJKa/~L/3o=4>5BO+!nhYu]+(/3Hu+/ {G|l`HH[Mppl3D)!+zPG#(z|k[2H+');
define('NONCE_KEY',        '^tInSs&ki&W8D3srIg8I8MAM6DC88&|$Z3bEl|jtKoWaKlchO*w_!`0`[1DVWhtm');
define('AUTH_SALT',        '?aE,OCQV+ClbrodpB&bihbF}*-[meV7sr3s7JdtLe}|P5hWm30aA5^V=Ct`Xj&-/');
define('SECURE_AUTH_SALT', '3$.c)a93cI3%8`gT-@Ed}tB+i#JcK.01SG+|pzCB(ZL7XOF7W+zj;vnRE$Luay*k');
define('LOGGED_IN_SALT',   's$k7:3,p`XmE.`vhzi4(,elviRDis D(_{XHu @yL8?p}QmJFcy,B}J:$V-b{{=%');
define('NONCE_SALT',       '-r0a+^qn9Ay4c(kieQH8&[f8}!g68|a2Nvo,9Ol;OUSSNlO:4-@jn)ZRj#-gWnsB');
```

Создаём конфигурацию Angie. Файл `/etc/angie/angie.conf`:

```nginx
user  www-data;

worker_processes  auto;

events {
}

http {
    include       /etc/angie/mime.types;
    default_type  application/octet-stream;

    sendfile           on;
    keepalive_timeout  65;

    include /etc/angie/http.d/*.conf;
}
```

Файл `/etc/angie/http.d/wordpress.conf`:

```nginx
server {
    # Пока не настроен TLS, редиректим на порт 8000
    # Сайт будет работать пока на этом порту.
    listen 80 default_server;
    return 302 http://$host:8000$request_uri;
}

server {
    listen  8000;

    charset utf-8;

    root /var/www/wordpress;
    index index.html index.php;

    # Запрещаем доступ к некоторым важным директориям.

    location ~ /\. {
        deny all;
    }

    location ~ ^/wp-content/cache {
        deny all;
    }

    location ~* /(?:uploads|files)/.*\.php$ {
        deny all;
    }

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # Кешируем статический контент.

    location /wp-content {
        add_header Cache-Control "max-age=31536000, public, no-transform, immutable";
    }

    location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
        add_header Cache-Control "max-age=31536000, public, no-transform, immutable";

        log_not_found off;
    }

    # Не логгируем ошибки доступа к некоторым малозначительным ресурсам.

    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    location = /robots.txt {
        log_not_found off;
        access_log off;
        allow all;
    }

    location ~ \.php$ {
        # Проксируем запросы к PHP-скриптам.

        include fastcgi.conf;
        fastcgi_intercept_errors on;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
    }
}
```

Перезапускаем Angie.
```sh
asor@test1:~$ sudo systemctl reload angie.service
```

Проверяем доступность сайта. Wordpress готов к установке:

![wp-install](https://github.com/user-attachments/assets/a25c5cc7-6c48-4513-9d14-7bb12bceb4ca)

Wordpress успешно установлен:

![wp-admin](https://github.com/user-attachments/assets/6d3d138e-03d6-498e-a130-5b62260ce916)

## Домашнее задание №5

### Оптимизация WordPress
#### Цель: Оптимизировать конфигурацию сайта WordPress. Использовать приёмы клиентской и серверной оптимизации.

Angie изнчально установлен на виртуальной машине с IP [51.250.108.79](http://51.250.108.79). Поскольку у нас нет доменного имени, мы будем обращаться к сайту по IP.

С помощью сервиса [WebPageTest](https://www.webpagetest.org/) было выполнено предварительное тестирования сайта. Получены следующие обобщённые результаты:

![Performance test 1](https://github.com/user-attachments/assets/3ad47389-35eb-466f-8d5c-cfa6e68d7fb6)

При создании конфигурации Angie уже были предусмотрены некоторые меры по оптимизации. В частности, клиентское кеширование статического контента:

```nginx
    location /wp-content {
        add_header Cache-Control "max-age=31536000, public, no-transform, immutable";
    }

    location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
        add_header Cache-Control "max-age=31536000, public, no-transform, immutable";

        log_not_found off;
    }
```

Поскольку на данный момент на сайте мало контента, небольшле количество CSS, скриптов и пр., трудно выбрать методы оптимизации, которые окажутся наиболее эффективными. Возможно, оптимизация лишь незначительныо улучшит показатели. Попробуем применить компрессию и серверное кеширование.

Добавим компрессию текстовых ресурсов:

```nginx
gzip on;
gzip_types text/plain text/css text/xml application/javascript application/json image/svg+xml font/woff2;
gzip_comp_level 6;
gzip_proxied any;
gzip_min_length 1000;
gzip_vary on;
```

Добавим серверное кеширование. Поскольку мы кешируем контент не от прокси-сервера, а передаваемый "движком" PHP по протоколу FastCGI, вместо директив `proxy_cache_...` используем соответствующие дирекстивы `fastcgi_cache_...`:

```nginx
fastcgi_cache_valid 1m;
fastcgi_cache_key $scheme$host$request_uri;
fastcgi_cache_path /wp_cache levels=1:2 keys_zone=wp_cache:10m inactive=48h max_size=800m;

# ...

server {
  # ...
    location ~ \.php$ {
         
        # ...
        
        fastcgi_cache wp_cache;
        fastcgi_cache_valid 200 1h;
        fastcgi_cache_lock on;
        fastcgi_cache_min_uses 2;
        fastcgi_ignore_headers "Cache-Control" "Expires";
        fastcgi_cache_use_stale updating error timeout invalid_header http_500 http_503;
        fastcgi_cache_background_update on;
# ...
```

Перезапускаем Angie.
```sh
asor@test1:~$ sudo systemctl reload angie.service
```

Повторяем тестирование с помощью WebPageTest. Результаты тестирования после оптимизации:

![Performance test 2](https://github.com/user-attachments/assets/179a3506-07b0-4a61-ad9f-e884943b0a11)

![Скриншот сайта](https://github.com/user-attachments/assets/b464c21b-478d-464a-a9b8-8c323a831200)

Как и предполагалось, оптимизация лишь незначительныо улучшила показатели. В частности, на несколько миллисекунд улучшилась скорость загрузки контента и время до первого байта, тогда как другие показатели остались без изменения, а показатель CPU Time даже ухудшился на 1 мс.

Оптимизация сайта произведена, однако лучших показателей можно добиться при большем количестве контента.

## Домашнее задание №6

### Настройка HTTPS
#### Цель: Настроить эффективную и безопасную конфигурацию для HTTPS.

У меня имеется в распоряжении доменное имя [sor0.ru](http://sor0.ru). Установим на нём сайт, использовавшися в [ДЗ №3](#домашнее-задание-3). Создадим следующую конфигурацию Angie:

```nginx
worker_processes auto;

events {
}

http {

    # Настроим получение сертификатов от Let's Encrypt. Для большей
    # совместимости будем использовать сертификаты двух типов - RSA и ECDSA.
    # Для этого настроим два ACME-клиента.

    acme_client sor0_rsa https://acme-v02.api.letsencrypt.org/directory key_type=rsa key_bits=2048;

    acme_client sor0_ecdsa https://acme-v02.api.letsencrypt.org/directory key_type=ecdsa key_bits=256;

    # адрес резолвера в нашей системе

    resolver 127.0.0.53 ipv6=off;

    # необходимые общие настройки

    root          html;
    index         index.html;
    error_page    404 /error/index.html;
    include       mime.types;
    default_type  application/octet-stream;
    sendfile      on;

    # Оптимизация. Задаём режим кеширования статического контента
    # с учётом браузера.

    map $msie $cache_control {

        # Ресурс актуален в течение 1 года, может быть закеширован в любом
        # кеше, не должны применяться никакие преобразования, не требует
        # обновлений.

        default "max-age=31536000, public, no-transform, immutable";

        # Для MS IE ответ предназначен для одного пользователя и не должен
        # помещаться в разделяемый кеш.

        "1"     "max-age=31536000, private, no-transform, immutable";
    }

    # Основной сервер

    server {

        listen               443 ssl reuseport;
        server_name          sor0.ru www.sor0.ru;

        # Используем HTTP/2

        http2 on;

        # Устанавливаем HTTPS по умолчанию.
        # в порядке эксперимента установим небольшое значение max-age для HSTS
        add_header Strict-Transport-Security "max-age=600" always;

        # Оптимизация SSL

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305;

        # Используем полученные от Let's Encrypt сертификаты
        # и соответствующие им ключи.

        ssl_certificate      $acme_cert_sor0_rsa;
        ssl_certificate_key  $acme_cert_key_sor0_rsa;

        ssl_certificate      $acme_cert_sor0_ecdsa;
        ssl_certificate_key  $acme_cert_key_sor0_ecdsa;

        # Кешируем SSL-сессии

        ssl_session_cache shared:ssl_cache:4m;
        ssl_session_timeout 28h;

        # На стороне клиента:

        ssl_session_tickets on;
        ssl_early_data      on;

        acme                 sor0_rsa;
        acme                 sor0_ecdsa;

        # Оптимизация. Кеширование картинок, css и скриптов

        location ~* \.(ico|jpeg|jpg|png)$ {
            add_header Cache-Control $cache_control;
        }

        location /assets {
            add_header Cache-Control $cache_control;
            try_files $uri =404;
        }

        location /error {
            add_header Cache-Control $cache_control;
            try_files $uri =404;
        }

        location /images {
            add_header Cache-Control $cache_control;
            try_files $uri =404;
        }

        location / {
            try_files /index.html =404;
        }
    }

    server {
        # Редирект с HTTP на HTTPS.

        listen               80 default_server;
        server_name          sor0.ru www.sor0.ru;

        return 302 https://sor0.ru$request_uri;
    }
}
```
