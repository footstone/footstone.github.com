---
layout: post
title: 在heroku上部署picasa-gallery
---

###在heroku上部署picasa-gallery

[picasa-gallery](https://github.com/angryziber/picasa-gallery)是Github上一个Java应用，作者重新设计了一个相册前端，并通过picasa api查询用户的picasa照片数据作展现。老严改了一些bug后，将它部署在GAE上，用来应对picasa被墙的问题。为了提升点逼格，我也跟风搞了一把，并且考虑到picasa图库间歇性抽风被墙的问题，索性将相册图片同步到DreamHost上，并且修改了picasa-gallery的代码，将页面图片直接修改链接指向到DreamHost的图库上。原本设想的是直接同步到GAE上，后来发现GAE完全不支持文件存储，甚至重新封装了Java io相关API，所以只能转向DH了。

在GAE上部署Java应用其实还是挺麻烦的，要下载GAE sdk，要安装Eclipse GAE插件，还要通过注册企业套件来配置自定义域名（现在已不支持）。所以，胡胖在部署时就很痛苦，其间数次吐槽，Java语言无辜躺枪数次。另外，GAE为了保证服务器的安全和性能，改造了一些接口，对应用功能也有很大限制。于是想到在heroku上试试。

在英文不通、git生疏、maven不熟的状况下，我原以为使用Heroku部署Java应用也会很麻烦，今天了解到可以通过上传war包的方式部署，瞬间觉得世界美好了很多。

picasa-gallery中原build.xml依赖GAE sdk，所以我新写了一个更简洁的simple-build.xml，毕竟我只是想打个war包而已啊！

####准备工作
1.	注册github、heroku
2.	本机安装git、配置ant、[HerokuToolbelt](https://toolbelt.heroku.com/)、[heroku-deploy插件](https://devcenter.heroku.com/articles/war-deployment)
3.	在heroku上创建一个用来部署相册应用的app


####部署步骤

1、源码下载

```
git clone git@github.com:footstone/picasa-gallery.git
```

2、修改src/config.properties，不需要镜像地址的忽略image.url配置,则依然链接到picasa图库上

```
google.user: google账号名
image.url: 镜像图库域名地址(如http://x.com/photos),多个地址之间用";"间隔（不需要镜像的忽略）
local.path: 镜像服务器上相册地址(如/home/app/photos)在同步比较相册时使用（不需要镜像的忽略）
```

3、在picasa-gallery目录下执行：

```
ant -f simple-build.xml
```
执行完后，会在picasa-gallery目录下创建dist文件夹，其中会有picasa-gallery.war和picasa-gallery.jar。

4、将picasa-gallery.jar和./lib/*.jar上传到镜像服务器，并在创建定时同步任务，执行：(不需要镜像的忽略)

```
java -classpath ${CLASSPATH} com.footstone.photos.proc.Synchronize sync
```

5、执行heroku deploy：

```
heroku deploy:war --war <path_to_war_file> --app <app_name>
```
部署成功后，即可通过http://app_name.herokuapp.com访问相册应用了。

6、在heroku app settings页面上，为应用增加custom domains。

7、修改自己的域名指向appname.herokuapp.com。
