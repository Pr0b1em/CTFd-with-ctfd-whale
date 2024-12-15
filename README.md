# CTFd-with-ctfd-whale
打包好了直接git clone，然后使用
## 1.配置Ubuntu Docker 环境

* 先配置 环境 然后启动，后续再添加其他功能

### 1.1安装 docker

```shell
apt install docker
apt install docker-compose
```
### 1.2初始化集群

```python
sudo docker swarm init --force-new-cluster
sudo docker node update --label-add='name=linux-1' $(sudo docker node ls -q)
```
### 1.3尝试启动


* 建议启动前执行这个

```bash
sudo chmod +x -R .
```
* 真正的启动了！！

```bash
docker-compose up -d
```
* 如果出现error重启尝试就好了

```pyhon
docker-compose stop # 关闭
docker-compose up -d # 然后再启动

后续就可以 用 stop 和 start 来控制 服务的开关
```
Dockerfile
```
FROM python:3.11-slim-bookworm as build

WORKDIR /opt/CTFd

# hadolint ignore=DL3008
#RUN echo `ls -lah /etc/apt/sources.list.d`
RUN sed -i "s@http://deb.debian.org@http://mirrors.aliyun.com@g" /etc/apt/sources.list.d/debian.sources
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        libffi-dev \
        libssl-dev \
        git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && python -m venv /opt/venv

ENV PATH="/opt/venv/bin:$PATH"

COPY . /opt/CTFd

RUN pip install --no-cache-dir -r requirements.txt -i https://mirrors.ustc.edu.cn/pypi/web/simple \
    && for d in CTFd/plugins/*; do \
        if [ -f "$d/requirements.txt" ]; then \
            pip install --no-cache-dir -r "$d/requirements.txt"  -i https://mirrors.ustc.edu.cn/pypi/web/simple ;\
        fi; \
    done;


FROM python:3.11-slim-bookworm as release
WORKDIR /opt/CTFd

# hadolint ignore=DL3008
RUN sed -i "s@http://deb.debian.org@http://mirrors.aliyun.com@g" /etc/apt/sources.list.d/debian.sources
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        libffi8 \
        libssl3 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

COPY --chown=1001:1001 . /opt/CTFd

RUN useradd \
    --no-log-init \
    --shell /bin/bash \
    -u 1001 \
    ctfd \
    && mkdir -p /var/log/CTFd /var/uploads \
    && chown -R 1001:1001 /var/log/CTFd /var/uploads /opt/CTFd \
    && chmod +x /opt/CTFd/docker-entrypoint.sh

COPY --chown=1001:1001 --from=build /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

USER 1001
EXPOSE 8000
ENTRYPOINT ["/opt/CTFd/docker-entrypoint.sh"]
```
docker-compose.yml
```
services:
  ctfd:
    build: .
    user: root
    restart: always
    ports:
      - "80:8000"
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
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - db
    networks:
        default:
        internal:
        frp_connect:
          ipv4_address: 172.1.0.5


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

  frps:
    image: glzjin/frp
    restart: unless-stopped
    volumes:
      - ./conf/frp:/conf
    entrypoint:
      - /usr/local/bin/frps
      - -c
      - /conf/frps.ini
    ports:
      - 50000-50100:50000-50100   # Pwn
      - 8080:8080   # Web
    networks:
      default:
      frp_connect:
        ipv4_address: 172.1.0.3

  frpc:
    image: glzjin/frp:latest
    restart: unless-stopped
    volumes:
      - ./conf/frp:/conf/
    entrypoint:
      - /usr/local/bin/frpc
      - -c
      - /conf/frpc.ini
    depends_on:
      - frps
    networks:
      frp_containers:
      frp_connect:
        ipv4_address: 172.1.0.4

networks:
  default:
  internal:
    internal: true
  frp_connect:
    driver: overlay
    internal: true
    attachable: true
    ipam:
      config:
        - subnet: 172.1.0.0/16
  frp_containers:
    driver: overlay
    internal: true  # 如果允许题目容器访问外网，则可以去掉
    attachable: true
    ipam:
      config:
        - subnet: 172.2.0.0/16
```
