---
title: GitHub pages搭建
tags: ["GitHub", "教程"]
---


## 一、创建gitHub仓库

在github上面创建一个名称为`username.github.io`的项目这个username就是github账号的用户名

![image-20191231211058025](http://image.yangyhao.top/blog/Github-pages%E6%90%AD%E5%BB%BA-%E4%B8%80-01.png)

创建完成后点击右侧的settings按钮，找到choose theme选项

![image-20191231212207745](http://image.yangyhao.top/blog/Github-pages%E6%90%AD%E5%BB%BA-%E4%B8%80-02.png)

之后随便选择一个，然后确定提交即可。之后浏览器访问https://`${username}`.github.io，效果如下：

![image-20191231212711312](http://image.yangyhao.top/blog/Github-pages%E6%90%AD%E5%BB%BA-%E4%B8%80-03.png)

现在一个简单的博客已经创建完成。

如果需要后续修改，需要先把github的仓库下载到本地

## 二、域名cname

如果不想要通过username.github.io来访问，可以通过域名利用cname指向，然后通过域名访问；这里以阿里云域名演示，记录类型为**CNAME**,主机记录就是你想访问的二级域名(顶级域名则为www),记录值就是username.github.io，截图如下：

![image-20200101103739213](http://image.yangyhao.top/blog/Github-pages%E6%90%AD%E5%BB%BA-%E4%BA%8C-01.png)

等待域名解析完成后就可以通过域名来访问了。

## 三、安装Ruby

​	上面裸奔的主题，肯定不是我们想要的效果，作为一个熟练使用轮子的程序员，肯定要找到很多轮子，jekyll就是一个很好的轮子；

<!--如果需要本地部署jeykll并扩展则需要往下看，如果只是简单利用则只需要fork其他主题然后改成自己的名字，或者把主题copy到自己的仓库即可。-->

​	[参考文章](https://jekyllrb.com/docs/installation/windows/)

​	jeykll依赖ruby的环境，所以需要先下载ruby     

### 1、下载rubyInstaller

https://rubyinstaller.org/downloads/选择Ruby+Devkit 2.6.5-1 (x64) （网速可能很慢，多试几次）

### 2、安装

（界面上全选）最好不要改安装路径

 安装完后不要勾选ridk install的选项，因为从默认的原去下载几百兆会非常缓慢；

### 3、更新源 

查找Ruby安装目录下的msys64\etc\pacman.d，编辑更新源：

mirrorlist.mingw32

~~~xml
Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/mingw/i686 
~~~

mirrorlist.mingw64

~~~xml
Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/mingw/x86_64 
~~~

mirrorlist.msys

~~~xml
Server = https://mirrors.tuna.tsinghua.edu.cn/msys2/msys/$arch
~~~

然后再msys目录下mingw64.exe里面执行`pacman -Syu`

之后需要安装ridk ，但是由于默认的源非常慢，所以更改ruby的源：

~~~shell
gem sources --add https://mirrors.tuna.tsinghua.edu.cn/rubygems/ --remove https://rubygems.org/
~~~

![image-20191231200835014](http://image.yangyhao.top/blog/Github-pages%E6%90%AD%E5%BB%BA-%E4%B8%89-01.png)

之后在命令行执行ridk install（如果安装时选择了不加入系统环境变量的，去Ruby安装目录的bin之下执行）选择安装3，然后一路回车至结束；

![image-20191231200324378](http://image.yangyhao.top/blog/Github-pages%E6%90%AD%E5%BB%BA-%E4%B8%89-02.png)

![image-20191231201532577](http://image.yangyhao.top/blog/Github-pages%E6%90%AD%E5%BB%BA-%E4%B8%89-03.png)



## 四、安装jekyll

### 1、安装jekyll

~~~shell
gem install jekyll
~~~

(如果遇到问题，大多数是下载ruby不全，没有选择devkit或者网速问题造成的，重新安装即可）

安装完成后运行

~~~shell
jekyll -version
~~~

效果如下：

![image-20191231202445187](http://image.yangyhao.top/blog/Github-pages%E6%90%AD%E5%BB%BA-%E5%9B%9B-01.png)

### 2、找到合适的主题

在[Jekyll Themes](http://jekyllthemes.org/)里面寻找；

笔者这里选择[TeXt](https://github.com/kitian616/jekyll-TeXt-theme)主题

~~~shell
git clone https://github.com/kitian616/jekyll-TeXt-theme.git
~~~

下载完成后把项目拷贝到自己的仓库内

### 3、bundle install

在自己的项目仓库里执行

![image-20200101114017720](http://image.yangyhao.top/blog/Github-pages%E6%90%AD%E5%BB%BA-%E5%9B%9B-02.png)

如果遇到问题：

`ffi requires Ruby version >= 2.0, < 2.6. The current ruby version is 2.6.5.114.`

解决办法：https://github.com/ffi/ffi/issues/598

执行gem install ffi -f然后bundle update

### 4、启动服务

~~~shell
bundle exec jekyll serve
~~~

当出现如下界面时说明已经部署成功

![image-20200101153141559](http://image.yangyhao.top/blog/Github-pages%E6%90%AD%E5%BB%BA-%E5%9B%9B-03.png)

然后浏览器输入127.0.0.1:4000访问即可

![image-20200101153114687](http://image.yangyhao.top/blog/Github-pages%E6%90%AD%E5%BB%BA-%E5%9B%9B-04.png)

### 5、解决中文乱码问题

如果本地访问中文乱码，则解决方式如下：

在`Ruby26-x64\lib\ruby\2.6.0\webrick\httpservlet`里面打开`filehandler.rb`

~~~xml
258行
path = req.path_info.dup.force_encoding(Encoding.find("filesystem"))
				path.force_encoding("UTF-8")
        if trailing_pathsep?(req.path_info)
~~~

~~~xml
333行
break if base == "/"
		  base.force_encoding("UTF-8")  #这里是添加的一行
          break unless File.directory?(File.expand_path(res.filename + base))
~~~

## 五、自定义博客

在本地部署完成以后就可以根据jekyll的语法进行写博客了http://jekyllcn.com/docs/usage/