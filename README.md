# Laravel + Docker 環境構築ハンズオン
* 3層アーキテクチャのコンテナの構築
* ウェブサーバー(web)  
 nginxで静的コンテンツ配信サーバを構築  
* アプリケーションサーバー(app)  
 nginxを経由してPHPを動作させるアプリケーションサーバを構築   
 PHPパッケージ管理ツールComposerのインストール  
* データベースサーバー(db)  
 MySQLデータベースサーバーの構築  
* Laravelをインストールしてwelcome画面の表示
* LaravelとMySQLを連携し、マイグレーションを実行

# docker, docker-compose のインストール
[Mac] https://docs.docker.com/docker-for-mac/install  
[Windows] https://docs.docker.com/docker-for-windows/install  

```
[mac] $ docker --version
Docker version 19.03.8, build afacb8b

[mac] $ docker-compose --version
docker-compose version 1.25.5, build 8a1c60f6
```

# 最終的なディレクトリ構成
```
.
├── README.md
├── docker
│   ├── mysql
│   │   └── my.cnf
│   ├── nginx
│   │   └── default.conf
│   └── php
│       ├── Dockerfile
│       └── php.ini
├── docker-compose.yml
├── logs
│   ├── access.log
│   ├── error.log
│   ├── mysql-error.log
│   ├── mysql-query.log
│   ├── mysql-slow.log
│   └── php-error.log
└── src
    └── readme.md
```
# 手順
## 作業ディレクトリを作成
```
[mac] $ mkdir docker-laravel-handson
[mac] $ cd docker-laravel-handson
```

## 環境ファイル(.env)を作成
```
[mac] $ touch .env
```

[.env]
```
DB_NAME=homestead
DB_USER=homestead
DB_PASS=secret
TZ=Asia/Tokyo
```

## アプリケーションサーバ(app)コンテナを作成
```
[mac] $ touch docker-compose.yml
```

[touch docker-compose.yml]
```
version: "3"
services:
  app:
    build:
      context: ./docker/php
      args:
        - TZ=${TZ}
    volumes:
      - ./src:/work
      - ./logs:/var/log/php
      - ./docker/php/php.ini:/usr/local/etc/php/php.ini
    working_dir: /work
    environment:
      - DB_CONNECTION=mysql
      - DB_HOST=db
      - DB_DATABASE=${DB_NAME}
      - DB_USERNAME=${DB_USER}
      - DB_PASSWORD=${DB_PASS}
      - TZ=${TZ}
```

```
[mac] $ mkdir -p docker/php
[mac] $ touch docker/php/Dockerfile
```

[Dockerfile]
```
FROM php:7.3-fpm-alpine
LABEL maintainer "your-name"

ARG TZ
ENV COMPOSER_ALLOW_SUPERUSER 1
ENV COMPOSER_HOME /composer

RUN set -eux && \
  apk add --update-cache --no-cache --virtual=.build-dependencies tzdata && \
  cp /usr/share/zoneinfo/${TZ} /etc/localtime && \
  apk del .build-dependencies && \
  docker-php-ext-install bcmath pdo_mysql && \
  curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin --filename=composer && \
  composer config -g repos.packagist composer https://packagist.jp && \
  composer global require hirak/prestissimo
```

```
[mac] $ touch docker/php/php.ini
```

[php.ini]
```
error_reporting = E_ERROR | E_WARNING | E_PARSE | E_NOTICE
display_errors = stdout
display_startup_errors = on
log_errors = on
error_log = /var/log/php/php-error.log
upload_max_filesize = -1
memory_limit = -1
post_max_size = 100M
max_execution_time = 900
max_input_vars = 100000
default_charset = UTF-8

[Date]
date.timezone = ${TZ}

[mbstring]
mbstring.language = Japanese
```

build & up
```
[mac] $ docker-compose up -d --build
[mac] $ docker-compose ps
            Name                          Command              State    Ports  
-------------------------------------------------------------------------------
docker-laravel-handson_app_1   docker-php-entrypoint php-fpm   Up      9000/tcp

```

## ウェブサーバー(web)コンテナを作成
[docker-compose.yml]
```
web:
    image: nginx:1.17-alpine
    depends_on:
      - app
    ports:
      - 10080:80
    volumes:
      - ./src:/work
      - ./logs:/var/log/nginx
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    environment:
      - TZ=${TZ}
```
app コンテナの設定と同じインデントレベルにする

```
[mac] $ mkdir docker/nginx
[mac] $ touch docker/nginx/default.conf
```

[default.conf]
```
server {
    listen 80;
    root /work/public;
    index index.php;
    charset utf-8;

    location / {
        root /work/public;
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

build & up
```
[mac] $ docker-compose down
[mac] $ docker-compose up -d --build
[mac] $ docker-compose ps
            Name                          Command              State           Ports        
--------------------------------------------------------------------------------------------
docker-laravel-handson_app_1   docker-php-entrypoint php-fpm   Up      9000/tcp             
docker-laravel-handson_web_1   nginx -g daemon off;            Up      0.0.0.0:10080->80/tcp
```

nginxのバージョン確認
```
[mac] $ docker-compose exec web nginx -v
nginx version: nginx/1.17.10
```

webコンテナの確認
```
[mac] $ mkdir src/public
[mac] $ echo "Hello World" > src/public/index.html
[mac] $ echo "<?php phpinfo();" > src/public/phpinfo.php
```

「Hello World」が表示されることを確認する http://127.0.0.1:10080/index.html  
phpinfoの情報が表示されることを確認する http://127.0.0.1:10080/phpinfo.php  

確認用に作成したHTML, PHPファイルを削除
```
[mac] $ rm -rf src/*
```

## Laravelをインストール
```
[mac] $ docker-compose exec app ash
[app] $ composer create-project --prefer-dist "laravel/laravel=6.0.*" .
[app] $ php artisan -V
Laravel Framework 6.0.4

[app] $ exit
```
Laravel ウェルカム画面の表示 http://127.0.0.1:10080

## データベース(db)コンテナを作成
[docker-compose.yml]
```
db:
    image: mysql:8.0
    volumes:
      - db-store:/var/lib/mysql
      - ./logs:/var/log/mysql
      - ./docker/mysql/my.cnf:/etc/mysql/conf.d/my.cnf
    environment:
      - MYSQL_DATABASE=${DB_NAME}
      - MYSQL_USER=${DB_USER}
      - MYSQL_PASSWORD=${DB_PASS}
      - MYSQL_ROOT_PASSWORD=${DB_PASS}
      - TZ=${TZ}

volumes:
  db-store:
```
db コンテナの設定は web コンテナの設定と同じインデントレベルにする  
volumes はトップレベル(servicesと同じレベル)にする  

```
[mac] $ mkdir docker/mysql
[mac] $ touch docker/mysql/my.cnf
```

[my.cnf]
```
[mysqld]
# character set / collation
character-set-server = utf8mb4
collation-server = utf8mb4_bin

# timezone
default-time-zone = SYSTEM
log_timestamps = SYSTEM

# MySQL8 caching_sha2_password to mysql_native_password
default-authentication-plugin = mysql_native_password

# Error Log
log-error = /var/log/mysql/mysql-error.log

# Slow Query Log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 5.0
log_queries_not_using_indexes = 0

# General Log
general_log = 1
general_log_file = /var/log/mysql/mysql-query.log

[mysql]
default-character-set = utf8mb4

[client]
default-character-set = utf8mb4
```

build & up
```
[mac] $ docker-compose down
[mac] $ docker-compose up -d --build
[mac] $ docker-compose ps
            Name                          Command              State           Ports        
--------------------------------------------------------------------------------------------
docker-laravel-handson_app_1   docker-php-entrypoint php-fpm   Up      9000/tcp             
docker-laravel-handson_db_1    docker-entrypoint.sh mysqld     Up      3306/tcp, 33060/tcp  
docker-laravel-handson_web_1   nginx -g daemon off;            Up      0.0.0.0:10080->80/tcp
```

mysqlのバージョン確認
```
[mac] $ docker-compose exec db mysql -V
mysql  Ver 8.0.17 for Linux on x86_64 (MySQL Community Server - GPL)
```

## マイグレーション実行
```
[mac] $ docker-compose exec app ash
[app] $ php artisan migrate

Migrating: 2014_10_12_000000_create_users_table
Migrated:  2014_10_12_000000_create_users_table (0.05 seconds)
Migrating: 2014_10_12_100000_create_password_resets_table
Migrated:  2014_10_12_100000_create_password_resets_table (0.02 seconds)
Migrating: 2019_08_19_000000_create_failed_jobs_table
Migrated:  2019_08_19_000000_create_failed_jobs_table (0.01 seconds)

[app] $ exit
```
