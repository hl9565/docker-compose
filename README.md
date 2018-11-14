# mysite的docker-compose.yml文件
-------------------------------------------------------------------------
    ├── mysite
    │   ├── account
    │   ├── mysite
    │   ├── dailyblog
    │   ├── Dockerfile
    │   ├── gunicorn.conf
    │   ├── manage.py
    │   ├── media
    │   ├── requirements.txt
    │   ├── start.sh
    │   └── static
    ├── docker-compose.yml
    └── nginx
        ├── Dockerfile
        └── nginx.conf
-------------------------------------------------------------------------
Dockerfile
----
    FROM ubuntu:16.04
    
    # 安装系统需要的库
    RUN apt update
    RUN apt install -y python3-pip libmysqlclient-dev
    
    # 设置工作目录
    RUN mkdir /mysite
    WORKDIR /mysite
    
    #将当前目录下的文件加入到工作目录中
    ADD . /mysite
    
    # 安装运行python3需要的库
    RUN pip3 install --upgrade pip -i https://pypi.tuna.tsinghua.edu.cn/simple/
    RUN pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple/
    
    #对外暴露端口
    EXPOSE 80 8080 8000 5000
    
    #设置环境变量
    ENV SPIDER=/mysite

----------------------------------

start.sh
--------
    #!/bin/bash
    #命令只执行最后一个,所以用 &&
    
    python3 manage.py collectstatic --noinput &&
    python3 manage.py migrate &&
    python3 manage.py shell -c "from django.contrib.auth.models import User; User.objects.create_superuser('admin', 'hl95599@aliyun.com', 'hl95599')" &&
    gunicorn mysite.wsgi:application -c gunicorn.conf 

----------------------------------

gunicorn.conf
--------
    workers=4
    bind=['0.0.0.0:8000']
    proc_name='mysite'
    pidfile='/tmp/mysite.pid'
    worker_class='gevent'
    max_requests=6000

----------------------------------

Dockfile
--------
    FROM nginx
    
    #对外暴露端口
    EXPOSE 80 8000
    
    RUN rm /etc/nginx/conf.d/default.conf
    
    ADD nginx.conf  /etc/nginx/conf.d/
    
    RUN mkdir -p /usr/share/nginx/html/static
    RUN mkdir -p /usr/share/nginx/html/media
------------------------------------------------------------------------- 
nginx.conf
--------
    server {
        listen      80;
        server_name 0.0.0.0;
        charset     utf-8;
    
        error_log /tmp/nginx_error.log;
        access_log /tmp/nginx_access.log;
    
    
        location /media {
            alias /usr/share/nginx/html/media;
        }
    
        location /static {
            alias /usr/share/nginx/html/static;
            }
    
        location / {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_redirect off;
            proxy_pass http://web:8000;
        }
    
    }
-------------------------------------------------------------------------

docker-compose.yml
--------------
    version: "3"
    
    services:
    
      db:
        image: mysql
        environment:
           MYSQL_DATABASE: mysite
           MYSQL_ROOT_PASSWORD: hl95599
        volumes:
          - /data/webroot/db:/var/lib/mysql
        restart: always
    
      redis:
        image: redis
        restart: always
    
      memcached:
        image: memcached
        restart: always
        ports:
          - "11211:11211"
        entrypoint:
          - memcached
          - -m 64
      web:
        build: ./mysite
        ports:
          - "8000:8000"
        volumes:
          - /data/webroot/static:/data/webroot/static
          - /data/webroot/media:/data/webroot/media
          - /data/webroot/tmp/logs:/tmp
        command: bash start.sh
        links:
          - redis
          - memcached
          - db
        depends_on:
          - db
        restart: always
    
      nginx:
        build: ./nginx
        ports:
          - "80:80"
        volumes:
          - /data/webroot/static:/usr/share/nginx/html/static:ro
          - /data/webroot/media:/usr/share/nginx/html/media:ro
        links:
          - web
        depends_on:
          - web
        restart: always
-----------------------------------------------------
    docker-compose build
    docker-compose up -d 
