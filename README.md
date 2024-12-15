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
