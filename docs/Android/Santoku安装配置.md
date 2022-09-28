### 镜像下载

下载地址：[http://santoku-linux.com/download/](https://udomain.dl.sourceforge.net/project/santoku/santoku_0.5.iso)

### 安装

使用`VMware`或者`VBox`进行安装。

![image-20220928141255868](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928141255868.png)

![image-20220928141350293](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928141350293.png)

系统选择`Linux`，版本选择`Ubuntu 64`位

![image-20220928141429357](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928141429357.png)

内存给`3G`以上

![image-20220928141536026](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928141536026.png)

磁盘容量给`60G`以上，选择”将虚拟磁盘拆分成多个文件“

![image-20220928141705490](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928141705490.png)

在虚拟机设置中配置`ISO`镜像文件

![image-20220928142110308](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928142110308.png)

开启虚拟机，进行系统安装。选择默认的第一个即可

![image-20220928142229219](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928142229219.png)

进入桌面后，双击”Install Santoku14.04“启动安装

![image-20220928142358545](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928142358545.png)

语言、键盘布局选择默认的英语（`English`），如果选中文，后面安装过程会出错。来到”Preparing to install Santoku“界面，需要勾选下面两个选项

![image-20220928142647965](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928142647965.png)

使用默认选择的第一个，直接点击”Install Now“即可

![image-20220928142823229](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928142823229.png)

键盘布局使用默认的`English(US)`，设置用户名和密码，等待安装完成。

弹出更新提示窗口，选择更新。

### 安装VMTool

点击菜单”虚拟机“-”安装VMware Tool“，在弹出窗口中复制`tar.gz`的压缩包到桌面，然后解压压缩包

```sh
tar xzvf VMwareTools-10.3.23-16594550.tar.gz
cd vmware-tools-distrib/
```

使用`root`权限安装`VMwareTool`

```sh
sudo bash vmware-install.pl
```

或者先切换到`root`用户，再执行安装脚本

```sh
sudo -i
./vmware-install.pl
```

第一个选项输入`yes`，其他的使用默认值，直接一路回车

![image-20220928145119855](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928145119855.png)

安装完成后，重启系统即可。删除`VMwareTool`安装文件

```sh
rm -rf VMwareTools*
rm -rf vmware-tools*
```

### 优化配置

##### 更换源

点击桌面左下角菜单，选择”Preferences“--”Software & Updates“，

![image-20220928161322930](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928161322930.png)

在”Ubuntu Software“的”Download from“选项，选择”Other...“

![image-20220928161514110](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928161514110.png)

在弹出的”Choose a Download Server“窗口，找到”China“，点击展开，选择一个国内的镜像站

![image-20220928161810753](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928161810753.png)

我这里选的是中科大的镜像站。选择好之后点击”close“关掉即可

![image-20220928162004397](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928162004397.png)

检查并更新软件包

```sh
apt update -y && apt upgrade -y
```



##### 桌面右键菜单

桌面右键选择”Desktop Preferences“，点击”Advanced“，勾选第一个选项，如下图：

![image-20220928154348775](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928154348775.png)

关闭之后，再桌面右键就有包括终端在内的许多快捷选项了

![image-20220928154510654](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928154510654.png)

##### 升级`Java`

`Santoku`默认提供的`Java`版本是1.7，需要升级到`Java1.8`

```sh
sudo add-apt-repository ppa:openjdk-r/ppa
sudo apt-get update
sudo apt-get install openjdk-8-jdk
sudo update-alternatives --config java
```

![image-20220928164623449](https://raw.githubusercontent.com/robinoneway/ImageHost/master/img/image-20220928164623449.png)

##### 升级`ApkTool`

下载最新版的`ApkTool jar`包，重命名为`apktool.jar`，替换掉旧版本的`apktool.jar`文件

```sh
mv apktool2.6.1.jar /usr/share/apktool/apktool.jar
```

`ApkTool`包装器脚本：https://raw.githubusercontent.com/iBotPeaches/Apktool/master/scripts/linux/apktool