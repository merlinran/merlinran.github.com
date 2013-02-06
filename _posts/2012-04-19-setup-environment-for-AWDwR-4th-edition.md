---
layout: default
title: 搭建Agile Web Development with Rails (4th edition)的学习环境
---
该书以Rails 3.0.0为基准，建立一致的开发环境有助于避免一些奇奇怪怪的问题打击学习的信心。

假设已经安装了Ruby的最新版本（>1.8.7 or >1.9.2）和RubyGems，现在安装Rails 3.0.0，以前安装过其它Rails版本也没有关系，可以共存。

<pre name="code" class="plain">gem install rails --version=3.0.0 --source http://ruby.taobao.org</pre>

现在，就可以建立一个3.0.0版本的Rails应用。

<pre name="code" class="plain">rails _3.0.0_ new depot -T</pre>

在Gemfile中增加以下内容：

<pre name="code" class="ruby">
source 'http://ruby.taobao.org/' #替换原Gemfile的第一行
group :development, :test do
  gem 'rspec-rails' #安装rspec
end

group :test do
  gem 'factory_girl_rails' # factory girl，自动生成fixture
  gem 'guard-rspec' # 测试自动化
end
</pre>

下一步：

<pre name="code" class="plain">
bundle #更新需要的gem
rails g rspec:install #安装rspec代码，以后每一次rails generate...，会自动生成相应的rspec
bundle exec guard init rspec #为Guard配置Rspec，这样，每当有任何文件改动，会自动运行Rspec测试
</pre>

然后新开几个console，分别运行web server和guard。

<pre name="code" class="plain">rails server guard</pre>

现在，享受学习的乐趣吧。
