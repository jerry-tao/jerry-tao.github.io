---
layout: post
title: "Move Rails Development to Docker"
date: 2015-8-14 14:47:06
categories: tools
image: /assets/images/chemistry-740453_1280.jpg

---


## Install Docker On Mac

### Prepare

需要了解的几个工具：

- docker
- boot2docker
- docker-compose
- virtualbox


docker才是我们真正想安装的东西，不过不像其他的大部分软件，docker只兼容linux内核，所以在mac和win下都需要boot2docker。

先说一下如果我们想装一个只能在linux下运行的软件的常规思路吧，装个虚拟机，虚拟个linux，boot2docker就做了这件事，他会下载一个很小的linux内核iso来安装到virtualbox里，然后把在本机上运行的docker命令映射到虚拟机里面去。

**所以，这里我们并没有在自己的系统上运行docker，使用网络的时候一定要注意，使用boot2docker本机IP对应的是虚拟机的IP。**

docker-compose是一种使用yml来配置简单的把几个docker容器放到一起跑的方法。



### Install

1. Virtualbox

    ```
      brew install caskroom/cask/brew-cask
      brew cask install virtualbox
    ```

2. 安装boot2docker

    ```
      brew install boot2docker
    ```

3. 初始化boot2docker

    这里初始化会下载boot2docker.iso还是很慢的，下面是baidu云链接。

    [http://pan.baidu.com/s/1eQi295S](http://pan.baidu.com/s/1eQi295S)

    ```
      make ~/.boot2docker && cp boot2docker.iso ~/.boot2docker
      boot2docker init
    ```

    上面这个执行一次就好，用来配置虚拟机的。

4. 启动boot2docker

    ```
      boot2docker up
      $(boot2docker shellinit)
  	```

  	前者用来启动boot2docker的虚拟机，建议设为开机自动执行，后者用来映射命令，建议放到terminal的.bashrc/.zshrc里。

  	如果不出错，这个时候docker就可以使用了，可以打印info测试一下。

  	```
      docker info
      docker pull ubuntu:latest
  	```

5. 安装docker-compose

    ```
      brew install docker-compose
    ```

6. 配置镜像加速

    国内下载image很慢。下面这个会略微提升，不过也谈不上很快就是了。


    [http://www.daocloud.io/](http://www.daocloud.io/ )

    注册个账号，找到加速器，按照提示修改即可。


### 使用之前需要了解的几个概念

- 镜像
- container
- Dockerfile
- docker-compose
- volumes

去这个的在线版先快速了解一下：

[Docker 技术入门与实战](https://github.com/yeasy/docker_practice)


### Rails On Docker

新建一个Dockerfile:

```
 # 以ruby:2.2.2作为基础镜像 维护者的邮箱
FROM ruby:2.2.2
MAINTAINER taojay315@gmail.com

 # RUN 命令会在构建过程中执行
RUN apt-get update && apt-get install -y \
build-essential \
nodejs

 # 新建一个/app文件夹
RUN mkdir -p /app
 # 我们命令执行的文件夹
WORKDIR /app

 # 把Gemfile拷贝进去先安装好所需Gem
COPY Gemfile Gemfile.lock ./
RUN gem install bundler && bundle install --jobs 20 --retry 5

 # 暴露3000端口
EXPOSE 3000

 # 设置默认命令执行的前缀
ENTRYPOINT ["bundle", "exec"]

 # 启动container时的命令，每次启动只能执行一个命令
CMD ["rails", "server", "-b", "0.0.0.0"]# default.

```

实际上，如果仅仅想使用Docker作为rails环境来说，这个就已经足够了。需要注意的是，dockerfile虽然规定了一些内容，但是我们在docker-compose或者build run的时候都是可以覆盖掉的。

比如CMD，我们可以自己执行docker-compose run app rails c 这样就会执行rails c而不是rails s了。

所以如果使用docker-compose的话，这里面的配置都只能算是默认值。

然后是docker-compose.yml文件：

```
app:
  build: .
  command: # 覆盖掉Dockerfile里的命令 注意 这个在运行docker-compose的时候还是可以覆盖掉的
    - rails server -p 3000 -b '0.0.0.0'
  volumes: # 把当前文件夹作为数据卷挂载到/app
    - .:/app
  ports:   # 映射端口 注意 这个端口是映射到了虚拟机里，而不是我们真正的系统
    - "3000:3000"
```

然后可以执行

```
docker-compose build
dcoker-compose up -d # 在后台运行
docker-compose run app rake db:create # 不是sqlite3 会出错
docker-compose run app rake db:migrate # 不是sqlite3 会出错
```

**注意 这个端口映射的是虚拟机端口，所以这个时候你可以选择**

- 映射本机端口到虚拟机端口

  打开Vbox，找到正在运行的虚拟机->设置->网络->高级->端口映射

- 直接访问虚拟机IP

  ifconfig->找到虚拟机IP->访问虚拟机IP

不过，如果你使用的数据库不是sqlite3，这里估计你看到的还是报错信息。


### Rails DB On Docker

然后来解决一下非sqlite3的情况（估计都是）。

修改docker-compose.yml，

```
app:
  build: .
  command:
    - rails server -p 3000 -b '0.0.0.0'
  volumes:
    - .:/app
  ports:
    - "3000:3000"
  links:
    - postgres
postgres:
  image: postgres:9.4
  ports:
    - "5432"
```

在这里关联了一个postgres的镜像，这就是docker的方便之处了，像这样关联之后，会自动暴露一些变量给app。

```
docker-compose run app env #打印app里的环境变量
```

默认情况下你应该能看到这两个变量：

```
POSTGRES_PORT_5432_TCP_ADDR
POSTGRES_PORT_5432_TCP_PORT
```

然后修改一下你的database.yml

```
default: &default
  adapter: postgresql
  encoding: unicode
  pool: 5
  timeout: 5000
  username: postgres
  host: <%= ENV['POSTGRES_PORT_5432_TCP_ADDR'] %>
  port: <%= ENV['POSTGRES_PORT_5432_TCP_PORT'] %>

development:
  <<: *default
  database: demo_proj
```
默认密码就是空，然后就可以使用数据库了。

再次执行：

```
docker-compose stop
docker-compose up -d
docker-compose run app rake db:create
docker-compose run app rake db:migrate
```

这个数据库的具体文件是保存在boot2docker的虚拟机里的，在我本机上的地址如下：

```
/mnt/sda1/var/lib/docker/volumes/6acb454afbd139c93803dffbdd4931cca3ddf61ad25dac620da7ff5b3dce5d6b/_data
```


### Rails XXXX On Docker

使用其他的依赖跟上面类似，下面这个配置文件配置了ES还有redis到rails上：

```
app:
  build: .
  command:
    - rails server -p 3000 -b '0.0.0.0'
  volumes:
    - .:/app
  ports:
    - "3000:3000"
  links:
    - postgres
    - redis
    - elasticsearch
postgres:
  image: postgres:9.4
  ports:
    - "5432"
redis:
  image: redis:3.0.3
  ports:
    - "6379"
elasticsearch:
  image: elasticsearch:1.7.1
  ports:
    - "9200"

```


至此，测试和开发都可以在rails里进行了。

执行测试命令：

```
docker-compose run -e 'RAILS_ENV=test' app rake db:create
docker-compose run -e 'RAILS_ENV=test' app rake db:migrate
docker-compose run app rspec
```

其他人拿到代码之后只需要使用docker-compose build 就可以创建一个完整的配置好了的开发环境了，并且环境完全一致。

已知问题：

- 每次更新了Gemfile都需要重新build一次。
- 运行spec速度慢于本地。
