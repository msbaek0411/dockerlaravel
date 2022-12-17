# Docker-compose를 이용한 Milc 저작도구

현재 리포지토리는 docker-compose를 이용하여 laravel 환경을 구성하는 프로젝트입니다.

현재 nginx를 시스템에서 `8081`포트를 사용하고 있다는 가정하에 작성하였습니다.

## 디렉토리 구조
- .docker : php source
- mysql : mysql 초기 DB dump sql
- nginx 
  - logs : nginx logs
  - milc.conf : nginx default conf
- php 
  - Dockerfile : 개발환경에서 쓰이는 dockerfile
  - DockerfilePro : production 환경에서 쓰이는 dockerfile. 소스를 이미지 내에 copy하여 빌드할때 사용

## port

- php80-fpm : 9000
- nginx : 8081
- mysql : 3309

# docker-compose

## docker-compose.yml

php80-fpm, nginx, mysql을 설치하는 파일이다.

```bash
version: "3.3"

services:
  php:
    build:
      context: .
      dockerfile: ./php/Dockerfile
#    restart: always
    expose:
      - "9000"
    ports:
      - "9000:9000"
    links:
      - mysql:mysql
    depends_on:
      - mysql
    volumes:
       - ./.docker:/milc:z
#    command: [ "sh", "-c", "./build.sh" ]
  mysql:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: milc
      MYSQL_USER: bubblecon
      MYSQL_PASSWORD: Bubble1!
    volumes:
      - ./mysql/1gram20220211.sql:/docker-entrypoint-initdb.d/milc.sql
    ports:
      - 3309:3306
    expose:
      - 3309
  nginx:
    image: nginx:latest
    restart: always
    ports:
      - "8081:80"
    depends_on:
      - php
    volumes:
      - ./nginx/milc.conf:/etc/nginx/conf.d/default.conf
      - ./.docker:/milc:z
      - ./nginx/logs:/var/log/nginx
```

### docker-compose 실행 파일
```bash
# 시작 명령어
$ ./build.sh

# 재 빌드 명령어
$ ./rebuild.sh
```

# nginx

nginx는 시스템에서 `80`를 사용한다는 가정하에 `8081 → 80`으로 포트포워딩을 시키는 구조로 구성하였습니다.

### milc.conf

```bash
server {
    listen      80;
    server_name www.milc.test admin.milc.test;

    root "/milc/public";
    index index.html index.htm index.php;
    charset utf-8;


    location / {
        add_header 'Access-Control-Allow-Origin' "*";
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, DELETE, PUT';
        add_header 'Access-Control-Allow-Credentials' 'true';
        add_header 'Access-Control-Allow-Headers' 'User-Agent,Keep-Alive,Content-Type';
        add_header 'Access-Control-Max-Age' 1728000;
        add_header Access-Control-Allow-Headers "X-Requested-With, X-Prototype-Version, X-CSRF-Token, x-csrftoken, Origin, Accept, Content-Type, Access-Control-Request-Method, Access-Control-Request-Headers, Authorization";
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    access_log /var/log/nginx/1gram.app-access.log;
    error_log  /var/log/nginx/1gram.app-error.log error;

    sendfile off;

    client_max_body_size 0;

    location /storage/image/ {
        alias /nas/;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;

        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
    }

    location ~ /\.ht {
        deny all;
    }

}

```

# docker-milc 설치 구성법

## 개발환경 docker-milc

```bash
# git clone으로 프로젝트 구성
$ git http://repo.bubblecon.io/bubblecon/milc-docker.git

# node package 구성
$ cd .docker
$ npm i && npm run dev

# docker-compose 실행 

$ cd ..

# docker-compose.yml 내의 주석 해제. command: [ "sh", "-c", "./build.sh" ] 부분
$ ./build.sh

# docker-compose.yml 내의 다시 주석. command: [ "sh", "-c", "./build.sh" ] 부분
$ ./rebuild.sh
```
! **주의 사항** !
`docker-compose.yml` 파일 내에 `command: [ "sh", "-c", "./build.sh" ]` 주석되어 있는 부분을 제거 한 후 `./build.sh`을 실행해 주어야 합니다.
주석 풀고 `build.sh` 실행 시 php 컨테이너는 꺼지게되며 `rebuild.sh`을 이용하여 다시 실행해 주어야합니다.

`docker-compose` 내에서 `composer install`을 하게 되면 `php-fpm` 컨테이너가 종료되게 됩니다. 해당 부분은 패치가 필요합니다.


## swagger ui 설치
docker 콘테이너 내에서 실행이 필요합니다.

```bash
# docker를 이용하여 php80-fpm 컨테이너 접속
$ docker exec -it docker-milc_php_1 bash

# artisan을 이용한 provider 생성
$ php artisan vendor:publish --provider "L5Swagger\L5SwaggerServiceProvider"

# artisan을 이용한 swagger 생성
$ php artisan l5-swagger:generate
```

## Laravel Stroage Permission 설정

laravel은 웹에서 접근할 수 있도록 `stoage`경로에 `php.ini`에 설정된 유저로 권한을 변경해주어야 합니다.

해당 프로젝트는 `php.ini`를 따로 수정하지 않았기 때문에 유저를 `www-data`로 변경해주어야 합니다.

```bash
$ chown -R www-data:www-data storage
```
