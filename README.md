最近在学习可持续集成，可持续部署这方面的东西，本着好记性不如烂笔头的特性，把学习的东西记录一下

***

*  **安装及部署docker** 

  ```
  # 安装依赖
  yum install -y yum-utils device-mapper-persistent-data lvm2
  
  # 设置yum源
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  
  # 安装docker
  yum install -y docker-ce
  
  # 设置开机启动
  systemctl enable docker
  
  # 启动 Docker
  systemctl start docker
  
  # 查看版本
  docker version
  ```

  ***

  * **安装及部署gitlab(社区版)**

  ```
    # 使用docker安装gitlab
   docker pull gitlab/gitlab-ce:latest
  ```

  ```
  mkdir gitlab gitlab/etc gitlab/log gitlab/opt  
  
  docker run -id -p 3000:80 -p 9922:22 -v /root/gitlab/etc:/etc/gitlab  -v /root/gitlab/log:/var/log/gitlab -v /root/gitlab/opt:/var/opt/gitlab --restart always --privileged=true --name gitlab gitlab/gitlab-ce
            
  
  命令解释：
  -i  以交互模式运行容器，通常与 -t 同时使用命令解释：
  
  -d  后台运行容器，并返回容器ID
  
  -p 3000:80  将容器内80端口映射至宿主机3000端口，这是访问gitlab的端口
  
  -p 9922:22  将容器内22端口映射至宿主机9922端口，这是访问ssh的端口
  
  -v ./gitlab/etc:/etc/gitlab  将容器/etc/gitlab目录挂载到宿主机./gitlab/etc目录下，若宿主机内此目录不存在将会自动创建，其他两个挂载同这个一样
  
  --restart always  容器自启动
  
  --privileged=true  让容器获取宿主机root权限
  
  --name gitlab-test  设置容器名称为gitlab
  
  gitlab/gitlab-ce  镜像的名称，这里也可以写镜像ID
  
  ```

  进入容器进行相应修改

  ```
  docker exec -it gitlab /bin/bash
  
  # 修改gitlab.rb
  vi /etc/gitlab/gitlab.rb
  ## 加入如下
  # gitlab访问地址，可以写域名。如果端口不写的话默认为80端口
  external_url 'http://101.133.225.166:3000'
  # ssh主机ip
  gitlab_rails['gitlab_ssh_host'] = '101.133.225.166'
  # ssh连接端口
  gitlab_rails['gitlab_shell_ssh_port'] = 9922
  #更改nginx端口
  nginx["listen_port"]= 80
  
  # 让配置生效
  gitlab-ctl reconfigure
  
  ### 注意不要重启，/etc/gitlab/gitlab.rb文件的配置会映射到gitlab.yml这个文件，由于咱们在docker中运行，在gitlab上生成的http地址应该是http://101.133.225.166:3000,所以，要修改下面文件
  
  # 修改http和ssh配置
  vi /opt/gitlab/embedded/service/gitlab-rails/config/gitlab.yml
  
    gitlab:
      host: 101.133.225.166
      port: 3000 # 这里改为3000
      https: false
  
  
  # 重启
  gitlab-ctl restart
  # 退出容器
  exit
  
  ```

  在浏览器中访问

  ```
  # 机器配置要大于4g，否则很容易启动不了，报502
  http://101.133.225.166:3000/
  
  # 第一次访问，会让修改root密码
  # 修改后以root用户登录即可
  ```

  获取root密码登录后修改

  ```
  sudo docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
  ```

  附: [gitlab 官方安装文档](https://docs.gitlab.cn/jh/install/docker.html)



***

* **安装及部署Jenkins**

  ```
  yum -y install java-11-openjdk
  wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo --no-check-certificate
  rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
  yum install jenkins
  systemctl daemon-reload
  systemctl start jenkins
  systemctl enable jenkins
  
  
  systemctl status jenkins
  systemctl stop jenkins
  
  ```
  
  * **安装git**
  
    ```
    yum install -y git
    
    # 查看版本
    git version
    
    # 查看git目录路径
    which git
    ```
  
    **然后找到Jenkins下的系统管理-全局工具配置-Git**
  
    **将上面复制的git的路径放在Path to Git executable中**
  
    
  
    **问题：** 
  
    ​    在Jenkins里使用docker build 打包镜像的时候出现了Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "/library/node:16-alpine-base/json": dial unix /var/run/docker.sock: connect: permission denied
  
    将jenkins所用的用户添加到docker用户组中
  
    
  
    1、查看当前docker用户组都有谁。
  
    cat /etc/group | grep docker
  
    
  
    2、将当前用户添加到docker组中。
  
    usermod -a -G docker jenkins
  
    
  
    3、更新用户组
  
    newgrp docker
  
    
  
    4、chmod 777 /var/run/docker.sock

##### 问题:

#####       Jenkins中提示Host key verification failed解决办法

```
[root@localhost ~]# grep "jenkins" /etc/passwd
jenkins:x:988:983:Jenkins Automation Server:/var/lib/jenkins:/bin/false
```

 变为:

```
[root@localhost ~]# vim /etc/passwd
jenkins:x:988:983:Jenkins Automation Server:/var/lib/jenkins:/bin/bash
```

切换jenkins用户 测试

```
[root@localhost ~]# su jenkins
bash-4.2$ exit
exit
```

重新定义jenkins用户的命令提示符

```
[root@localhost ~]# vim .bash_profile 
末行追加
export PS1='[\u@\h \W]\$ '
[root@localhost ~]# source .bash_profile 
```

再次切换到jenkins用户测试

```
[root@localhost ~]# su jenkins
[jenkins@localhost root]$ exit
exit
```

jenkins用户对目标主机做免密登陆

```
[root@localhost ~]# su jenkins
[jenkins@localhost root]$ ssh-keygen -t rsa
[jenkins@localhost root]$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@10.0.0.41

# 验证
ssh 'root@10.0.0.41'

```



  **jenkins 插件安装**:

​    ***git parameter***      # 作用是从gitlab拉取代码

  ***Role-based Authorization Strategy***   # 作用是jenkins的权限管理(安装后在全局安全配置的授权策略选项中选择它)

***GitLab Plugin***   # This plugin allows [GitLab](http://gitlab.com/) to trigger Jenkins builds and display their results in the GitLab UI.(gitlab提交代码Jenkins自动打包发布)

注意:  1.在安装 GitLab Plugin 后在相应的项目的 构建触发器中 选择相应的选项;

​           2.在gitlab 总的设置(不是项目的设置)中 选择 **网络** 的 **出站请求**  然后 *Allow requests to the local network from webhooks and integrations*  这个选项选中保存.

​         3.在gitlab 某个项目的**设置**中选择 **Webhooks**，并填写和选择相应选项(url填写为jenkins **触发构建器** 选项中的地址)

​        4.在Jenkins 的**系统管理** 的 **gitlab** 中去掉 *Enable authentication for '/project' end-point*  选项

   

   然后就可以做到gitlab提交代码jenkins自动构建发布了。

附:[参考视频](https://www.bilibili.com/video/BV1pF411Y7tq?p=33&vd_source=ee5c1fe08cb3fe64aaa4063de52804c4)

**nodejs**

**publish over  SSH**



**Jenkins 指定具体的分支进行构建:**

  在项目配置中点击“参数化构建过程”,名称:branch; 参数类型:分支或标签;指定分支(为空时代表any):${branch}  即可



```
#!/bin/bash
pwd
docker build -t nginx-agent:$version .
docker tag nginx-agent:$version 124.221.113.29:90/public/nginx-agent:$version
docker login 124.221.113.29:90 -u admin -p Harbor12345
docker push 124.221.113.29:90/public/nginx-agent:$version
# 推送镜像后删除镜像
docker rmi nginx-agent:$version
docker rmi 124.221.113.29:90/public/nginx-agent:$version

ssh root@124.221.113.29 "docker stop nginx-test;docker container rm nginx-test"

# 要删除现有容器未使用的所有镜像
ssh root@124.221.113.29 "docker image prune -a  --force"

ssh root@124.221.113.29 "docker pull 124.221.113.29:90/public/nginx-agent:$version"
ssh root@124.221.113.29 "docker run --name nginx-test -p 8001:80 -d 124.221.113.29:90/public/nginx-agent:$version"
```



***

1.**安装及部署Harbor**

   安装 docker-compose

```powershell
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum -y install docker-compose-plugin
cp /usr/libexec/docker/cli-plugins/docker-compose /usr/bin
```

**安装 Harbor**

```
wget  https://github.com/goharbor/harbor/releases/download/v2.4.2/harbor-online-installer-v2.4.2.tgz

tar xvf harbor-online-installer-v2.4.2.tgz -C /usr/local/
```

```

# harbor配置文件模板
harbor.yml.tmpl  

# 首次安装需要先执行，会生成docekr-compose.yml文件
# 更新harbor配置文件后也需要执行此文件
# ./prepare 
prepare 

# 安装启动脚本
install.sh

# 容器启动相关配置文件目录
common


```



修改相应配置:

修改**harbor.yml.tmpl**文件的相关内容，不需要的需要屏蔽掉

```
cd /usr/local/harbor/
cp harbor.yml.tmpl harbor.yml
vi harbor.yml

###==== 修改以下参数

# 访问域名
hostname : my.harbor.com  



# 配置https证书路径
certificate: /data/cert/my.harbor.com.crt
private_key: /data/cert/my.harbor.com.key

# 管理员密码
harbor_admin_password: AdminPassword

database:
	# 设置pgsql管理员密码
	password: root123

# 存储路径
data_volume: /data

```

安装:

```
# 执行安装脚本
./install.sh
```

harbor重启

```
cd /usr/local/harbor/
docker-compose restart
```

```
// 使用insecure-registries参数添加http支持
 vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://vpr3xe3f.mirror.aliyuncs.com"],
  "insecure-registries": ["192.168.92.130:90"]
}
 systemctl daemon-reload
 systemctl restart docker
```

登陆

```
docker login 192.168.92.130:90
```

推送镜像包到私有仓库:

```
docker pull nginx
 // 远程有linux13这个镜像仓库(harbor地址/目录/镜像名:版本)
docker tag nginx:latest 192.168.92.130:90/linux13/nginx:1.21.1
docker images

docker push 192.168.92.130:90/linux13/nginx:1.21.1
```

下载镜像:

```
docker pull 192.168.92.130:90/linux13/nginx:1.21.1
```



***

* centos 安装locate 命令

  ```
  sudo yum -y install mlocate
  # 初始化
  sudo updatedb
  ```



***

* Jenkins通过安装ssh插件实现构建后推送到其它服务器(非docker镜像)

   1、在**系统管理** ———>**系统配置**里添加不同的要发布项目的SSH Servers;

  2、在项目**配置**下的**构建后操作**的**添加构建后添加操作**选择**send build artifacts. over SSH**(注意:Transfer Set

Source files下需填写dist/**(2个星号),才能复制dist文件下的目录)

***

* 安装nginx

  ```
  # 安装所需环境
  yum install -y gcc-c++   pcre pcre-devel zlib zlib-devel openssl openssl-devel
  
  # 官网下载nginx
  wget -c https://nginx.org/download/nginx-1.22.1.tar.gz
  
  tar -zxvf nginx-1.12.1.tar.gz
  cd nginx-1.12.1
  
  # 配置
  ./configure
  
  # 编译安装
  make
  make install
  
  # 查找安装路径
  whereis nginx
  
  # 启动、停止nginx
  
  cd /usr/local/nginx/sbin/
  ./nginx 
  ./nginx -s stop
  ./nginx -s quit
  ./nginx -s reload
  
  
  
  ```

  

