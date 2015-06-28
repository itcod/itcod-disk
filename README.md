# itcod-disk
SOA ITCOD
WEBDAV CLOUD Service for SOA ITCOD

Сервис "ITCOD-DISK" Облачное хранилище.


-- Copyright (c) 2015 by Yura Vdovytchenko (max@itcod.com)
-- Copyright (c)itcod 2010-2015
-- version: 15.06.27
-- license: MIT


Назначение: Сетевой диск(хранилище) файлов по технологии  WEBDAV. 
С публикацией по http/https и индексный файл с контрольными суммами md5/etc.
Предназначен для хранения и публикации NoSQL информационных массивов.

Принцип: Сервис-ориентированная архитектура построения. Nginx обеспечивает 
стандартный протокол WEBDAV over HTTP/HTTPS. Lua-модули itcod обеспечивают 
расширение функций и сетевые сервисы управления ITCOD-DISK'ом.
ITCOD UI WWII обеспечивает WEB-интерфейс между пользователем и сервисами.

ОТЛИЧИЕ ОТ АНАЛОГОВ
NoLAMP NoLEMP NoSQL
На сервере только Nginx + Lua и никаких PHP SQL и т.д.

БАЗОВЫЕ КОМПОНЕНТЫ ITCOD-DISK

LINUX - операционная система
NGINX - http daemod (with WebDAV and Lua)
LUA - язык программирования
Resty - библиотека Lua
add - дополнительные библиотеки (см. require в *.lua)
ITCOD Lua Modules & Services - модули SOA ITCOD для операций с хранилищем
ITCOD WWII - web-интерфейс для ITCOD-DISK (в разработке)

БАЗОВЫЕ КОМПОНЕНТЫ ITCOD Lua Modules & Services

auth-dav.lua - авторизатор для HTTP/HTTPS/WEBDAV
md5index.lua - расширитель функций autoindex NGINX
itcod-user.lua - создание пользовательских юзербоксов на диске WEBDAV
itcod-exchange.lua - сервис транспорта файлов между пользователями и дисками
itcod-search.lua - REST-сервис авторизованного поиска информации в закрытых пользовательских массивах
libs/ - библиотека иконок типов файлов для md5index

Подробнее о компанентах см. https://ihome.itcod.com/max/project/


КОНФИГУРАЦИЯ NGINX

nginx version: nginx/1.7.11
built by gcc 4.4.4 20100630 (Red Hat 4.4.4-10) (GCC) 
TLS SNI support enabled
configure arguments: 
--prefix=/usr/share/nginx 
--sbin-path=/usr/sbin/nginx 
--conf-path=/etc/nginx/nginx.conf 
--error-log-path=/var/log/nginx/error.log 
--http-log-path=/var/log/nginx/access.log 
--http-client-body-temp-path=/var/lib/nginx/tmp/client_body 
--http-proxy-temp-path=/var/lib/nginx/tmp/proxy 
--http-fastcgi-temp-path=/var/lib/nginx/tmp/fastcgi 
--http-uwsgi-temp-path=/var/lib/nginx/tmp/uwsgi 
--http-scgi-temp-path=/var/lib/nginx/tmp/scgi 
--pid-path=/run/nginx.pid 
--lock-path=/run/lock/subsys/nginx 
--user=nginx 
--group=nginx 
--with-pcre-jit 
--with-debug 
--with-file-aio 
--with-ipv6 
--with-http_ssl_module 
--with-http_realip_module 
--with-http_addition_module 
--with-http_xslt_module 
--with-http_image_filter_module 
--with-http_geoip_module 
--with-http_sub_module 
--with-http_dav_module 
--add-module=/usr/src/nginx-dav-ext-module-master 
--with-http_flv_module 
--add-module=/usr/src/f4f-hds-master 
--with-http_mp4_module 
--with-http_gzip_static_module 
--with-http_random_index_module 
--with-http_secure_link_module 
--with-http_degradation_module 
--with-http_stub_status_module 
--with-http_perl_module --with-mail 
--with-mail_ssl_module 
--with-http_auth_request_module 
--add-module=/usr/src/echo-nginx-module-master 
--add-module=/usr/src/nginx_md5_filter-master 
--add-module=/usr/src/ngx_devel_kit-master 
--add-module=/usr/src/lua-nginx-module-master 
--with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector 
--param=ssp-buffer-size=4 -m64 -mtune=generic' 
--with-ld-opt=' -Wl,-E,-rpath,/usr/local/lib'


КОНФИГУРАЦИЯ ВИРТУАЛЬНОГО WEB-СЕРВЕРА (WEBDAV)

Приведена конфигурация nginx работающего на виртуальной машине 
за проксирующим первичным nginx. Для работы на первичном вам необходимо
изменить listen на 80 и 443. А так же не забудьте поправить основные 
настройки на ваши собственные (имена сервера и т.д.)

Файл ihome.conf

server {
    listen       7070;
    server_name "~^ihome\d+\.itcod\.com$"
		ihome.virtual.ko
		ihome.itcod.com
		;
    server_name_in_redirect	off;
    expires	epoch;
    ssl                  off;
    #default_type application/octet-stream;
    set_real_ip_from 10.255.255.7;
    real_ip_header	X-Forwarded-For;
    real_ip_recursive on;
    access_log /var/log/nginx/ihome.itcod.com-access.log main;
    resolver 10.255.255.1 [::1]:5353;
    charset utf-8;
    
    set $dir /opt/home;
    set $testdir $dir$uri;
    set $uri_type none;
    if (-d $testdir) { # такая папка есть
	set $uri_type dir;
	rewrite ^(.*)$ $1/;
	rewrite ^(.*)/+$ $1/;
    }
    if (-f $testdir) { # такой файл есть
	set $uri_type file;
    }
    if ($request_method = "MKCOL") {
	rewrite ^(.*)$ $1/;
	rewrite ^(.*)/+$ $1/;
	set $uri_type dir; #клиент webdav создает папку
    }
    if ($request_method = "PUT") {
	set $uri_type file; #передаем только файлы
    }
    if ($request_method = "POST") { 
	set $uri_type file; #постим только файлы
    }
    set $sadm_passwd .uhtpsw;
    set $user_passwd .htpasswd; #user:password[crypt(3)/md5/sha1]
    set $user_permit .htpermit; #user:GET,PUT,....OPTIONS
    set $user_permit_default GET,PROPFIND,OPTIONS; # Allow

    merge_slashes on;
    
    location / {
        limit_req	zone=itcod	burst=200 nodelay;
        limit_rate	2048k;
	access_by_lua_file /etc/nginx/lua/auth-dav.lua;
	dav_methods PUT DELETE MKCOL COPY MOVE;
	dav_ext_methods PROPFIND OPTIONS;
	create_full_put_path on;
	dav_access user:rw group:rw;
	client_body_temp_path /opt/itcod-dav.tmp/;
	client_max_body_size 0;
	autoindex on;
        root $dir;
        header_filter_by_lua_file /etc/nginx/lua/itcod-exchange.lua;
	set $md5index on; #on/off nil=off # вкл/выкл обработчик
	set $md5index_hash md5; #none/md5/md4/sha1/sha/ripemd160 nil=none # тип выводых хэшей
	set $md5index_size 50000; #kb nil=unlimit # не считать для файлов более N kb
	set $md5index_path on; #on/off nil=off  # заменять относительный путь ссылок на полный URI
	set $md5index_nonblank on; #on/off nil=off # заменить множественные пробелы одним
	set $md5index_type on; #on/off nil=off # добавит в строки описание типа file/directory/etc...
	set $md5index_ico http://ihome.itcod.com/max/projects/libs/icons16ext/; # путь к библиотека иконок
	set $md5index_icopref icon-; # префикс имени файла иконки
	#set $md5index_icosuf -icon; # суфикс имени файла иконки
	set $md5index_icoext .gif; # расширение файла иконки
	set $md5index_win _blank; # target window for !winext! files
	set $md5index_winext htm.html.txt; # file extension for target windows
        body_filter_by_lua_file /etc/nginx/lua/md5index.lua; # addon обработчик
        
    }
    location ~/\.uht {
	deny all;
    }
    location /search/ {
	content_by_lua_file /etc/nginx/lua/itcod-search.lua;
    }
    location /user/ {
	content_by_lua_file /etc/nginx/lua/itcod-user.lua;
    }
}


ПРИМЕЧАНИЕ

Для программистов адекватных perl, проблем определить и загрузить недостающие 
модули require не составит труда. В случае если у вас, что то не получается 
пишите тут или на max@itcod.com обязательно помогу.


ТЕКУЩИЕ РАБОТЫ

Формирование WebUI ITCOD-DISK
