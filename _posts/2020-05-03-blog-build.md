---
layout: post
title: 本博客搭建指南
categories: 杂文
description: 使用Jekyll搭建一个个人博客
keywords: Jekyll
topmost: true
---

这里汇总了一下基于Jeklly以及码志模板搭建个人博客的整个流程：

## 安装Ruby

Ruby是使用Jeklly最核心的组件，推荐使用rvm来进行Ruby的安装：

> https://ruby-china.org/wiki/rvm-guide

- Mac

  Mac系统自带了Ruby，但是其版本相对较低，所以可以直接安装rvm来进行Ruby版本的管理：

  ```shell
  $ gpg2 --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
  $ curl -sSL https://get.rvm.io | bash -s stable
  $ source ~/.bashrc
  $ source ~/.bash_profile
  ```

  修改RVM的Ruby安装源来提高安装速度：

  ```shell
  $ echo "ruby_url=https://cache.ruby-china.com/pub/ruby" > ~/.rvm/user/db
  ```

  安装完成后读取系统已经安装的Ruby：

  ```shell
  $ rvm automount
  ```

  然后可以列出所有可用的Ruby并安装想要安装的版本，这里选择了Ruby 2.7.2：

  ```shell
  $ rvm list known
  $ rvm install 2.7.2 --disable-binary
  ```

  安装完成后切换到Ruby 2.7.2并设置为默认的Ruby：

  ```shell
  $ rvm use 2.7.2 --default
  ```

  至此在Mac上Ruby的安装就已经结束了。

- Centos 7

  Centos 7上安装Ruby可以采用rvm也可以编译源码安装，无论是那种这里可以首先使用yum安装一个默认低版本的Ruby：

  ```shell
  $ yum install ruby -y
  ```

  然后如果要安装rvm，那和上面Mac的步骤一致，如果要编译安装，那么首先到Ruby官网下载源码包，然后编译安装：

  ```shell
  $ wget https://cache.ruby-lang.org/pub/ruby/2.7/ruby-2.7.4.tar.gz
  $ tar zxvf ruby-2.7.4.tar.gz
  $ cd ruby-2.7.4
  $ ./configure --prefix=/usr/local/ruby
  $ make && make install
  ```

  即可。

## 安装Jekyll和码志模板

Jekyll只需要使用gem安装即可：

```shell
$ gem install jekyll bundle
```

安装好之后如果要新建一个空项目可以执行：

```shell
$ jekyll new my-awesome-site
$ cd my-awesome-site
$ bundle install
$ bundle exec jekyll serve
```

执行完上述指令后，打开http://localhost:4000即可访问，如果要修改端口可以使用 `bundle exec jekyll serve --port PORT指定端口启动，具体可以参考文档：

> http://jekyllcn.com/docs/configuration/

而如果要使用码志主题，那么直接克隆即可：

```shell
$ git clone https://github.com/mzlogin/mzlogin.github.io.git
```

就可以将带有码志模板的博客克隆本地了，接下来可以参考官方git给出的步骤进行设置：

> https://github.com/mzlogin/mzlogin.github.io

这里要注意的一点就是，如果要部署到自己的服务器上，那么要保留CNAME文件，并且在CNAME文件中和_config.yml文件中的baseUrl属性填上自己的网址，否则在页面跳转时会因为找不到网址而重定向到localhost:PORT下。

## 部署

由于Jekyll可以编译为静态网站，所以这里首先编译Jekyll网站，进入到网站根目录中：

```shell
$ cd mzlogin.github.io.git
$ bundle build
```

编译完成后就会在网站根目录下出现_site文件夹，这就是后面要使用静态网站的目录。这里配置的时候使用了Nginx作为服务器，那么只需要将location指向这个文件夹即可：

```
server {
    listen 80;
    server_name blog.saturn-io.com;
    rewrite ^(.*)$ https://$host$1;
    root html;
    location / {
        index index.html index.htm;
    }
}

server {
    listen 443 ssl;
    server_name blog.saturn-io.com;
    ssl_certificate cert/blog.pem;
    ssl_certificate_key cert/blog.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
        root /home/site/mzlogin.github.io/_site;
        index index.html index.htm;
    }
}
```

这样启动nginx即可，这样会存在的一个问题就是每次添加或修改文档都需要重新编译网站（暂未解决）。
