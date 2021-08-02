---
title: 博客搭建 - 基于Docker+Hexo+Nginx+Butterfly
tags: Hexo
categories: Hexo博客​
abbrlink: 28839
date: 2021-07-28 12:00:13
---



#  前言

> ​        本博客是基于[Hexo](https://github.com/hexojs/hexo)框架搭建，用[Buterfly](https://github.com/jerryc127/hexo-theme-butterfly)主题进行美化。Hexo是基于Node.js通过将Markdown文章渲染成静态网页，非常方便、简洁。hexo-theme-butterfly是基于hexo-theme-melody 的基础上进行开发的。考虑到环境搭建方便，在云主机上使用了Docker+Nginx进行部署。在搭建过程中经历了许多坎坷 ，有些问题没有记录下来，也参考了其他许多博主的教程。

##  参考教程：

|     作者     |                             链接                             |
| :----------: | :----------------------------------------------------------: |
|    Jerry     | [Butterfly官方文档](https://butterfly.js.org/posts/21cfbf15/) |
|    Akilar    | [Butterfly 主题美化日记](https://akilar.top/posts/f99b208/)  |
|  过客～励む  | [Hexo+github 搭建博客 (超级详细版，精细入微)](https://yafine-blog.cn/posts/4ab2.htm) |
| 唐先森の博客 | [Hexo+Butterfly主题美化](https://ethant.top/articles/hexo541u/#%E5%AE%89%E8%A3%85%E4%B8%BB%E9%A2%98) |
|   小冰博客   | [小冰插件包 butterfly-orchid 1.0](https://zfe.space/post/hexo-butterfly-orchid.html) |
|  Hugh_Dong   | [「Docker」配置Hexo+Git+Nginx](https://blog.csdn.net/qq_33282586/article/details/80637218) |
|    因特马    | [如何搭建自己的博客](https://blog.csdn.net/Colton_Null/article/details/93805908) |

>  <i class="fas fa-bullhorn"></i>    如有遗漏文章注明出处，请联系我添加

#  博客搭建

> 主机环境信息：
>
> 1. Ubuntu
> 2. Git v2.22.0
> 3. npm v6.14.13
> 4. node v12.18.0
>

## Ubuntu安装[Git](https://git-scm.com/download/linux)

   ```bash
   sudo apt install git-all
   ```

##  Clone Git 库

1. 在Github创建一个仓库，用于备份博客文件。

2. 先在服务器上生成一个ssh key。

    ```
    ssh-keygen -t rsa -b 4096 -C "email@example.com"
    ```
   
3. 然后将仓库克隆下来

    ```bash
    touch /usr/blog
    cd /usr/blog
    git clone git@github.com:Hem1ock/hexo-blog.git
    ```


## 创建Hexo镜像并运行容器

1. 创建Dockerfile

    ```dockerfile
    FROM node:latest
    MAINTAINER Hem1ock 
    COPY . /usr/blog
    # set home dir
    WORKDIR /usr/blog
    RUN apt update
    RUN apt upgrade
    RUN apt install vim
    RUN npm install
    # install hexo
    RUN npm install hexo-cli -g
    # install hexo server
    RUN npm install hexo-server
    RUN npm install hexo-deployer-git
    RUN npm i hexo-theme-butterfly
    RUN npm install hexo-renderer-pug hexo-renderer-stylus --save
    # set butterfly themes
    RUN sed "s/theme: landscape/theme: butterfly/g" -i /usr/blog/_config.yml
    ```

2. 执行dockerfile文件，创建一个镜像：
    ```bash
    docker build -t hexo-image 
    ```

3. 创建名字为hexo-blog的容器
    ```bash
    docker run -itd -p 4000:4000 -v /usr/blog/hexo-blog/:/usr/blog --name hexo-blog hexo-image
    ```

4. 进入docker容器，初始化Hexo
    ```bash
    docker exec -it hexo-blog /bin/bash
    hexo init
    ```

5. 生成静态博客网页（之后文章有更新只用到这两条）

    ```bash
    hexo clean
    hexo g -d //生成页面后部署
    #或者在宿主机执行以下的命令
    docker exec hexo-blog hexo clean
    docker exec hexo-blog hexo g -d
    ```

##### <i class="fas fa-code"></i> Hexo命令配置
- `- clean`         //清缓存
- `- deploy`       //部署网站
- `- douban`     //生成豆瓣的页面，需要安装
- `- generate`   //生成静态html文件
- `- init`            //创建新的Hexo文件夹（初始化）
- `- new page <page-title>`           //生成新的页面
- `- server  -P`  端口号    //本地服务预览

##  创建并运行Nginx容器

1. 获取Nginx镜像
    ```bash
    docker pull nginx
    ```
2. 运行Nginx容器并进入
    ```BASH
    docker run --name=nginx -d -p 80:80 4002:22 nginx
    docker exec -it nginx /bin/bash
    ```

    - `--name`：设置容器的名称。
    - `--d`：表示在后台运行容器。
    - `--p`：指定端口映射。第一个80是宿主机的端口，第二个80是Nginx容器内部的端口。22端口用于git拉取页面
    - `--nginx`：指定nginx容器镜像。

3. 编辑Nginx配置文件

    ```bash
    vi /etc/nginx/conf.d/default.conf
    ```

    ```bash
    server { 
    listen       80; 
    listen  [::]:80; 
    server_name  blog.hem1ock.com;  #修改域名
    access_log  /var/log/nginx/host.access.log  main; 
    location / { 
    root   /usr/blog/hexo-blog;  #修改根目录路径为hexo生成的目录
    index  index.html index.htm; 
    } 
    error_page  404              /404.html; 
    location = /404.html { 
    root   /usr/blog/hexo-blog;  #修改404路径为hexo生成的目录
    } 
    ```

4. 重新加载配置
    ```bash
    ./nginx -s reload
    ```

5. 创建git库

    ```bash
    git init --bare ~/blogs.git 
    ```

6. 创建git hook用于拉取hexo生成的页面

    ```bash
    echo "git --work-tree=/usr/blog/hexo-blog/ --git-dir=/root/blog.git checkout -f" >~/blogs.git/hooks/post-receive 
    ```

7. 给hook执行权限

    ```bash
    chmod a+x ~/blogs.git/hooks/post-receive 
    ```

8. 安装并启动SSH服务

    ```bash
    apt-get install openssh-client openssh-server
    sudo /etc/init.d/ssh start
    ```

## Hexo容器发布到Nginx容器

##### 进入Hexo容器
```bash
docker exec -it hexo-blog /bin/bash
```

1. 生成服务端公钥，建立的过程中会提示输入 passphrase，可填可不填。

    ```bash
    ssh-keygen 
    cat ~/.ssh/id_rsa.pub //复制公钥
    ```
    
    ##### 进入Nginx容器

    ```bash
    sudo docker exec -it nginx /bin/bash
    echo "复制的公钥放这里" > ~/.ssh/authorized_keys 
    chmod 600 ~/.ssh/authorized_keys // 赋予权限
    chmod 700 ~/.ssh 
    /usr/sbin/sshd  // 启动ssh服务
    ```

##### 再进入Hexo容器

1. 在博客根目录下执行下面的命令，安装发布的插件：
    ```bash
    npm install hexo-deployer-git --save
    ```

2. 本地测试ssh

    ```bash
    ssh root@172.17.0.2  -p 22  #172.17.0.2为Nginx容器的IP，后面为容器内部远程端口
    ```
	修改hexo根目录下配置文件_config.yml ，配置 hexo 博客可以自动 deploy 到nginx容器上
    ```yml
    #172.17.0.2为Nginx容器的IP，/root/blogs.git为 git 仓库的目录
    deploy:  type: git  repo: root@172.17.0.2:/root/blogs.git  branch: master
    ```

3. 生成推送并查看博客页面

    ```bash
    hexo clean && hexo g -d
    ```

## 更新博客脚本

编写shell脚本并赋予执行权限，更新文章只需要使用Hexo一键三连

```shell
vi blog.sh 
chomod 777 blog.sh

#内容如下：
#! /bin/bash 
cd /usr/blog/hexo-blog 
docker exec hexo-blog hexo clean 
docker exec hexo-blog hexo g -d
```

另外可以设置一个定时任务，每日凌晨定时一键三连

```bash
crontab -e #编辑定时任务
* 0 * * * /root/blog.sh#添加任务
```



# 博客优化

## 修改Butterfly主题

在博客根目录下安装稳定版

```bash
git clone -b master https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly
```

> 升级方法：在主题目录下git pull

## 应用主题

修改博客根目录的全局配置文件 _config.yml，改成butterfly

```yml
theme: butterfly
```

## 安装渲染插件

```bash
npm install hexo-renderer-pug hexo-renderer-stylus --save
```

## 添加豆瓣插件

```bash
#如果设置淘宝国内镜像需要改回来原来的镜像：
npm config set registry https://registry.npm.taobao.org
#设置回原来的镜像：
npm config set registry https://registry.npmjs.org
#安装豆瓣插件
npm install hexo-butterfly-douban --save
```

> 如果使用高版本的node.js安装 hexo-douban 这个插件更新豆瓣页面会出现网络错误，
>
> 需要下载nvm管理工具更改低版本
>
> ```bash
> wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
> nvm install v12.18.0
> ```
>
> 

## 添加字数统计

```bash
npm install hexo-wordcount --save
```

## 添加小猫咪

博客根路径下执行

```bash
npm install --save hexo-helper-live2d
```

下载猫咪的模型，[其他模型点这](https://github.com/xiazeyu/live2d-widget-models)

```bash
npm install --save live2d-widget-model-tororo 
```

在根目录下的配置文件_config.yml添加如下内容

```bash
live2d:
  enable: true
  scriptFrom: local # 默认
  tagMode: false # 标签模式, 是否仅替换 live2d tag标签而非插入到所有页面中
  debug: false # 调试, 是否在控制台输出日志
  model:
    use: live2d-widget-model-shizuku  #模型选择
  display:
    position: right  #模型位置
    width: 150       #模型宽度
    height: 300      #模型高度
    hOffset: 20
    vOffset: -20
  mobile:
    show: false      #是否在手机端显示
```

Hexo一键三连即可看到效果

## 添加Github贡献日历

参考 [Butterfly 主题的首页置顶 gitcalendar](https://akilar.top/posts/1f9c68c9/)

# 遇到的部分问题

## 编译git时报错： zlib.h: No such file or directory

> 缺少 zlib的头文件， 没装开发包。

ubuntu 使用

```bash
 sudo apt-get install zlib1g-dev
```

## Docker 报错“Dockerfile parse error line 1: FROM requires either one or three arguments”

>  '#' 开头一行被视为评论，出现在其他位置视为参数。

报错原因：将写在同一行的注释视为参数了。

## make: *** [po/build/locale/bg/LC_MESSAGES/git.mo] 错误 127

> 报错：   SUBDIR templates
>     MSGFMT po/build/locale/bg/LC_MESSAGES/git.mo
> /bin/sh: msgfmt: command not found
> make: *** [po/build/locale/bg/LC_MESSAGES/git.mo] 错误 127

解决方法：

```bash
apt install gettext
```

暂时记录这么多先，如果后续还有补充再更新~

