**查询进程**
sudo ss -tulnp | grep <port>
sudo kill -9 <PID>

**下载和上传**
apt upload
apt install lrzsz
sz
rz

**查看文件有多大 进入cd**
du -sh 

**查看外网ip**
curl http://api.ip.sb/ip

###### 查看端口情况

nc -zv localhost 80

###### 指定目录解压

tar -xvf archive.tar.gz -C /target/path

zip -r archive.zip dir1/

##### 提取大文件log

提取12点到17点（包含12,13,14,15,16,17点）

grep -E "02/Dec/2025:(12|13|14|15|16|17):" /var/log/nginx/access.log > /tmp/nginx_12_17.log

##### 生成分析报告

goaccess nginx_12_17.log --log-format=COMBINED -o index.html

###### 查看防火墙状态

ufw status

允许来自任何IP地址对9100端口的访问（通用设置） 

sudo ufw allow 9100/tcp 

# 或者，为了更安全，可以只允许来自您特定管理IP（172.16.2.1）的访问（推荐） 

sudo ufw allow from 172.16.2.1 to any port 9100
