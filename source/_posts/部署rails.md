---
title: 部署rails
date: 2016.04.27
tags:
---

## 脚本

```sh
# Change source
# 把archive.ubuntu.com改为/etc/apt/sources.list中对应的
# sudo sed -i "s/archive.ubuntu.com/mirrors.163.com/" /etc/apt/sources.list

sudo locale-gen zh_CN.UTF-8
echo "export LC_ALL='zh_CN.UTF-8'" >> ~/.bashrc
source ~/.bashrc
locale

echo "Install Ubuntu packages..."
sudo apt-get install -y liberror-perl imagemagick redis-tools htop nodejs
echo "--------------------------- Install Successed -----------------------------"

echo "Add apt repository"
sudo add-apt-repository ppa:chris-lea/redis-server -y
sudo add-apt-repository ppa:nginx/stable -y
sudo apt-add-repository ppa:git-core/ppa -y
sudo apt-get update
echo "--------------------------- Add Successed -----------------------------"

echo "install Git"
sudo apt-get install git
git --version
echo "--------------------------- Install Successed -----------------------------"

echo "Install Rbenv"
git clone git://github.com/sstephenson/rbenv.git ~/.rbenv
git clone git://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
# 通过 gem 命令安装完 gem 后无需手动输入 rbenv rehash 命令, 推荐
git clone git://github.com/sstephenson/rbenv-gem-rehash.git ~/.rbenv/plugins/rbenv-gem-rehash
# 通过 rbenv update 命令来更新 rbenv 以及所有插件, 推荐
git clone git://github.com/rkh/rbenv-update.git ~/.rbenv/plugins/rbenv-update
# 使用 Ruby China 的镜像安装 Ruby, 国内用户推荐
git clone git://github.com/AndorChen/rbenv-china-mirror.git ~/.rbenv/plugins/rbenv-china-mirror

echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
source ~/.bash_profile

rbenv install 2.2.3
rbenv global 2.2.3
ruby -v
echo "--------------------------- Install Successed -----------------------------"

echo "Install Bundler"
gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/

gem install bundler
bundle config mirror.https://rubygems.org https://gems.ruby-china.org
bundle -v
echo "--------------------------- Install Successed -----------------------------"

echo "Install Redis"
sudo apt-get install redis-server -y
redis-cli -v
echo "--------------------------- Install Successed -----------------------------"

echo "Install Postgres"
sudo apt-get install postgresql postgresql-contrib libpq-dev -y
psql --version
sudo -u postgres createuser -s rails
echo "--------------------------- Install Successed -----------------------------"

echo "Install Nginx"
sudo apt-get install -y nginx
nginx -v
echo "--------------------------- Install Successed -----------------------------"

echo "Install GraphicsMagick"
sudo apt-get install GraphicsMagick -y
sudo cp 目录/skylark/current/bin/vagrant/华文黑体.ttf /usr/share/fonts/truetype/
sudo cp 目录/skylark/current/bin/vagrant/type-windows.mgk /usr/lib/GraphicsMagick-1.3.18/config
echo "--------------------------- Install Successed -----------------------------"
```



Use rbenv instead of rvm.  

```sh
echo "Install RVM"
command gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
command curl -L https://get.rvm.io | bash -s stable
command source ~/.rvm/scripts/rvm

echo "ruby_url=https://cache.ruby-china.org/pub/ruby" > ~/.rvm/user/db

rvm requirements
rvm install 2.2.3
rvm use 2.2.3 --default
rvm -v
ruby -v
echo "--------------------------- Install Successed -----------------------------"
```


## 遇到的问题

### 包的版本问题

- sudo aptitude install libtool

### 使用的Ruby China的ruby镜像

[cache.ruby-china.org](https://cache.ruby-china.org/pub/ruby)  
按照[教程](https://ruby-china.org/topics/29760)  
`echo "ruby_url=https://cache.ruby-china.org/pub/ruby" > ~/.rvm/user/db`

从日志中可以看出并没有从ruby-china去获取

> Found remote file [https://rubies.travis-ci.org/ubuntu/14.04/x86_64/ruby-2.2.3.tar.bz2](https://rubies.travis-ci.org/ubuntu/14.04/x86_64/ruby-2.2.3.tar.bz2)

使用 rvm install 2.2.3 –debug，显示更多的日志，从日志中可以看出，会去先检查 rvm_remote_server_url是否相关文件

> Remote file does not exist [https://rvm_io.global.ssl.fastly.net/binaries/osx/10.11/x86_64/ruby-2.2.3.tar.bz2](https://rvm_io.global.ssl.fastly.net/binaries/osx/10.11/x86_64/ruby-2.2.3.tar.bz2)  
> Remote file does not exist [https://s3.amazonaws.com/jruby.org/downloads/ruby-2.2.3.tar.bz2](https://s3.amazonaws.com/jruby.org/downloads/ruby-2.2.3.tar.bz2)

解决方式（3种）：  
1、把`~/.rvm/config/db`中设置`rvm_remote_server_url`全部注释掉

2、在`~/.rvm/user/db`覆写所有的`rvm_remote_server_url`  
```
ruby_url=https://cache.ruby-china.org/pub/ruby

rvm_remote_server_url= ''
rvm_remote_server_url1= ''
rvm_remote_server_path1= ''
rvm_remote_server_url2= ''
rvm_remote_server_url3= ''
```

3、换成[rbenv](https://github.com/rbenv/rbenv)

- 删除rvm，rvm implode
- 删除 .bash_profile .bashrc .profile中相关rvm代码
- 使用插件`git clone git://github.com/AndorChen/rbenv-china-mirror.git ~/.rbenv/plugins/rbenv-china-mirror`，直接搞定。

### 部署后通过公网ip不能访问

查看 nginx.access.log没有内容  
查看使用的端口, `sudo netstat -peant | grep ":80 "`，没有公网对应的端口  
最后才发现vps默认没开启80端口，心好累。。

### 设置GraphicsMagick字体

`gm convert -list font` 查看可用字体，第一行是font配置的`Path`
打开`Path`文件，在`typemap`下加入`<include file="type-windows.mgk" />`

### 软连接

sudo ln -nfs ‘文件’ ‘目标’

### nginx

sudo service nginx restart  
sudo nginx -t 检查配置文件  
默认配置文件：`/etc/nginx/nginx.conf`，`/etc/nginx/nginx.conf`中include`/etc/nginx/sites-enabled/*`

### chown

改变拥有者和群组 `sudo chown mail:mail log2012.log`
