##### 底层详细信息

docker inspect logstash

###### Nginx容器缺少vim

1. 启动调试容器 docker run -it --rm --name nginx-fix --entrypoint /bin/bash nginx:latest

2. 在容器内安装vim apt-get updateapt-get install -y vim 

3. 退出容器但保持运行（不要用exit，用Ctrl+P然后Ctrl+Q） 

4. 在另一个终端提交 docker commit nginx-fix nginx-with-vim:latest 

5. 使用新镜像 docker run -d --name my-nginx nginx-with-vim:latest


