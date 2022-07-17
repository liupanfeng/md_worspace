### Linux 常用命令总结

#### 1.查看Linux系统类别

```shell
lsb_release -a
```



#### 2.安装命令

```shel
ubuntu: apt-get install 
```

```shell
contOS： yum install 
```

```shell
MacOS： brew
```



#### 3.光标移动快捷键

##### 3.1光标移动到最前

```shell
Ctrl + A
```



##### 3.2.光标移动到最后

```shell
Ctrl + E
```



##### 3.3.清空当前输入的内容

```shell
 Ctrl + U 
```



#### 4.Linux系统路径介绍

##### 4.1bin 目录一些执行文件

##### 4.2home 目录用户

##### 4.3lib 目录常用的 so

##### 4.4.opt 和 proc 是与进程相关的



#### 5.ls相关

##### 5.1. ls  当前文件夹下面的所有文件/文件夹

##### 5.2. ls -all 当前文件夹下面的所有文件/文件夹等 的详细显示

##### 5.3. ls -lh 当前文件夹下面的所有文件/文件夹等 的大小详细显示

##### 5.4 ls -R  递归当前文件夹 到 文件，有点像 树形结构输出的效果



#### 6.查看当前时间

```shell
./date
```



#### 7.文件创建

```shell
touch file.sh
```



#### 8.文件介绍

```shell
-rw-r--r-- 1      ms     ms  0  Mar 25 11:15   file.sh
文件权限 硬链接计数 所有者 所属组 大小   时间         名称

-rw-rw-r--
文件类型，rw- 所有者可读可写可执行，rw- 同一组用户可读可写可执行 r-- 其他人可读可写可执行

```



#### 9.Linux 文件类型

##### 9.1. - 普通文件

##### 9.2. d 文件夹 

##### 9.3  l 软链接，硬链接软件接文件

##### 9.4 c 字符设备文件

##### 9.5 b 块设备文件

##### 9.6 p 管道文件

##### 9.7 s套接字文件



#### 10 删除相关

##### 10.1. rmdir 文件夹  只能清空空目录文件夹，如果文件夹里面有内容，就不会删除

##### 10.2. rm -r 文件夹 递归清空目录文件夹



#### 11.环境变量

##### 11.1. /ect/profile 全局环境变量（所有用户）

##### 11.2. ~/.bashrc 全局变量 （当前用户）

##### 11.3. export Android-SDK=/tools/sdk  临时变量

##### 11.4. echo $Android-SDK 可以查看



#### 12.文件读取操作

##### 12.1. cat file.sh     快速查看文件内容 

##### 12.2. tac file.sh    倒着快速查看文件内容

##### 12.3. more file.sh   每次只查看一页，回车查看下一页

##### 12.4.head -2 file.sh   查看前面2行内容 

##### 12.5. tail -3 file01.txt   查看后面2行内容



#### 13.查看当前的用户

```shell
whoami
```



#### 14.修改权限

权限分当前用户权限、同组用户权限、其他用户权限  4：可读   2：可写 1：可执行

```shell
chmod 777   全部用户可读可写可执行 
chmod 111   全部可执行
chmod 511   当前用户可读可执行 其他用户可执行

chmod +x    全部组增加可执行权限
chmod +w    全部组增加可写权限
chmod +r    全部组增加可读权限

//u(当前用户) ，g(同组)，o(other)，a(all)
chmod u+rwx  只给当前用户增加权限
```



#### 15.创建修改用户和用户组

```shell
sudo adduser lpf           		创建新用户 lpf
sudo chown lpf file.sh     		修改文件所属用户
sudo chgrp lpf file.sh     		修改文件所属组
sudo chown lpf:lpf file.sh 		修改所属用户/所属组
```



#### 16.Vim操作

* wq! !代表强制退出 

*  shift + zz 快捷退出vim编辑器 

*  :3 直接进入3行 

* /sa 查询sa字符、

* gg 最上面

* G 最下面 

* k 上一行



#### 17下载文件与解压

```shell
wget https://ffmpeg.org/releases/ffmpeg-4.0.2.tar.bz2
解压操作：
tar xvf ffmpeg-4.0.2.tar.bz2
.zip解压操作：
unzip xxx.zip
```



#### 18.文件夹移动与文件夹重命名

```shell
复制文件夹 到 另外一个目录
cp -rf /root/android-ndk-r17c/ /ms/home/tools/
重命名
mv ndk24 ndk25
```











