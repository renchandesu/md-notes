1. 先创建个elasticsearch用户，因为es限制root用户无法使用es
```
sudo useradd -r -m -s /bin/bash username
passwd username
su username
cd /home/username
```
2. 从官网下载压缩包 kibana和es的
```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.8.6.tar.gz
wget https://artifacts.elastic.co/downloads/kibana/kibana-6.8.6-linux-x86_64.tar.gz
```
3. 然后tar -zxvf进行解压
4. 修改./config/elasticsearch.yml 因为我这里是开启远端登录且需要用户认证但不需要ssl所以这样配置
```
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: false
```
5. 然后启动es `./bin/elasticsearch`
6. 启动完成后需要修改登录的用户名密码
```
./bin/elasticsearch-setup-passwords interactive
# 这个命令可以修改es中elastic kibana等的密码 ， 按照提示修改即可
```
7. 然后安装kibana 
8. 修改kibana的配置 ./config/kibana.yml 开启远程访问
```
server.host: "0.0.0.0"
elasticsearch.username: "elastic"
elasticsearch.password: "your password"
```