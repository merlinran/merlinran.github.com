---
layout: post
title: 从TurnkeyLinux开始搭建OpenParty应用
---
准备为重庆OpenParty搭建网站，从没接触过Django，按照各方资料捣腾出来了，小记。

首先下载[Django Appliance的VM](http://www.turnkeylinux.org/django)，VM适用于VMWare虚拟机，由于VirtualBox也支持vmdk格式，只需要在VirtualBox里创建虚拟机时选择该vmdk文件即可。

启动VM后会提示你设置root密码、MySQL密码之类，一路往下即可。

然后安装需要的工具

{% highlight sh linenos %}
    sudo apt-get install libmysqlclient-dev libxml2-dev libxslt1-dev  
    sudo apt-get install python-pip python-dev build-essential  
    sudo pip install --upgrade pip  
    sudo pip install --upgrade virtualenv  
{% endhighlight %}

下载OpenParty的源代码

{% highlight sh linenos %}
    git clone https://github.com/merlinran/openparty.git  
{% endhighlight %}

然后建立环境，自动安装依赖的Python包

{% highlight sh linenos %}
    cd openparty  
    virtualenv --no-site-packages .  
    source bin/activate  
    pip install -r requirements  
{% endhighlight %}

因为我的Django是1.4，需要在settings.py中编辑第18行

{% highlight python linenos %}
    DATABASES = {  
        'default': {  
            'ENGINE': 'mysql', # Add 'postgresql_psycopg2', 'postgresql', 'mysql', 'sqlite3' or 'oracle'.
{% endhighlight %}

换成

{% highlight python linenos %}
    'ENGINE': 'django.db.backends.mysql'  
{% endhighlight %}

同时设置好访问MySQL数据库的用户和密码。

然后创建数据库

{% highlight sh linenos %}
    mysql -uroot -p < init_db.sql  
    ./manage.py syncdb  
    ./manage.py migrate core  
{% endhighlight %}

可以启动Web服务啦

{% highlight sh linenos %}
    ./manage.py runserver 0.0.0.0:8000  
{% endhighlight %}
