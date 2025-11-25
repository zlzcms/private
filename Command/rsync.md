### 要使用rsync命令从一台Linux机器同步一个文件夹到另一台Linux机器，你需要确保两台机器上都安装了rsync并且网络连接正常。

rsync -avz --progress <源路径> <目标路径>

### 当需要从一台远程 Linux 机器同步到本地时，可以使用如下格式：

rsync -avz --progress user@remote-host:/path/to/source /local/path/to/destination

### 使用ssh

下载：

rsync -avz --progress -e "ssh -i /mnt/c/Users/Admin/.ssh/<id>" user@remote-host:<源路径> <目标路径>

上传:
rsync -avz --progress -e "ssh -i /mnt/c/Users/Admin/.ssh/"  <目标路径> user@remote-host:<源路>

## 从 WSL 同步到 Windows

rsync -avz --progress /home/用户名/project/ /mnt/c/Users/Windows用户/project/


### 排除文件/目录

rsync -avz --progress --exclude 'node_modules' /源路径/ /目标路径/



### 反之， 交换

详细解释
-a: 归档模式，保留原有文件属性。
-v: 显示详细信息。
-z: 启用压缩。
--progress: 显示进度条。

rsync -avz --progress root@110.40.138.41:/www/ .

rsync -avz --progress root@172.16.2.231:/www/_projects/MTAgentAPI/deploy/ .