# Android逆向环境搭建
## 真机准备
推荐真机，真机需要能够解锁BootLoader，并且能够root。推荐 Google 的 Pixel 系列手机，可以在淘宝，咸鱼等平台购买二手手机，注意一定要解锁BootLoader，或者让卖手机的人给你root，并安装 Magisk。如果是使用国产的手机，推荐使用小米的手机，目前小米的手机可以解锁BootLoader，但是条件比较苛刻，需要答题，需要等待 7 天才能解锁BootLoader。
后续教程中，笔者会使用一台 Redmi K40 手机作为示例。
## adb 安装
1. 下载 adb，下载地址：[https://dl.google.com/android/repository/platform-tools_r28.0.0-windows.zip](https://developer.android.google.cn/tools/releases/platform-tools?hl=zh-cn)，一般情况下下载最新版本即可。
2. 解压到任意位置，例如 D:\Android\platform-tools
3. 将 D:\Android\platform-tools 添加到环境变量 PATH 中
4. 测试 adb 是否安装成功，在 cmd 中输入 adb version
5. 如果出现如下结果，则表示安装成功

    ```
    Android Debug Bridge version 1.0.41
    Version 35.0.2-12147458
    Installed as E:\learn\tools\platform-tools-latest-windows\platform-tools\adb.exe
    Running on Windows 10.0.22631
    ```
## adb 操作手机
1. 启动手机，并打开开发者选项，然后开启 USB 调试。以 Redmi K40 （更新到了澎拜OS）手机为例。

    **设置 -> 我的设备 -> 全部参数与信息 -> MIUI版本**
    
    连续多次点击 MIUI版本，直到进入开发者模式为止。然后在搜索关键字“开发者”，进入开发者选项。在开发者选项中，找到 USB 调试并开启。然后使用能够传输数据的数据线连接手机和电脑，此时如果是第一次连接，那么会弹出提示框，选择“始终允许”。这样ADB就可以正常使用了。

2. 检测设备是否连接，在 cmd 中输入 adb devices，如果显示了手机的序列号（在手机设置中搜索序列号，即可看到），则表示连接成功。
3. adb 常用命令 

    1. adb devices ：查看已连接的设备
    2. adb shell ：进入手机的 shell 模式，可以执行一些命令，例如输入 ls 查看当前目录下的文件，输入 ps 查看当前运行的进程，输入 cat /proc/cpuinfo 查看 CPU 信息，输入 cat /proc/meminfo 查看内存信息，输入 cat /proc/version 查看内核版本，输入 cat /proc/stat 查看 CPU 使用情况，输入 cat /proc/net/route 查看网络信息，输入 cat /proc/net/arp 查看 ARP表，输入 cat /proc/net/tcp 查看 TCP 连接
    3. adb pull ：从手机中拉取文件到电脑中，例如 adb pull /sdcard/test.txt D:\test.txt
    4. adb push ：将电脑中的文件推送到手机中，例如 adb push D:\test.txt /sdcard/test.txt
    5. adb install ：安装应用，例如 adb install D:\test.apk
    6. adb uninstall ：卸载应用，例如 adb uninstall com.example.test
## 刷机
**刷机前，先备份好手机的数据，包括应用，文件，数据库等**，不建议在日常使用的手机上进行刷机，因为刷机后，手机的数据会丢失，包括应用，文件，数据库等。

**刷机不是我们的目的，我们的目的是 root 手机**。之所有要刷机，是因为我们我们这里介绍的方法是通过 Magisk 获取 root 权限，而 Magisk 是通过修补 Android 系统的 Binary （boot.img，init_boot.img 或 recovery.img 文件）来实现的，如果我们拥有当前手机的 boot.img，init_boot.img 或 recovery.img 文件，那么我们就可以直接使用 Magisk 获取 root 权限，不需要刷机。

因此刷机的目的实际上是为了让 Magisk 能够获取到 boot.img，init_boot.img 或 recovery.img 文件。通过修补这些文件，进行 root 操作。

前文已经提过，小米手机需要解锁 BootLoader，才能刷机。可以在小米官方下载解锁BL的工具：[解锁工具](https://www.miui.com/unlock/index.html)。使用解锁工具进行BL解锁之后，就可以进行刷机了。

刷机的资源可以在 [miui历史网站](https://miuiver.com/) 中找到，也可以在小米社区中搜索刷机教程。这里给一个刷机包的帖子地址：https://web.vip.miui.com/page/info/mio/mio/detail?postId=37093637&app_version=dev.20051

大家可以在其中找到小米/红米的刷机ROM包，按照自己手机的型号，找到想刷的 ROM 包，下载 ROM 包。耐心等待即可。下载完成之后解压ROM 包。

然后就要开始进行刷机了，线刷手机的时候，使用 MiFlash 刷机即可，https://miuiver.com/miflash/ 中汇总了非常多版本的 MiFlash，大家可以自行选择。

之后，可以按照 https://miuiver.com/how-to-flash-xiaomi-phone/ 中的教程进行刷机。**需要注意的是，在刷机过程中，手机没有自动重启或者 MiFlash 软件没有显示成功或者失败的情况下，不要随意操作该软件，否则刷机可能会失败，需要重新刷入，又要经历等待阶段，因此一定要耐心等待，不要乱动。**

## root 手机
1. 下载 Magisk，下载地址：[https://github.com/topjohnwu/Magisk/releases](https://github.com/topjohnwu/Magisk/releases)，下载最新版本的 Magisk。

2. 然后使用 adb install 命令安装 Magisk，例如 adb install app-release.apk

3. 将之前解压的ROM包中的 boot.img，init_boot.img 或 recovery.img 文件推送到手机中。使用adb push 命令，例如 adb push boot.img /sdcard/Download/boot.img，这样在 Download 文件夹中就会有一个 boot.img 文件。

4. 参考 https://miuiver.com/install-magisk-for-xiaomi/ 中的教程，对文件进行修补，修补文件默认保存在手机内部存储 Download 目录。之后就完成了 root 手机的操作。

5. 重启手机之后，打开 Magisk, 此时超级用户这栏已经可以点击进去，就说明 root 成功, 之后，可以在电脑上使用 abd shell 命令，进入手机的 shell 模式，然后执行 su 命令，看到手机上弹出 root 权限申请，就表示成功了。
