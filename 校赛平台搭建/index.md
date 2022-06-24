# 校赛CTFd平台docker搭建—pixo主题


## 前言

（使用github图床，请科学上网）

搭建校赛平台，这里是发现有一个很好看的ctfd的主题[pixo](https://github.com/hmrserver/CTFd-theme-pixo)

所以就想着来搭建这么一个平台，这次我打算使用docker来搭建，使用docker搭建有两条路可走

- docker pull ctfd/ctfd
- docker-compose up

## 第一种方法的尝试

首先我是打算直接使用docker pull的方式来搭建的，的确可以搭建，但是主页会有广告

![image-20220419025252948](https://raw.githubusercontent.com/huamang/image/master/image-20220419025252948.png)

我想着怎么把他删掉，我把文件通过docker传回服务器

```jsx
docker cp xx:/opt/CTFd ./
```

然后下载到本机进行搜索，发现他存在于views.py文件里面

![image-20220419025306044](https://raw.githubusercontent.com/huamang/image/master/image-20220419025306044.png)

我对这个文件进行了审计，发现他会在setup的时候把内容写进html内，但是这里就有问题了，我如果是pull下来的镜像，我运行的时候他就已经启动了views.py，所以我无法修改里面的广告信息。所以我无奈之下只能放弃这个方法，我开始尝试docker-compose来进行搭建

## docker-compose

我从github下载到CTFd3.4.3版本的文件，我找到docker-compose.yml文件，修改了端口号一后，我就开始执行`docker-compose up` 

我观察dockerfile，这里会进行apt的更新和下载

![image-20220419025319014](https://raw.githubusercontent.com/huamang/image/master/image-20220419025319014.png)

但是发现apt会下载的十分缓慢，等了很久很久都下不完，于是我选择进行换源，我dockerfile进行如下修改，增加如下命令执行

```jsx
RUN  sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list
RUN  apt-get clean
RUN apt-get update
```

我空了一段时间打开，发现apt已经处理好了，但是pip出问题了，他下载超时了。。

![image-20220419025328678](https://raw.githubusercontent.com/huamang/image/master/image-20220419025328678.png)

![image-20220419025339382](https://raw.githubusercontent.com/huamang/image/master/image-20220419025339382.png)

所以我还得对pip进行换源，我又对dockerfile进行如下的修改，添加阿里云源

```jsx
RUN pip config set global.index-url http://mirrors.aliyun.com/pypi/simple
RUN pip config set install.trusted-host mirrors.aliyun.com
```

看见pip已经把所有模块安装完毕，就当我以为快要成功的时候，发现这里又有报错，说我的80端口被占用，那我不可能因为一个docker就把80端口给让出来吧，所以我修改了docker-compose.yml文件，将Nginx的端口转发到其他的端口

![image-20220419025349406](https://raw.githubusercontent.com/huamang/image/master/image-20220419025349406.png)

这下总算成功的搭建起来了

![image-20220419025359651](https://raw.githubusercontent.com/huamang/image/master/image-20220419025359651.png)

## END

附上这次dockerfile和docker-compose.yml

dockerfile

```docker
FROM python:3.7-slim-buster
WORKDIR /opt/CTFd
RUN mkdir -p /opt/CTFd /var/log/CTFd /var/uploads

# hadolint ignore=DL3008
RUN  sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list
RUN  apt-get clean
RUN apt-get update
RUN pip config set global.index-url http://mirrors.aliyun.com/pypi/simple
RUN pip config set install.trusted-host mirrors.aliyun.com
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        python3-dev \
        libffi-dev \
        libssl-dev \
        git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt /opt/CTFd/

RUN pip install -r requirements.txt --no-cache-dir

COPY . /opt/CTFd

# hadolint ignore=SC2086
RUN for d in CTFd/plugins/*; do \
        if [ -f "$d/requirements.txt" ]; then \
            pip install -r $d/requirements.txt --no-cache-dir; \
        fi; \
    done;

RUN adduser \
    --disabled-login \
    -u 1001 \
    --gecos "" \
    --shell /bin/bash \
    ctfd
RUN chmod +x /opt/CTFd/docker-entrypoint.sh \
    && chown -R 1001:1001 /opt/CTFd /var/log/CTFd /var/uploads

USER 1001
EXPOSE 8000
ENTRYPOINT ["/opt/CTFd/docker-entrypoint.sh"]
```

docker-compose.yml

```yaml
version: '2'

services:
  ctfd:
    build: .
    user: root
    restart: always
    ports:
      - "50010:8000"
    environment:
      - UPLOAD_FOLDER=/var/uploads
      - DATABASE_URL=mysql+pymysql://ctfd:ctfd@db/ctfd
      - REDIS_URL=redis://cache:6379
      - WORKERS=1
      - LOG_FOLDER=/var/log/CTFd
      - ACCESS_LOG=-
      - ERROR_LOG=-
      - REVERSE_PROXY=true
    volumes:
      - .data/CTFd/logs:/var/log/CTFd
      - .data/CTFd/uploads:/var/uploads
      - .:/opt/CTFd:ro
    depends_on:
      - db
    networks:
        default:
        internal:

  nginx:
    image: nginx:1.17
    restart: always
    volumes:
      - ./conf/nginx/http.conf:/etc/nginx/nginx.conf
    ports:
      - 50011:80
    depends_on:
      - ctfd

  db:
    image: mariadb:10.4.12
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=ctfd
      - MYSQL_USER=ctfd
      - MYSQL_PASSWORD=ctfd
      - MYSQL_DATABASE=ctfd
    volumes:
      - .data/mysql:/var/lib/mysql
    networks:
        internal:
    # This command is required to set important mariadb defaults
    command: [mysqld, --character-set-server=utf8mb4, --collation-server=utf8mb4_unicode_ci, --wait_timeout=28800, --log-warnings=0]

  cache:
    image: redis:4
    restart: always
    volumes:
    - .data/redis:/data
    networks:
        internal:

networks:
    default:
    internal:
        internal: true
```

