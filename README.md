## 使用uWSGI和Nginx部署Django	
***

### 0x00 何为WSGI、uwsgi、uWSGI?
* [WSGI](http://www.nowamagic.net/academy/detail/1330310)是一种Web服务器网关接口`协议`，是基于现存的 CGI 标准而设计的。它是一个Web服务器（如Nginx）与应用服务器（如uWSGI服务器）通信的一种规范，即它是一种通信协议。  
<br /> 
*  [uwsgi](http://uwsgi-docs.readthedocs.io/en/latest/Protocol.html)是一个uWSGI服务器自有的`协议`，它用于定义传输信息的类型（type of information），每一个uwsgi packet前4byte为传输信息类型描述，它与WSGI相比是两样东西，但都是通信协议。
<br /> 
* [uWSGI](http://flask.pocoo.org/docs/0.10/deploying/uwsgi/)是一个`Web服务器`，它实现了WSGI、uwsgi、FastCGI、HTTP等协议。Nginx中HttpUwsgiModule的作用是与uWSGI服务器进行交换。

### 0x01 何为Nginx?
* [Nginx](https://zh.wikipedia.org/wiki/Nginx)是一款面向性能设计的`轻量级HTTP服务器`，它能反向代理HTTP,HTTPS, SMTP, POP3,IMAP的协议链接，以及提供一个负载均衡器和一个HTTP缓存。
<br />
* 相较于Apache、lighttpd具有`占有内存少`，`稳定性高`等[优势](http://lnmp.org/nginx.html)。
<br /> 
* 它不采用每客户机一线程的设计模型，而是充分使用异步逻辑，削减了上下文调度开销，所以`并发服务能力更强`。
***

### 0x02 为什么有了uWSGI为什么还需要Nginx？
* 相对来说，Nginx具备优秀的`静态内容处理能力`，因而更加适合取处理静态文件，如css、javascript、png、html等。
<br /> 
* 而`动态的文件`最好由Nginx转交给uWSGI去进行处理，这样可以达到很好的客户端响应。
<br /> 
* 基础页面输出的模型：
> 英文描述：`web client <-> web server <-> socket <-> uwsgi <-> Django`
> 中文描述：`网页客户端<->网页服务器<->网络套接字<->uwsgi<-> Django应用`


***
### 0x03 环境准备
服务器:ubuntu,拥有root权限  

`sudo apt-get update`

Nginx安装:  

`sudo apt-get install nginx`

uWSGI安装:  

`sudo apt-get install python2.7-dev`  
`sudo pip install uwsgi`  

Django安装:  

`sudo pip install django==1.6.1`（此处填入要安装的Django版本）  
有时候下载非常缓慢或者无法连接，可以尝试使用豆瓣源:  
`sudo pip install django==1.6.1 -i http://pypi.douban.com/simple`

Mysql安装：

`sudo apt-get install mysql-server`
如果是已经安装好，却忘记root密码，如下找回:
> 1. 修改MySQL的登录设置：`sudo vim /etc/mysql/my.cnf`
> 2. 在[mysqld]的段末尾加上一句：`skip-grant-tables`，保存并退出
> 3. 重新启动mysqld ：`sudo /etc/init.d/mysqld restart`  ( sudo service mysql restart )
> 4. 登录并修改MySQL的root密码:`mysql` -> `USE mysql` -> `UPDATE user SET Password = password ( 'new-password' ) WHERE User = 'root'; `
> 5. 将刚才在[mysqld]的段中加上的skip-grant-tables删除 
> 6. 重新启动mysqld.

***
### 0x04 Django应用设置
以本次部署使用的wslcore.tar.gz为例，解压到/home/wsl/下：  

`tar zxvf wslcore.tar.gz -C /home/wsl/`  
`cd /home/wsl/wslcore/`

#### 设置连接Mysql数据库信息:
```
# wslcore/settings/common.py
# 特别提醒，数据库用户密码必须为非空，否则无法连接
DATABASES = {
    'default': {
       'ENGINE': 'django.db.backends.mysql',
       'NAME': 'wslcore', #数据库名称
       'USER': 'root',
       'PASSWORD': 'YOUR_PASSWORD',
    }
}
```
#### 同步数据库:
1. 必须先在数据库内创建数据库:  
* `mysql -u root -p`  
* `set names utf8`
* `CREATE DATABASE wslcore CHARACTER SET 'utf8' COLLATE 'utf8_general_ci';`  
(特别提醒，本例中使用Django1.6.1版本时的syncdb同步功能，不会自动设置数据库及其内表的编码格式，所以手动创建数据库的时必须生声明编码格式。如果是高版本，则可能可以不用这样声明，只需简单建库即可)
* `exit`  

2. 同步:
* `python manage.py syncdb`
* 对于1.7.0以上版本使用
`python manage.py makemigrations`
`python manage.py migrate`

#### 配置定时任务:
1. 题外话，扩展manage.py功能：
* 在django包目录`/usr/local/lib/python2.7/dist-packages/django/core/management/commands`中添加如下文件`mycommand.py`，就可以使用`python manage.py mycommand`执行handle()中的代码。
* 参考:http://www.cnblogs.com/linjiqin/p/3965046.html  
```
from django.core.management.base import BaseCommand   
class Command(BaseCommand):
    def handle(self, *args, **options):         
        print "hello world"
```
2. 使用django-crontab插件：
* `sudo pip install django-crontab`  (import时候名字为django_crontab)  
* 将django_crontab加入到settings.py的INSTALLED_APPS即可，例如：
INSTALLED_APPS = (
......，
'django_crontab',
......，)
* 配置调用时间和调用函数，例如
CRONJOBS = [
    ('*/60 * * * *', 'wslcore.apps.pocdb.crawl.crawl_exploitdb',), ]
表示每隔60分钟，调用项目中pocdb文件夹下crawl.py文件中的crawl_exploitdb()函数
* 任务加载或结束
`python manage.py crontab add`即可加载任务
`python manage.py crontab remove` 即可移除任务
可以用`crontab -u SYSTEM_USERNAME -l`查看当前任务
可以用`crontab -u SYSTEM_USERNAME -r`结束当前任务
* 参考：
[官方说明及详细定时规则](https://github.com/kraiz/django-crontab)
[django-crontab实现Django定时任务](http://www.zhidaow.com/post/django-crontab)
[利用django-crontab来部署系统cron jobs](http://my-first-read-the-docs.readthedocs.io/en/latest/django_crontab.html)

***

### 0x05 Nginx、uWSGI测试环境
#### 1. web client <-> uWSGI <-> python 连通性测试:
在工程根目录下创建test.py，内容如下：
```
def application(env, start_response):
	start_response('200 OK', [('Content-Type','text/html')])
	return ["Hello World"] 
```
写入完成后执行以下命令：  

`sudo uwsgi --http :8000 --wsgi-file test.py`  
浏览器访问`http://localhost:8000`，看浏览器是否正常输出""Hello World"  
如果输出正常，表明`web client <-> uWSGI <-> python` 连通性正常  

#### 2. web client <-> uWSGI <-> Django 连通性测试:
在工程根目录下：  

`python manage.py runserver`  
访问`http://127.0.0.1:8000`查看效果  
如果没有问题，表明`web client <-> uWSGI <-> Django` 连通性正常  

#### 3.web client <-> web server 连通性测试---配置Nginx：
启动、停止和重启
```
sudo /etc/init.d/nginx start
sudo /etc/init.d/nginx stop
sudo /etc/init.d/nginx restart
```
或者
```
sudo service nginx start
sudo service nginx stop
sudo service nginx restart
```
启动nigix服务后，打开浏览器，访问127.0.0.1:80，出现如下图示则表明配置成功  
![](http://upload-images.jianshu.io/upload_images/1898272-93e129f9b34a8f57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上述的一切正常，则表明`web client <-> web server` 连通性正常

#### 4. web server <-> socket <-> uWSGI 连通性测试：

在工程根目录下的server_conf/wsl_nginx.conf内容如下：
```
# configuration of the server
server {
    # the port your site will be served on
    listen 8000;
    
    # the domain name it will serve for
    # TODO: change the domain to your own
    server_name 127.0.0.1; # substitute your machine's IP address or FQDN
    charset utf-8;

    # max upload size
    client_max_body_size 75M;   # adjust to taste

    # Django media
    location /media  {
        alias /home/wsl/wslcore/media;  # your Django project's media files - amend as required
    }

    location /static {
        alias /home/wsl/wslcore/static; # your Django project's static files - amend as required
    }

    # Finally, send all non-media requests to the Django server.
    location / {
        uwsgi_pass  127.0.0.1:9191;
        include /home/wsl/wslcore/server_conf/uwsgi_params; # the uwsgi_params file you installed
    }
}
```
<br />
* 该配置告诉Nginx使用8000作为服务的端口，同时提供了static和media的handle处理，路径如上。其他的服务（即动态操作）通过127.0.0.1:9191这个handle服务去处理。
<br />
* 为了让Nginx服务器能识别这个配置，创建一个链接给该文件，如下：
`sudo ln -s /home/wsl/wslcore/server_conf/wsl_nginx.conf /etc/nginx/sites-enabled`
<br />
* 将你网站使用到的静态文件自动搜集起来放到Nginx指定的目录下：
`mkdir media static`
`sudo python manage.py collectstatic`
<br />
* 重启nigix服务，`sudo /etc/init.d/nginx restart`
<br />
* 工程根目录下`sudo uwsgi --socket :9191 --wsgi-file test.py`
<br />
*  通过浏览器访问127.0.0.1:8000，如果能正常显示”Hello World“则表明`web client <-> web server <-> socket <-> uWSGI <-> Python` 配置成功

#### 5. 通过.ini文件去执行uWSGI:
server_conf中的uwsgi.ini：
```
[uwsgi]
# Django-related settings

## the base directory (full path)
chdir = /home/wsl/wslcore

## Django's wsgi file
module = wslcore.wsgi

## master
master = true

## pidfile
pidfile = /home/wsl/wslcore/.tmp/wslcore-master.pid

## maximum number of worker processes
processes = 10

## Socket use tcp/ip
socket = 127.0.0.1:9191

## clear environment on exit
vacuum = true

#run as a daemon
#daemonize = /var/log/uwsgi/yourproject.log
```
配置完成上述的文件和执行`sudo uwsgi --ini server_conf/uwsgi.ini`就可以一键让服务运行起来.  

或者使用工程根目录下提供的脚本`sudo ./run.sh`，其中最后执行的核心部分即为上述命令，所以达到的是同样的效果.  

如果要服务在后台运行，就取消uwsgi.ini末尾行的注释即可，以守护进程的形式运行。 

*** 

如果要结束当前daemonize，并且取消守护进行模式时，可以修改.ini和.conf这两个配置文件中的sock端口和监听端口，以及注释掉daemonize那行，之后再重新执行`sudo run.sh`时，原先的进程将被结束、替换为新建的进程。  

有时候调试时光重启nginx服务是不管用的，此时就需要重启该守护进程，或者取消它、改为命令行下阻塞模式并输出运行信息。

更多使用参考：[uWSGI官方文档](https://uwsgi-docs.readthedocs.io/en/latest/)
