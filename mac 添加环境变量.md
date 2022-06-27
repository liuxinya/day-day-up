* 打开mac的终端  默认路径是  `~` 
* 这玩意可不是电脑的根目录
* 如果你输入 ` cd / ` 会发现 还有其他目录  或者用 `pwd` 命令来获取当前路径

####  拿添加 `MongoDB` 的例子

*   我解压好的mongodb文件夹 直接放到桌面的app 文件夹里了
![file-src.jpg](https://upload-images.jianshu.io/upload_images/9948410-169c90f980c5b11a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

+ Mac系统的环境变量，加载顺序为： 
  - a. /etc/profile 
  - b. /etc/paths 
  - c. ~/.bash_profile 
  - d. ~/.bash_login 
  - e. ~/.profile 
  - f. ~/.bashrc 

 其中a和b是系统级别的，系统启动就会加载，其余是用户接别的。c,d,e按照从前往后的顺序读取，如果c文件存在，则后面的几个文件就会被忽略不读了，以此类推。~/.bashrc没有上述规则，它是bash shell打开的时候载入的。这里建议在c中添加环境变量  (来源： https://blog.csdn.net/handsomefuhs/article/details/79687381)

##### 1. vim编辑器打开环境变量文件 
```
 vi .bash_profile
```

#####  2. 添加变量 

```
export PATH="<mongodb-install-directory>/bin:${PATH}"
```
* 将` <mongodb-install-directory>`  替换为 mongodb的安装路径
![bash_profile.jpg](https://upload-images.jianshu.io/upload_images/9948410-48ab84c25785d887.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#####  3. 让环境变量生效
```
source .bash_profile
```

##### 4. 重启终端

*  保存退出之后，在重启终端之前可以确认下刚才有没有成功的添加的变量
```
echo $PATH     // 可以打印出环境变量 如果成功了 列表会有显示
```
*  然后输入mongod mongo 都会有反应了


