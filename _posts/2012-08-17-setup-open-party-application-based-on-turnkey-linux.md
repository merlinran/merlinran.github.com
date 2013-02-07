---
layout: post
title: 从TurnkeyLinux开始搭建OpenParty应用
---
准备为重庆OpenParty搭建网站，从没接触过Django，按照各方资料捣腾出来了，小记。

首先下载[Django Appliance的VM](http://www.turnkeylinux.org/django)，VM适用于VMWare虚拟机，由于VirtualBox也支持vmdk格式，只需要在VirtualBox里创建虚拟机时选择该vmdk文件即可。

启动VM后会提示你设置root密码、MySQL密码之类，一路往下即可。

然后安装需要的工具

    sudo apt-get install libmysqlclient-dev libxml2-dev libxslt1-dev  
    sudo apt-get install python-pip python-dev build-essential  
    sudo pip install --upgrade pip  
    sudo pip install --upgrade virtualenv  

下载OpenParty的源代码

    git clone https://github.com/merlinran/openparty.git  

然后建立环境，自动安装依赖的Python包

    cd openparty  
    virtualenv --no-site-packages .  
    source bin/activate  
    pip install -r requirements  

因为我的Django是1.4，需要在settings.py中编辑第18行

    DATABASES = {  
        'default': {  
            <span style="color:#ff0000;">'ENGINE': 'mysql', # Add 'postgresql_psycopg2', 'postgresql', 'mysql', 'sqlite3' or 'oracle'.</span>  

换成

    'ENGINE': 'django.db.backends.mysql'  

同时设置好访问MySQL数据库的用户和密码。

然后创建数据库

    mysql -uroot -p < init_db.sql  
    ./manage.py syncdb  
    ./manage.py migrate core  

可以启动Web服务啦

    ./manage.py runserver 0.0.0.0:8000  
