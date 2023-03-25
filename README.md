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
  
  -p 3000:80  将容器内80端口映射至宿主机9980端口，这是访问gitlab的端口
  
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
  external_url 'http://101.133.225.166'
  # ssh主机ip
  gitlab_rails['gitlab_ssh_host'] = '101.133.225.166'
  # ssh连接端口
  gitlab_rails['gitlab_shell_ssh_port'] = 9922
  
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
