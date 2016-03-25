本地pypi搭建过程（待更新）
========================

简介
---
此次搭建主要使用了Flask-Pypi-Proxy。原理为设置一个pip源的代理，之后在安装pip包时，第一次会从代理的源上拉取资源，之后会缓存到本地。之后的安装就会使用本地的pip包来安装。

搭建本地pypi过程：
---------------

1.安装所必须的软件包，因为使用的Flask-pypi-Proxy,所以需要安装跟其相关的包。

	pip install flask_pypi_proxy
	yum install httpd httpd-devel -y

2.安装http的wsgi模块，wsgi模块可以用来配置代理相关设置。
首先需要下载mod_wsgi-3.4c1.tar.gz,解压并编译安装。
在编译时可能出现python库的问题，解决方法为：拷贝libboost_python.a. 到/usr/local/lib/python2.7/config 更名并替换原来的libpython2.7.a 重新编译即可。

3.创建pypi-proxy用户，在其家目录下。创建eggs,envs,logs目录和pypi-proxy.wsgi文件分别用来存放环境配置，本地pip包，日志文件，已经代理配置。

4.编写pypi-proxy.wsgi（自己内部的pip包需要配置private_egg）

	import os

	os.environ['PYPI_PROXY_BASE_FOLDER_PATH'] = '/home/pypi-proxy/eggs/'
	os.environ['PYPI_PROXY_LOGGING_PATH'] = '/home/pypi-proxy/logs/proxy.log'
	os.environ['PYPI_PROXY_PYPI_URL'] = 'http://pypi.douban.com'

	# if installed inside a virtualenv, then do this:
	activate_this = '/home/pypi-proxy/envs/proxy/bin/activate_this.py'
	execfile(activate_this, dict(__file__=activate_this))

	from flask_pypi_proxy.views import app as application

注：以上配置了本地pip包的存放路径，log存放路径，代理的源的url。

5.配置http，调通代理

主要在/etc/httpd/conf的httpd.conf文件中

	<VirtualHost *:80>
		ServerName pypi.xiaohongshu.com

		WSGIDaemonProcess pypi_proxy threads=5
		WSGIScriptAlias / /home/pypi-proxy/pypi-proxy.wsgi

		<Directory /home/pypi-proxy>
			Order deny,allow
			Allow from all
		</Directory>

注：以上定义了虚拟主机端口，servername，wsgi文件路径。通过读取次文件，打通和源的通道。

pypi服务器的管理
---------------

1.关于上传包的配置
首先在客户端的家目录下创建.pypirc文件，内容如下：
	[distutils]
	index-servers =
		red

	[red]
	username:foo
	password:bar
	repository:http://pypi.xiaohongshu.com/pypi/

2.上传
	执行 python setup.py sdist upload -r red 

3.在server端加入private_egg.具体操作如下：
	cd /home/pypi-proxy/
	向pypi-proxy.wsgi的private_egg的变量中插入包的名字即可：
	os.environ['PYPI_PROXY_PRIVATE_EGGS'] = 'lumos,mongoengine,red_cloud_image'


pip的下载
---------

类似如下：
	pip install lumos==1.0.4 -i http://pypi.xiaohongshu.com/simple --trusted-host pypi.xiaohongshu.com
