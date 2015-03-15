---
layout:         post
title:          "Gitolite with Amazon EC2"
date:           2015-03-15
categories:     AWS EC2
tags:           AWS EC2 gitolite
image:          /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---


<br/>
昨天重启了一下 EC2 实例，结果导致 ssh 不能连接到远程，总是 permission denied，想必是先前自己胡乱配置的 gitolite 把 authorized_key 搞出翔了。尝试使用 snapshot 创建 AMI 恢复虚拟机，悲催的是仍然无解。(此处省略惨叫若干声... 喵...)

好吧，被逼无奈外加的确需要严肃对待这等严重问题。重新搞一遍吧，顺便写个 blog 把步骤理干净清晰了，给健忘的自己提个醒。

### 准备工作
自己使用的是 Amazon Linux AMI, 属于 RHEL 的一个分支发行版，所以操作命令沿袭 RHEL 使用习惯。重新 launch 一个 instance 就不记录了，完全无脑 next next next ... 进入穿越至配置阶段。

{% highlight bash %}
# Install Git
cd ~
[ec2-user@foo]$ sudo yum -y update
[ec2-user@foo]$ sudo yum -y install git

# Add Git user
[ec2-user@foo]$ sudo useradd git
[git@foo]$ sudo su git

# Config git global user's info
cd ~
[git@foo]$ git config --global user.email "admin@yourwebsite.com"
[git@foo]$ git config --global user.name "admin"
{% endhighlight %}

上面的操作很基础，安装配置 Git 同时为 gitolite 新建一个 git 用户。接着就是在服务器端生成 rsa 密钥。这里需要说明一下的是我计划如何管理远程仓库, 以安全性为考量的出发点，我只希望涉及 gitolite 的管理操作都只能在 remote shell 中完成，所以这就需要给新建的 git 账户生成一套密钥以供远程操作 gitolite-admin。

{% highlight bash %}
[git@foo]$ ssh-keygen -t rsa -C "admin"
{% endhighlight %}

<br/>

### 安装 GITOLITE

{% highlight bash %}
[git@foo]$ git clone git://github.com/sitaramc/gitolite
[git@foo]$ mkdir -p /home/git/bin
[git@foo]$ export PATH=/home/git/bin:$PATH

# install gitolite to ~/bin
[git@foo]$ gitolite/install -ln
# Use the pub key we just created for gitolite setup
[git@foo]$ gitolite setup -pk ~/admin.pub
# Copy private key to ~/.ssh
[git@foo]$ cp admin ~/.ssh/id_rsa

# clone gitolite-admin
[git@foo]$ git clone git@localhost:gitolite-admin
{% endhighlight %}

简单的几步，gitolite 就准备就绪，通过 remote shell 可以方便的管理远程的代码仓库。需要本地连接远程的时候，过程也很简单。先本地 ssh-keygen, 然后把 public key 上传到 gitolity-admin/keydir, commit & push。远程修改 gitolite.conf 中仓库的访问控制。

Job done!

<br/>

> We destroy, we create and reshape, that is how we roll ...