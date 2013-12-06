---
layout: post
category:  Life
title: My first blog with jekyll
---

# 博客由来
年中在[homezz](http://www.homezz.com/)购买了vps，在[耐思尼克](http://www.iisp.com/)购买域名，搭建了Wordpress博客，日常管理博客需要花费很多精力，而且，一直想尝试用git+github+markdown+jekyll的方式来写作，故搭建此博客。
# 搭建过程
1. apt-get安装rubygems之后，gem install jekyll安装jekyll，并用同样的gem命令安装directory_watcher、liquid、open4、maruku、classifier，rdiscount这几个包。
2. github上建立项目“lxWei.github.com”，注意将“lx.Wei”换成自己的用户名。
3. git clone项目到本地。
4. 为简单起见，下载别人的jekyll模板框架，我用的是[Hawstein](https://github.com/Hawstein/hawstein.github.com)的，另外，[https://github.com/mojombo/jekyll/wiki/sites](https://github.com/mojombo/jekyll/wiki/sites)上有很多优秀的模板框架。
5. 用下载的模板框架覆盖掉自己的项目，切记，一定要删除下载的.git文件夹。
6. 改动框架中的内容，主要是修改个人信息和文章，如果个人懂前端技术，可以自己修改页面。
7. 在_post文件夹下添加yy-mm--dd-title.md格式命名的文件。写完之后push到github上即可。
8. Congratulations!

# 迁移wordpress的blog
参考[exitwp](http://yishanhe.net/exitwp-convert-wordpress--markdown/)。
1.	sudo apt-get install python-yaml python-beautifulsoup python-html2text。
2.	git clone git://github.com/thomasf/exitwp.git。
3.	将wordpress导出的xml文件放到exitwp下的wordpress-xml文件夹下。
4.	python exitwp。此时，可能报错，需要安装BeautifulSoup。
5.	生成的markdown文件在build文件夹下，拷贝到博客的_post文件夹下，提交即可。


# 参考资料
* [写作环境搭建(git+github+markdown+jekyll)][1]
* [Git与Github入门资料](http://www.yangzhiping.com/tech/git.html)
* [Markdown: Basics （快速入门)](http://wowubuntu.com/markdown/basic.html)
* [Hawstein的blog](https://github.com/Hawstein/hawstein.github.com)

[1]: http://ellochen.github.io/2013/03/写作环境搭建(git+github+markdown+jekyll)/


