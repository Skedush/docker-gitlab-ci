# 使用docker搭建持续化集成环境
## 一、docker部署gitlab-ce


1. 拉取gitlab-ce镜像
```
docker pull gitlab/gitlab-ce:latest
```
2. 运行gitlab-ce容器
```
sudo docker run -d -p 30001:22 -p 30000:80 -p 30002:443 --privileged=true \
-h 127.0.0.1:30000 \
--name gitlab-ce \
-v $HOME/gitlab/config:/etc/gitlab \
-v $HOME/gitlab/logs:/var/log/gitlab \
-v $HOME/gitlab/data:/var/opt/gitlab \
gitlab/gitlab-ce:latest
```

> - -d : 后台进程方式启动  
> - –p 30001:22 ：分别使用30000 ~ 30002 端口映射端口80，22，443  
> - 在创建docker容器的选项中加入--privileged=true，使创建的容器拥有root权限，即可正常访问。
> - –-hostname gitlab ：发布域名叫gitlab，还需要配置域名绑定  
> - –-restart always ：电脑启动时自动启动  
> - –-volume $GITLAB_HOME/config:/etc/gitlab ：挂接卷，映射gitlab的配置到本地文件夹  
> - –-volume $GITLAB_HOME/logs:/var/log/gitlab ：挂接卷，映射gitlab的日志到本地文件夹  
> - –-volume $GITLAB_HOME/data:/var/opt/gitlab ：挂接卷，映射gitlab的数据到本地文件夹  


3. 找到Nginx监听端口行文件$HOME/gitlab/config/gitlab.rb中  找到并修改#nginx['listen_port']=nil-->nginx['listen_port'] = 80，或者直接添加nginx['listen_port'] = 80保存  
	

`docker restart gitlab-ce `

4. 等待5分钟左右访问localhost\:30000或者127.0.0.1\:30000或者0.0.0.0\:30000,要是还在启动中访问会报502错误

5. 配置gitlab，不配置gitlab很消耗内存和cpu，默认的配置消耗资源很高,在映射的宿主机路径$HOME/gitlab/config/gitlab.rb修改或添加相关配置，里面的配置都注释掉了，查看官方文档。以下是我添加的配置

```
 unicorn['worker_memory_limit_min'] = "200 * 1 << 20"
 unicorn['worker_memory_limit_max'] = "300 * 1 << 20"
 sidekiq['concurrency'] = 8
 postgresql['max_worker_processes'] = 3
 unicorn['worker_timeout'] = 60
 unicorn['worker_processes'] = 3
 postgresql['shared_buffers'] = "256MB"
```

6. 创建一个项目，进入项目中的ci/cd设置，展开Runner设置，能看到specific Runner的地址与token后面配置runner的时候要用。

## 二、Docker搭建Gitlab CI Runner
1. 拉取gitlab-runner镜像

```
sudo docker pull gitlab/gitlab-runner:latest
```
2. 运行gitlab-runner容器
```
sudo docker run -d --privileged=true --name gitlab-runner \
  -v $HOME/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \ # 宿主机的docker路径映射
  gitlab/gitlab-runner:latest
```



3. 进入git-runner容器内部
`docker exec -it gitlab-runner bash`
4. 注册runner这里要用到上面gitlab中项目的地址与token。

```
gitlab-runner register
```
```

Please enter the gitlab-ci coordinator URL:
# 这里不能用127.0.0.1与localhost，使用ip或者域名如：http://192.168.0.0.1:30000
Please enter the gitlab-ci token for this runner:
# xxxxxx
Please enter the gitlab-ci description for this runner:
# 示例：这是一个node项目
Please enter the gitlab-ci tags for this runner (comma separated):
# 示例：node
Please enter the executor: docker, parallels, shell, kubernetes, docker-ssh, ssh, virtualbox, docker+machine, docker-ssh+machine:
# docker
Please enter the default Docker image (e.g. ruby:2.1):
# docker:latest 使用docker容器运行runner脚本，运行完成后会关闭docker
```

> docker访问不了宿主机的服务导致regiter访问不到gitlab  
> 关闭防火墙命令  
> sudo firewall-cmd --permanent --zone=trusted --change-interface=docker0  
> sudo firewall-cmd --reload

成功之后在gitlab项目的runner配置下面有激活的runner

5.拉取项目并新建一个以node项目为例子，编写文件  
 package.json
 
```
{
  "name": "docker_demo",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "koa": "^2.5.0"
  }
}
```
server.js

```
const Koa = require('koa');
const app = new Koa();

app.use(async ctx => {
  ctx.body = 'Hello docker';
});

app.listen(3000);
```

dockerfile

```


#制定node镜像的版本
FROM node:latest
#声明作者
MAINTAINER xl
#移动当前目录下面的文件到app目录下
ADD . /app/
#进入到app目录下面，类似cd
WORKDIR /app
#安装依赖
RUN npm install
#对外暴露的端口
EXPOSE 3000
#程序启动脚本
CMD ["npm", "start"]
```
.gitlab-ci.yml
```
image: docker:latest  #使用docker运行脚本
variables:
  DOCKER_HOST: tcp://172.16.1.241:2375    
# docker host，本地的gitlab-runner不需要设置，我们使用的runner是运行在容器内部的，docker在宿主机上，输入宿主机的ip地址与docker端口，docker默认是没有暴露端口的，需要设置。
  TAG: hello-docker:v1 #镜像名称
before_script:
  - echo "hello gitlab ci" #运行脚本前执行的命令
stages:
  - deploy	#应该是有build，test，deploy三个，这里简单一点就用一个
build_job: # 定义一个job
  image: docker:latest
  stage: deploy # 设置job所属的stage
  tags:
    - node #runner注册时输入的tags
  script: # 定义后面Runner来执行的具体脚本
    - docker build -t $TAG .    #创建镜像
    - docker rm -f test || true		#移除容器
    - docker run -d --name test -p 3000:3000 $TAG    #创建容器

```

> dockerfile 是gitlab-ci创建的镜像容器的文件，gitlab-runner检测项目push，检测到之后会运行.gitlab-ci.yml里面的脚本，脚本使用dockerfile创建镜像与容器部署项目

6. git push之后持续化集成开始会报错，找不到docker，在宿主机上暴露docker端口

> vim /usr/lib/systemd/system/docker.service  
> 在 ExecStart=/usr/bin/dockerd-current 后 增加  
> -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock  

7. git push 之后成功自动部署流水线开始。
