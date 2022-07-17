**Typora配置PicGo-Core（Command line）**

图片自动上传的工具

**1.配置图片仓库**

可以使用githug 也可以使用gee，基本类似

创建设置私人令牌，注意只显示一次，保存下来。

创建图片仓库，设置会公开的，不能是私密的。



2.配置npm环境因为需要安装插件

* nodejs下载网址：[Node.js](https://nodejs.org/en/) ，直接安装就行

* 配置缓存目录

* npm config set prefix "D:\nodejs\node_global"

* npm config set cache "D:\nodejs\node_cache"

* 配置一个镜像站，为了提升速度 ：npm config set registry=http://registry.npm.taobao.org

* npm config list  查看配置信息





3.设置Typora偏好设置以及安装插件

Typora文件--偏好设置---图像

选择 PicGo-Core（Command line）

点击安装，安装等待成功

点击打开配置文件

```shell
{
  "picBed": {
    "current": "gitee",
    "uploader": "gitee",
    "gitee": {
      "branch": "master",
      "customPath": "",
      "customUrl": "",
      "path": "img/",
      "repo": "geeId/创建的仓库名",
      "token": "设置个人token"
    }
  },
  "picgoPlugins": {
    "picgo-plugin-gitee-uploader": true,
    "picgo-plugin-super-prefix": true
  },
}
```



安装插件 cd C:\Users\%用户名%\AppData\Roaming\Typora\picgo\win64

```shell
 .\picgo.exe install gitee-uploader
 .\picgo install super-prefix
```

点击测试上传，就能成功了。





