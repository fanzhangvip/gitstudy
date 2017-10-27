转载_CI 系统搭建：Git、Gerrit与Jenkins
转载 2014年08月11日 20:55:33 26315
去年写的这五篇 CI 文章时候方便邮件测试，自己搞了一个 thstack.com 域名玩。当时也没在意，所有的文章里邮箱地址都是引用 @thstack.com 域名。让我没想到是，2014 年这个神奇的一年，thstack.com 会成为我们的公司名字和域名。

我想说的是，我们内部的邮件系统也在用 @thstack.com，和这几个文章里的邮箱会冲突，导致一些朋友完全照着我的文档测试。结果最近收到了很多垃圾邮件。
我还想说的是，不要让别人怀疑你的智商。
我最想说的一句是，搞 CI 是一个高尚的工作。
随着计算机的高速发展、各种时代变革的到来。百度、360、腾讯这些赚钱机器之间为了利益掐个没完没了。开源项目、社区的活跃进而引来持续集成（CI）系统的诞生。也越发的听到更多的人在说协同开发、敏捷开发、迭代开发、持续集成和单元测试这些拉风的术语。但是也仅仅只是听到在说而已，也没有见到国内有几家公司能有完整的 CI 体系流程。反之一些开源项目都有完整的 CI 体系。我觉得就这一点国内的公司还是要去学习的。

由于对上面那些生涩的词感冒，专门研究了下 CI 相关的系统，才有了这几篇文章

我用到的系统有：

Gitlab：代码托管
Gerrit：Code Review
Jenkins：代码测试

一开始测试使用了 Gitorious 来做代码的托管，发现界面的功能不全，比较蛋疼。虽然 Gerrit 本身有代码托管功能，Gerrit 的界面不敢恭维、也没有 Gitlab 的功能强大。so …

决定使用 Gitlab 还有一点重要原因就是它本身提供保护分支的功能，可以达到 Review 效果。这样和 Gerrit 结合的话，可以针对不同的用户群来分配 Review 方式。

强制 Review：在 Gitlab 上创建的项目，指定相关用户只有 Reporter 权限，这样用户没有权限使用 git push 功能，只能 git review 到 Gerrit 系统上，Jenkins 在监听 Gerrit 上的项目事件会触发构建任务来测试代码，Jenkins 把测试结果通过 ssh gerrit 个这个项目打上 Verified 成功或失败标记，成功通知其它人员 Review。

Gitlab 保护 Master 分支：在 Gitlab 上创建的项目可以把 Master 分支保护起来，普通用户可以自己创建分支并提交代码到自己的分支上，没有权限直接提交到 Master 分支，用户最后提交申请把自己的分支 Merge 到 Master，管理员收到 Merge 请求后，Review 后选择是否合并。

由于没有闲置服务器、公网 IP、域名让我去浪费。想到 qingcloud 上还有 2k 没花，所以需要把三套系统整合塞到一个 vps 中。在整合三套系统中遇到一些需要规划的小问题

每个系统都有发送邮件的功能，最好弄三个邮箱账号
三个系统都默认监听了 8080 端口，需要规划端口
规划了端口的同时随便规划下三个域名，后端做个端口转发方便用户访问
知道什么需要规划后，就来设置吧：

使用系统

ubuntu-12.04.2-server
修改 ssh 时候不需要输入 yes，如果不设置后面 Jenkins 连接 Gerrit 时候会报错

echo 'StrictHostKeyChecking  no' >> /etc/ssh/ssh_config
/etc/init.d/ssh restart
Gitlab、Gerrit、Jenkins 版本和下载地址

Gitlab 6.4 Stable
Gerrit 2.8 War
Jenkins 1.544 Deb
设置主机名

hostname www.thstack.com
sysctl -w kernel.hostname=www.thstack.com
echo www.thstack.com > /etc/hostname
echo '192.168.8.224 www.thstack.com www' >> /etc/hosts
注册四个邮箱账号，其中 admin@thstack.com 用来管理三套系统

admin@thstack.com
gitlab@thstack.com
gerrit@thstack.com
jenkins@thstack.com
规划三个域名和端口

gitlab.thstack.com:8081
review.thstack.com:8082
jenkins.thstack.com:8083
设置解析

vim /etc/hosts
192.168.8.224 gitlab.thstack.com gitlab
192.168.8.224 review.thstack.com review
192.168.8.224 jenkins.thstack.com jenkins
简单规划完后就可以动手去搭建了，后面几篇文章会记录安装的过程。

二. GitLab 的安装配置

上一篇文章 CI 系统搭建：一. 基础环境设置、规划 大概规划了下环境，本文主要用来记录安装 Gitlab 的过程，主要参考官方文档 并没有做太多的修改。


目录

1 设置源
2 安装依赖包
3 系统用户
4 GitLab Shell
5 Mysql
6 GitLab
7 Nginx
8 界面简单使用
设置源

设置国内 163 apt 源

# vim /etc/apt/sources.list
deb http://mirrors.163.com/ubuntu/ precise main universe restricted multiverse
deb http://mirrors.163.com/ubuntu/ precise-security universe main multiverse restricted
deb http://mirrors.163.com/ubuntu/ precise-updates universe main multiverse restricted
deb http://mirrors.163.com/ubuntu/ precise-proposed universe main multiverse restricted
deb http://mirrors.163.com/ubuntu/ precise-backports universe main multiverse restricted

deb-src http://mirrors.163.com/ubuntu/ precise main universe restricted multiverse
deb-src http://mirrors.163.com/ubuntu/ precise-security universe main multiverse restricted
deb-src http://mirrors.163.com/ubuntu/ precise-proposed universe main multiverse restricted
deb-src http://mirrors.163.com/ubuntu/ precise-backports universe main multiverse restricted
deb-src http://mirrors.163.com/ubuntu/ precise-updates universe main multiverse restricted
# apt-get update
安装依赖包

Gitlab 依赖包、库

sudo apt-get install -y build-essential zlib1g-dev libyaml-dev libssl-dev libgdbm-dev libreadline-dev \
                        libncurses5-dev libffi-dev curl openssh-server redis-server checkinstall \
                        libxml2-dev libxslt-dev libcurl4-openssl-dev libicu-dev logrotate
安装 markdown 文档风格依赖包

sudo apt-get install -y python-docutils 
安装 git，Gerrit 依赖 gitweb，同时 GitLab 依赖 git 版本 >= 1.7.10，Ubuntu apt-get 默认安装的是 1.7.9.5，当然不升级也是没有问题的

sudo apt-get install -y git git-core gitweb git-review
升级 Git 版本（可选）

sudo apt-get install -y git gitweb
sudo apt-get remove git-core
sudo apt-get install -y libcurl4-openssl-dev libexpat1-dev gettext libz-dev libssl-dev build-essential
cd /tmp
curl --progress https://git-core.googlecode.com/files/git-1.8.4.1.tar.gz | tar xz
cd git-1.8.4.1/
make prefix=/usr/local all
sudo make prefix=/usr/local install
# * 如果升级了 git 的版本，相应修改 Gitlab.yml 中的 git 脚本位置，这一步在 clone gitlab 后在操作*
sudo -u git -H vim /home/git/gitlab/config/gitlab.yml
bin_path: /usr/local/bin/git
Gitlab 需要收发邮件，安装邮件服务器

sudo apt-get install -y postfix
如果安装了 ruby1.8，卸载掉，Gitlab 依赖 2.0 以上

sudo apt-get remove ruby1.8
下载编译 ruby2.0

mkdir /tmp/ruby && cd /tmp/ruby
curl --progress ftp://ftp.ruby-lang.org/pub/ruby/2.0/ruby-2.0.0-p353.tar.gz | tar xz
cd ruby-2.0.0-p353
./configure --disable-install-rdoc
make
sudo make install
修改 gem 源指向 taobao

gem source -r https://rubygems.org/
gem source -a http://ruby.taobao.org/
安装 Bundel 命令

sudo gem install bundler --no-ri --no-rdoc
系统用户

给 Gitlab 创建一个 git 用户

sudo adduser --disabled-login --gecos 'GitLab' git
GitLab Shell

下载 Gitlab Shell，用来 ssh 访问仓库的管理软件

cd /home/git
sudo -u git -H git clone https://github.com/gitlabhq/gitlab-shell.git
cd gitlab-shell
sudo -u git -H cp config.yml.example config.yml
修改 gitlab-shell/config.yml

sudo -u git -H vim /home/git/gitlab-shell/config.yml
gitlab_url: "http://gitlab.thstack.com/"
安装 GitLab Shell

cd /home/git/gitlab-shell
sudo -u git -H ./bin/install
Mysql

安装 Mysql 包

sudo apt-get install -y mysql-server mysql-client libmysqlclient-dev
给 Gitlab 创建 Mysql 数据库并授权用户访问

sudo mysql -uroot -p
> create database gitlabdb;
> grant all on gitlabdb.* to 'gitlabuser'@'localhost' identified by 'gitlabpass';
GitLab

下载 GitLab 源代码，并切换到最新的分支上

cd /home/git
sudo -u git -H git clone https://github.com/gitlabhq/gitlabhq.git gitlab
cd gitlab
sudo -u git -H git checkout 6-4-stable
配置 GitLab，修改 gitlab.yml，其中 host： 项和 gitlab-shell 中 gitlab_url 的主机一致

cd /home/git/gitlab
sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml
sudo -u git -H vim config/gitlab.yml
host: gitlab.thstack.com
email_from: gitlab@thstack.com
support_mail: gitlab@thstack.com
signup_enabled: true             ＃开启用户注册
创建相关目录

cd /home/git/gitlab
sudo -u git -H mkdir tmp/pids/
sudo -u git -H mkdir tmp/sockets/
sudo -u git -H mkdir public/uploads 
sudo -u git -H mkdir /home/git/repositories
sudo -u git -H mkdir /home/git/gitlab-satellites
修改相关目录权限

sudo chown -R git:git log/ tmp/
sudo chmod -R u+rwX  log/ tmp/ public/uploads
修改 unicorn.rb 监听端口为：8081

cd /home/git/gitlab/
sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb
sudo -u git -H cp config/unicorn.rb.example config/unicorn.rb
sudo -u git -H vim config/unicorn.rb
listen "gitlab.thstack.com:8081", :tcp_nopush => true
配置 GitLab 访问 mysql 数据库设置

cd /home/git/gitlab/
sudo -u git cp config/database.yml.mysql config/database.yml
sudo -u git -H vim config/database.yml  
*修改 Production 部分:*
production:
  adapter: mysql2
  encoding: utf8
  reconnect: false
  database: gitlabdb
  pool: 10
  username: gitlabuser
  password: "gitlabpass"
  host: localhost
  socket: /var/run/mysqld/mysqld.sock
设置 GitLab 使用指定邮箱发送邮件，注意 production.rb 的文件格式，开头空两格

cd /home/git/gitlab/
sudo -u git -H vim config/environments/production.rb
  #修改 :sendmail 为 :smtp
  config.action_mailer.delivery_method = :smtp
  config.action_mailer.smtp_settings = {
    :address              => "smtp.googlemail.com",
    :port                 => 587,
    :domain               => 'thstack.com',
    :user_name            => 'gitlab@thstack.com',
    :password             => 'password',
    :authentication       =>  :plain,
    :enable_starttls_auto => true
  }
end          # 上面内容加入到 end 里面
安装 gem

修改 Gemfile 文件中源指向为 taobao

cd /home/git/gitlab/
sudo -u git -H vim Gemfile
source "http://ruby.taobao.org/"
虽然在文件中指向了国内 taobao 源，但是依然会卡一会，耐心等待…

cd /home/git/gitlab/
sudo -u git -H bundle install --deployment --without development test postgres aws
初始化数据库并激活高级功能

cd /home/git/gitlab/
sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production
输入 yes 来初始化数据库、创建相关表，最后会输出 GitLab Web 管理员用来登录的账号和密码

Do you want to continue (yes/no)? yes
...
Administrator account created:
login.........admin@local.host
password......5iveL!fe
设置 GitLab 启动服务

cd /home/git/gitlab/
sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab
sudo update-rc.d gitlab defaults 21
设置 GitLab 使用 Logrotate 备份 Log

cd /home/git/gitlab/
sudo cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab
检查GitLab及其环境的配置是否正确：

cd /home/git/gitlab/
sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production
System information
System:         Ubuntu 12.04
Current User:   git
Using RVM:      no
Ruby Version:   2.0.0p353
Gem Version:    2.0.14
Bundler Version:1.3.5
Rake Version:   10.1.0

GitLab information
Version:        6.4.2
Revision:       214a013
Directory:      /home/git/gitlab
DB Adapter:     mysql2
URL:            http://gitlab.thstack.com
HTTP Clone URL: http://gitlab.thstack.com/some-project.git
SSH Clone URL:  git@gitlab.thstack.com:some-project.git
Using LDAP:     no
Using Omniauth: no

GitLab Shell
Version:        1.8.0
Repositories:   /home/git/repositories/
Hooks:          /home/git/gitlab-shell/hooks/
Git:            /usr/bin/git
启动 GitLab 服务

/etc/init.d/gitlab restart
Shutting down both Unicorn and Sidekiq.
GitLab is not running.
Starting both the GitLab Unicorn and Sidekiq..
The GitLab Unicorn web server with pid 17771 is running.
The GitLab Sidekiq job dispatcher with pid 17778 is running.
GitLab and all its components are up and running
最后编译一下

sudo -u git -H bundle exec rake assets:precompile RAILS_ENV=production
Nginx

安装 Nginx 包

sudo apt-get install -y nginx
配置 Nginx

cd /home/git/gitlab
sudo cp lib/support/nginx/gitlab /etc/nginx/sites-available/gitlab
sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/gitlab
sudo vim /etc/nginx/sites-available/gitlab
listen *:80 default_server;
server_name gitlab.thstack.com;
proxy_pass http://gitlab.thstack.com:8081;
启动 Nginx

/etc/init.d/apache2 stop
/etc/init.d/nginx restart
访问

用浏览器访问: http://gitlab.thstack.com
用户名：admin@local.host
密码：5iveL!fe
2: https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/insta llation.md

界面简单使用

使用 admin@local.host 账号登录： gitlab-login

登录后系统让修改密码，先修改密码： gitlab-set_pass

修改完密码后，用新密码登录一下，然后修改 Admin 用户得邮箱地址为：admin@thstack.com gitlab-set-email

点击保存更改后，系统会自动给刚才输入的邮箱地址发送一封确认修改信息，点击邮件内容中的连接后会自动用新账号邮箱登录。

创建一个 GROUP：

gitlab-click-group

输入 Group 名字为：DevGroup gitlab-create-gourp

点击创建项目： gitlab-click-project

创建一个项目，名为：OpenStack，这个项目属于 DevGroup: gitlab-create-project

创建完项目后，点击添加 ssh 密钥： gitlab-click-sshkey

生成 admin@thstack.com 邮箱密钥： gitlab-gen-sshkey

在界面输入刚才生成得密钥： gitlab-add-sshkey

再注册一个账号，登录页面点击注册按钮： gitlab-click-user

注册一个 Longgeek 用户，输入邮箱、密码，然后去输入得邮箱验证： gitlab-create-user

创建一个 Longgeek 用户，并生成密钥： gitlab-longgeek-gen-sshkey

用 longgeek@thstack.com 用户登录 GitLab，添加刚才生成得密钥到 sshkey 里： gitlab-longgeek-add-sshkey

用 admin@thstack.com 用户登录 GitLab，把 longgeek@thstack.com 添加到 DevGroup 组中，权限是 Reporter，这样 longgeek 用户也可以访问 OpenStack 这个项目，不过没有权限直接 Push： gitlab-add-longgeek-gourp

用 longgeek@thstack.com 用户登录就可以看到 OpenStack 项目： gitlab-longgeek-view-project

用 longgeek@thstack.com clone 项目，尝试 push： gitlab-longgeek-try-push

很显然 longgeek 用户是没有 push 到 master 分支得权限。接下来会安装 Gerrit、Jenkins。以及它们三个如何整合成流程。请参考后面得文章。

三. Gerrit 的安装配置

重新安装 gerrit 需要特别注意：

如果有项目在使用记得备份 /etc/gerrit/git 目录
rm -fr /etc/gerrit
mysql -uroot -p
>>>drop database gerritdb;
>>>create database gerritdb;
重新安装吧。
如果 /etc/gerrit/logs/error_log 里出现 java.lang.IllegalStateException: Missing project All-Projects 这个错误，那就重新安装下，记得清除一下 db。
之前写过两篇文章，CI 系统搭建：一. 基础环境设置、规划、CI 系统搭建：二. GitLab 的安装配置，本文主要记录 Gerrit 的安装的过程。


目录

1 下载 Gerrit 包
2 安装 Gerrit 依赖
3 创建 Gerrit 数据库
4 开始安装
5 Nginx
6 访问
7 最后
下载 Gerrit 包

目前最新版为 2.8

wget http://gerrit-releases.storage.googleapis.com/gerrit-2.8.war 
安装 Gerrit 依赖

Gerrit 的包是 java 格式，需要安装 jre

apt-get install default-jre daemon
创建 Gerrit 数据库

上一篇再安装 GitLab 时已经安装了 Mysql，直接创建库

sudo mysql -uroot -p
> create database gerritdb;
> grant all on gerritdb.* to 'gerrituser'@'localhost' identified by 'gerritpass';
开始安装

把 gerrit 安装再 /etc/gerrit/ 下

# java -jar gerrit-2.8.war init -d /etc/gerrit/

*** Gerrit Code Review 2.8
***

Create '/etc/gerrit'           [Y/n]? y

*** Git Repositories
***

Location of Git repositories   [git]:

*** SQL Database
***

Database server type           [h2]: mysql

Gerrit Code Review is not shipped with MySQL Connector/J 5.1.21
**  This library is required for your configuration. **
Download and install it now [Y/n]? y
Downloading http://repo2.maven.org/maven2/mysql/mysql-connector-java/5.1.21/mysql-connector-java-5.1.21.jar ... OK
Checksum mysql-connector-java-5.1.21.jar OK
Server hostname                [localhost]:
Server port                    [(mysql default)]:
Database name                  [reviewdb]: gerritdb
Database username              [root]: gerrituser
gerrituser's password          :
              confirm password :

*** User Authentication
***

Authentication method          [OPENID/?]: http
Get username from custom HTTP header [y/N]? n
SSO logout URL                 :

*** Email Delivery
***

SMTP server hostname           [localhost]: smtp.googlemail.com
SMTP server port               [(default)]: 587
SMTP encryption                [NONE/?]: tls
SMTP username                  [root]: gerrit@thstack.com
review@thstack.com's password  :
              confirm password :

*** Container Process
***

Run as                         [root]:
Java runtime                   [/usr/lib/jvm/java-6-openjdk-amd64/jre]:
Copy gerrit-2.8.war to /etc/gerrit/bin/gerrit.war [Y/n]? y
Copying gerrit-2.8.war to /etc/gerrit/bin/gerrit.war

*** SSH Daemon
***

Listen on address              [*]:
Listen on port                 [29418]:

Gerrit Code Review is not shipped with Bouncy Castle Crypto v144
  If available, Gerrit can take advantage of features
  in the library, but will also function without it.
Download and install it now [Y/n]? y
Downloading http://www.bouncycastle.org/download/bcprov-jdk16-144.jar ... OK
Checksum bcprov-jdk16-144.jar OK
Generating SSH host key ... rsa... dsa... done

*** HTTP Daemon
***

Behind reverse proxy           [y/N]? y
Proxy uses SSL (https://)      [y/N]? n
Subdirectory on proxy server   [/]:
Listen on address              [*]:
Listen on port                 [8081]: 8082
Canonical URL                  [http://www.thstack.com/]: http://review.thstack.com/

*** Plugins
***

Install plugin reviewnotes version v2.8 [y/N]? y
Install plugin download-commands version v2.8 [y/N]? y
Install plugin replication version v2.8 [y/N]? y
Install plugin commit-message-length-validator version v2.8 [y/N]? y

Initialized /etc/gerrit
Executing /etc/gerrit/bin/gerrit.sh start
Starting Gerrit Code Review: OK
Waiting for server on review.thstack.com:80 ... OK
Opening http://review.thstack.com/#/admin/projects/ ...FAILED
Open Gerrit with a JavaScript capable browser:

http://review.thstack.com/#/admin/projects/

在安装完后 Gerrit 默认会打开浏览器，由于我的系统是 Server 版没有桌面和浏览器，所以才会出现上面倒数第三行的 …FAILED 同时从上面的信息看出我使用了 http 方式验证、启用代理服务器，指定了 Gerrit 端口为 8082，URL 为 review.thstack.com。需要在 nginx 里设置下端口转发。

Gerrit 启动脚本

cp /etc/gerrit/bin/gerrit.sh /etc/init.d/gerrit
vim /etc/init.d/gerrit
GERRIT_SITE=/etc/gerrit/       # 在代码 47 行增加
sudo update-rc.d gerrit defaults 21
service gerrit restart
Nginx

修改 Nginx 配置文件，给 Gerrit 做端口转发和访问控制，把下面内容写入到文件最后

# vim /etc/nginx/sites-available/gitlab
server {
  listen *:80;
  server_name review.thstack.com;
  allow   all;
  deny    all;
  auth_basic "THSTACK INC. Review System Login";
  auth_basic_user_file /etc/gerrit/etc/htpasswd.conf;

  location / { 
    proxy_pass  http://review.thstack.com:8082;
  }   
}
创建 htpasswd.conf 文件，并添加 admin 用户、密码到文件中

# touch /etc/gerrit/etc/htpasswd.conf
# htpasswd /etc/gerrit/etc/htpasswd.conf admin
  New password: 
  Re-type new password: 
  Adding password for user admin
默认第一个登录 Gerrit 的用户是 Admin。

访问

在浏览器 url 输入：http://review.thstack.com/，记得添加 hosts 解析。由于做了访问控制，会出现如下验证框，输入刚才创建的 admin 和密码 gerrit-admin-access

注册 admin 的邮箱，并添加 admin@thstack.com 密钥 gerrit-admin-sshkey

使用 htpasswd 创建 longgeek 用户和密码

htpasswd /etc/gerrit/etc/htpasswd.conf longgeek
换个浏览器或清除下浏览器保存的 admin 的密码 访问 http://review.thstack.com/ 输入 longgeek 和密码： gerrit-longgeek-login

注册邮箱和添加 longgeek@thstack.com 的密钥 gerrit-longgeek-create-user

最后

如果想 Gitlab 上创建的项目使用 Gerrit 的 Code Review 功能，两个系统的用户必须统一，也就是说不管哪个用户使用 Gerrit，前提是这个用户在 Gitlab 和 Gerrit 上都已注册，邮箱一致、sshkey 一致。当然 Nginx 访问控制用户的密码那就随意了。至于 Gitlab 上创建的项目如何同步到 Gerrit 上、Gitlab 如何使用 Gerrit 的 Code Review 功能等等都在 Jenkins 安装完之后会一起整合在一起。请关注后面的文章。

四. Jenkins 的安装配置


本文作为之前几篇文章的延续，在动手安装 Jenkins 之前，确保参考了如下文章:

CI 系统搭建：一. 基础环境设置、规划
CI 系统搭建：二. GitLab 的安装配置
CI 系统搭建：三. Gerrit 的安装配置
目录

1 Jenkins
2 下载包
3 Jenkins
4 Nginx
5 访问
Jenkins

官网上简单介绍 Jenkins 是一个可扩展的开源持续集成服务器，可扩展 Jenkins 系统集群方式、大量的插件方式扩展，显然 Jenkins 自己已经形成了一个小生态圈。还有一点不得不说的是 Jenkins 本身的迭代开发非常猛，看了下版本发布周期平均一周一个版本。具体的 Jenkins 介绍可以去看 IBM developerWorks 的文章。

下载包

在写这篇文章时候，Jenkins 最新版本为 1.544

wget http://pkg.jenkins-ci.org/debian/binary/jenkins_1.544_all.deb
Jenkins

安装依赖包

apt-get install daemon
安装 Jenkins

dpkg -i jenkins_1.544_all.deb
jenkins 默认监听了 8080 端口，修改为 8083

vim /etc/default/jenkins
HTTP_PORT=8083
重新 jenkins 服务

/etc/init.d/jenkins restart
Nginx

配置 Nginx 端口转发，在文件末尾加入下面配置

# vim /etc/nginx/sites-available/gitlab
server {
  listen *:80;
  server_name jenkins.thstack.com;

  location / {
    proxy_pass  http://jenkins.thstack.com:8083;
  }
}
重启 Nginx，就可以用 jenkins.thstack.com 访问 Jenkins 了

/etc/init.d/nginx restart
访问

http://jenkins.thstack.com/ jenkins-dashboard

开启用户注册功能，点击 -> 系统管理 -> Configure Global Security -> 勾上启用安全，就可以看到下图 jenkins-user-set

保存后，会自动跳转到登录页面，点击右上角注册按钮 jenkins-user-restryges

输入管理员信息 jenkins-create-admin

为了安全，设置 Jenkins 不对普通用户开放登录权限，只有管理员可以设置、构建任务，普通用户可以查看任务状态 点击 系统管理 -> Configure Global Security -> 去掉开放用户注册勾 jenkins-disable-user-siup

接下来要安装 Jenkins 的插件，来支持 Gerrit 点击 系统管理 -> 管理插件，会看到下图显示 jenkins-plugin

如果上图种 可更新、可选插件、已安装 三个菜单点开为空白的话，需要获取更新下 Jenkins 的信息，之后就可以看到插件信息了 点击 系统管理 -> 管理插件 -> 高级 jenkins-update-info

安装 Gerrit Trigger 插件 点击 系统管理 -> 管理插件 -> 可选插件 -> “Gerrit Trigger” jenkins-select-gerrit

安装完 Gerrit Trigger 插件重启 Jenkins，进度掉如果卡在 Gerrit Trigger 最后时，尝试刷新下页面 jenkins-install-gerrit

在系统中给 Jenkins 用户生成 ssh 密钥，Jenkins 用户在安装包的时候自动创建了，家目录在 /var/lib/jenkins

su - jenkins
ssh-keygen -C jenkins@thstack.com
cat ~/.ssh/id_rsa.pub         # 把公钥内容复制一下，后面需要添加到 Gerrit 中
用 htpasswd 创建 jenkins 用户访问控制密码

htpasswd /etc/gerrit/etc/htpasswd.conf jenkins
继续换个浏览器或者清除浏览器记录，用 jenkins 用户访问 Gerrit
http://review.thstack.com jenkins-review-login

设置 jenkins 用户邮箱，顺便去邮箱里确认 jenkins-jenkins-mail

添加刚才对 jenkins@thstack.com 邮箱生成的 ssh 密钥 jenkins-sshkey

设置 Jenkins 系统 SMTP 点击 -> 系统管理 -> 系统设置 -> 找到邮件通知栏 -> 高级 jenkins-email

回到 Jenkins，设置 Gerrit Trigger 插件，填写刚才在 Gerrit 上注册的 jenkins 用户信息 点击 -> 系统管理 -> Gerrit Trigger -> 填写 Gerrit Server 中的信息，最后点击 Test Connection，会在左侧出现 Success，如果失败检查输入的信息是否有误*

2014.04.05 update:

点击 ‘Test Connection’ 如果出现下面内容，不要着急，继续跟着我的博客往下做。这个问题会在第五篇文章里解决掉。

User jenkins has no capability to connect to Gerrit event stream.
jenkins-gerrit-trigger-ssh

如果上面一步出现下图错误，是因为 jenkins ssh Gerrit 时候需要输入 yes，卡住了 可以手工 ssh 输入一下 也是。在返回界面 多刷新几次就没有错误信息了。 su – jenkins ssh -p 29418 jenkins@review.thstack.com gerrit

jenkins-gerrit-conn-err

由于 Gerrit 的版本为 2.8，Gerrit 在 2.7 后做了些更改，去掉了 gerrit approve 的 approve 命令，而改为 review，所以需要修改 点击 -> 系统管理 -> Gerrit Trigger -> Gerrit Reporting Values -> Code Review -> 点击 “高级” jenkins-gerrit-approve-to-review

修改 gerrit approve 为 gerrit review jenkins-gerrit-reviews

OK，Jenkins 设置完毕，接下来就需要针对项目在 jenkins 上创建构建任务，放在下一篇文章里。

五. GitLab、Gerrit、Jenkins 三者整合

参考之前的文章:

CI 系统搭建：一. 基础环境设置、规划
CI 系统搭建：二. GitLab 的安装配置
CI 系统搭建：三. Gerrit 的安装配置
CI 系统搭建：四. Jenkins 的安装配置

目录

1 Gerrit 和 Jenkins 整合
2 GitLab 上为 openstack 项目的一些准备
2.1 .gitreview
2.2 .testr.conf
3 Gerrit 上为 openstack 项目的一些准备
3.1 在 Gerrit 上创建 openstack 项目
3.2 clone –bare Gitlab 上的仓库到 Gerrit
3.3 同步 Gerrit 的 openstack 项目到 Gitlab 上的 openstack 项目目录中
4 在 Jenkins 上对 openstack 项目创建构建任务
5 提交 Review 任务
Gerrit 和 Jenkins 整合

让 Gerrit 支持 Jenkins，Gerrit 在 2.7 版本后去掉了 ‘lable Verified’，需要自己添加

# cd /tmp
# git init cfg; cd cfg
# git config user.name 'admin'
# git config user.email 'admin@thstack.com'
# git remote add origin ssh://admin@review.thstack.com:29418/All-Projects
# git pull origin refs/meta/config
# vim project.config        # 在文件末尾添加下面 5 行
[label "Verified"]
    function = MaxWithBlock
    value = -1 Fails
    value =  0 No score
    value = +1 Verified
# git commit -a -m 'Updated permissions'
# git push origin HEAD:refs/meta/config
查看 Jenkins 的 log 发现会一直出现下面的信息，是因为 Jenkins 没有权限监听 Gerrit 的 ‘Stream Events’

＃ tail -f /var/log/jenkins/jenkins.log 
Dec 26, 2013 1:16:49 AM com.sonyericsson.hudson.plugins.gerrit.gerritevents.GerritHandler run
INFO: Ready to receive data from Gerrit
Dec 26, 2013 1:16:49 AM com.sonyericsson.hudson.plugins.gerrit.gerritevents.GerritHandler run
INFO: Ready to receive data from Gerrit
在 Gerrit 全局设置中能看到 Gerrit 的 ‘Stream Events’ 动作权限默认对 ‘Non-Interactive Users’ 组开放

gerrit-Non-Interactive-Users

用 admin@thstack.com 用户登录 Gerrit 系统，添加 Jenkins@thstack.com 用户到 ‘Non-Interactive Users’ 组

点击 -> People -> List Groups -> Non-Interactive Users gerrit-add-jenkins-user-in-non-interactive-group

现在提交的 Review 请求只有 Code Rivew 审核，我们要求的是需要 Jenkins 的 Verified 和 Code Review 双重保障，在 Projects 的 Access 栏里，针对 Reference: refs/heads/ 项添加 Verified 功能*

点击 Projects -> Access -> Edit -> 找到 Reference: refs/heads/* 项 -> Add Permission -> Label Verified -> Group Name 里输入 Non-Interactive Users -> Add -> 在最下面点击保存更改 gerrit-add-verified-lable

GitLab 上为 openstack 项目的一些准备

.gitreview

之前在 CI 系统搭建：二. GitLab 的安装配置 文章中在 GitLab 上创建了一个叫 openstack 项目，并设置了权限，普通用户是没有办法去 push 的，只能使用 git review 命令提交. 而 git review 命令需要 .gitreview 文件存在于项目目录里，用 admin@thstack.com 用户添加 .gitreview 文件

git clone git@gitlab.thstack.com:devgroup/openstack.git
cd openstack
git config user.name 'admin'
git config user.email 'admin@thstack.com'
编辑 .gitreview 文件

vim .gitreview
[gerrit]
host=review.thstack.com
port=29418
project=openstack.git
添加到版本库

git add .gitreview
git commit .gitreview -m 'Admin add .gitreview file.'
git push origin master
.testr.conf

Python 代码我使用了 testr，需要先安装 testr 命令

apt-get install python-pip
pip install testrepository
在 openstack 这个项目中添加 .testr.conf 文件

cd openstack
vim .testr.conf
[DEFAULT]
test_command=OS_STDOUT_CAPTURE=1 OS_STDERR_CAPTURE=1 OS_TEST_TIMEOUT=60 ${PYTHON:-python} -m subunit.run discover -t ./ ./ $LISTOPT $IDOPTION
test_id_option=--load-list $IDFILE
test_list_option=—list
提交到版本库中

git add .testr.conf
git commit .testr.conf -m 'Admin add .testr.conf file'
git push origin master
Gerrit 上为 openstack 项目的一些准备

在 Gerrit 上创建 openstack 项目

要知道 review 是在 gerrit 上，而 gerrit 上现在是没有项目的，想让 gitlab 上的项目能在 gerrit 上 review 的话，必须在 gerrit 上创建相同的项目，并有相同的仓库文件.

用 admin 用户在 Gerrit 上创建 openstack 项目

ssh -p 29418 admin@review.thstack.com gerrit create-project openstack
clone –bare Gitlab 上的仓库到 Gerrit

在 Gerrit 上 clone Gitlab 的 openstack 项目

cd /etc/gerrit/git
rm -fr openstack.git
git clone --bare git@gitlab.thstack.com:devgroup/openstack.git
同步 Gerrit 的 openstack 项目到 Gitlab 上的 openstack 项目目录中

当用户 git review 后，代码通过 jenkins 测试、人工 review 后，代码只是 merge 到了 Gerrit 的 openstack 项目中，并没有 merge 到 Gitlab 的 openstack 项目中，所以需要当 Gerrit openstack 项目仓库有变化时自动同步到 Gitlab 的 openstack 项目仓库中。Gerrit 自带一个 Replication 功能，同时我们在安装 Gerrit 时候默认安装了这个 Plugin。现在只需要添加一个 replication.config 给 Gerrit。

添加 replication.config 文件

vim /etc/gerrit/etc/replication.config
[remote "openstack"]
  # Gerrit 上要同步项目的名字      
  projects = openstack
  url = root@gitlab.thstack.com:/home/git/repositories/devgroup/openstack.git
  push = +refs/heads/*:refs/heads/*
  push = +refs/tags/*:refs/tags/*
  push = +refs/changes/*:refs/changes/*
  threads = 3
上面的 url 是用 root 用户来做 Gerrit 的 openstack 项目复制到 Gitlab 的 openstack 项目中，需要免密码登录，生成密钥

ssh-copy-id -i gitlab.thstack.com  # 输入 root 用户密码
设置 ~/.ssh/config

vim ~/.ssh/config
Host gitlab.thstack.com:
    IdentityFile ~/.ssh/id_rsa
    PreferredAuthentications publickey
在 ~/.ssh/known_hosts 中，给 gitlab.thstack.com 添加 rsa 密钥

> ~/.ssh/known_hosts
ssh-keyscan -t rsa gitlab.thstack.com > ~/.ssh/known_hosts
ssh-keygen -H -f ~/.ssh/known_hosts
重新启动 Gerrit 服务

/etc/init.d/gerrit restart
Gerrit 的复制功能配置完毕，在 gerrit 文档中有一个 ${name} 变量用来复制 Gerrit 的所有项目，这里并不需要。如果有多个项目需要复制，则在 replication.config 中添加多个 [remote ....] 字段即可。务必按照上面步骤配置复制功能。

在 Jenkins 上对 openstack 项目创建构建任务

用 admin 用户登录 jenkins，针对 openstack 项目创建构建任务 点击 新建 -> 输入任务名 -> 选中构建一个自由风格的软件项目 jenkins-create-task

选中 ‘git’ -> 在 ‘Repository URL’ 里输入 openstack 项目在 Gerrit 上的地址(在 gerrit 的 project list 中 openstack 后面的 gitweb 里有具体 url，) -> 在 ‘Branches to build’ 里输入 ‘origin/$GERRIT_BRANCH’ jenkins-create-task1

选中 Gerrit event -> 点击 Add 分别添加 ‘Patchset Created’ 和 ‘Draft Published’ －> 在第一个 plain 输入项目名字 openstack，第二个 plain 输入 master -> 点击 增加构建步骤 -> 选择 ‘Execute shell’ -> 写入要测试的脚本 -> 保存 同时也可以选择 ‘添加构建后操作步骤’ 来邮件通知一个群组等功能。 jenkins-create-task2

提交 Review 任务

切换到 longgeek 用户，之前创建了一个 longgeek 用户，方便测试

su - longgeek
cd openstack   # 之前添加过一个 testfile 文件，push 没有权限，所以用 git review 来代替 git push
git push
git review
Creating a git remote called gerrit that maps to:
    ssh://longgeek@review.thstack.com:29418/openstack.git
Your change was committed before the commit hook was installed
Amending the commit to add a gerrit change id
remote: Processing changes: new: 1, refs: 1, done    
remote: 
remote: New Changes:
remote:   http://review.thstack.com/1
remote: 
To ssh://longgeek@review.thstack.com:29418/openstack.git
* [new branch]      HEAD -> refs/for/master/master
用 admin 用户登录 http://review.thstack.com 就可以看到 longgeek 用户提交的 reivew 请求 gerrit-review

可以看到 jenkins 已经通过，并打上了 ‘绿勾’ 标记，接下来 admin 用户 ＋2 并提交就可以 Merge 代码 gerrit-review-submit

在 Merge 代码后，Gerrit 会自动同步到 Gitlab 上，如下图 gitlab-replication-ok

同步后，使用 git pull 命令就可以从 Gitlab 上的 openstack 仓库下来代码

cd openstack
git pull
From gitlab.thstack.com:devgroup/openstack
   70c59ff..7948aae  master     -> origin/master
Already up-to-date.
如此便是一个完整的 CI 体系流程了，剩下的最重要的是如何用、真正的用起来。