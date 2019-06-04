#### 使用github

1. 先在github创建一个版本库，此时版本库有一个Git地址如： https://github.com/CNTcnt/CoolWeather.git；

2. 创建一个Android项目，在Git Bash 中把远程版本库克隆到本地；

   * 先在Git Bash中切换到本机的Android项目的储存位置；

   * 然后输入把远程版本库克隆到本地的项目中

     ~~~java
     git clone  https://github.com/CNTcnt/CoolWeather.git
     ~~~

   * 使用到Android项目的目录下ls -al命令可以查看

   * 然后，我们需要将我们的Android项目的目录中所有的文件全部复制到它的上一层目录中，这样就能将整个Android项目添加到版本控制中去了；在复制时，上一层目录中也有一个.gitignore文件，直接覆盖即可；

3. 接下来就可以把Android项目中的所有文件提交到GitHub上面了

   * 先将所有文件添加到版本控制中，如下：git  add
   * 然后在本地执行提交操作:git commit -m "记录更新信息随意"
   * 最后将提交的内容同步到远程版本库，也就是GitHub上(这一步GitHub要求输入用户名和密码验证)：git  push  origin  master