# Android 逆向环境搭建之简单抓包（三）
虽然 Android APP 有很多手段来防止我们抓包或者是逆向。但是正所谓道高一尺魔高一丈，有反调试的方式就有调试的方式。只是看对抗的成本而已。
## 抓包浏览器
目前大多数的 APP 应该都无法直接通过抓包来获取到数据。但是万一你遇到的那个 APP 可以呢。另外抓到包了之后，可以根据其中的一些信息去在反编译之后的代码中进行搜索。从而获取到加密的数据或者加密的方式。
### 安装和配置 charles
1. 下载 charles

    点击这里在[charles](https://www.charlesproxy.com/download/)的官网直接下载即可。

2. 安装 charles

    Charles 的安装是傻瓜式的，直接双击安装包，然后下一步，下一步，直到安装完成。

3. 配置 charles
安装完成以后，配置代理的域名和端口。
![image](https://github.com/user-attachments/assets/f1b998e2-726d-44c1-b4b8-1b2efcf4f697)

通常我们都是配置`*:*`，这样就会匹配所有域名和端口，如果你的抓包目标是明确的，那么可以配置对应域名下的所有端口。
![image](https://github.com/user-attachments/assets/0970a690-6d77-4d89-904a-ba57d5bc036b)

我这里就都是填上*了
![image](https://github.com/user-attachments/assets/c622158e-004a-44f2-8057-ba2ac35f57c4)

接下来配置代理监听的端口和协议
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/ebf51ba0583242cdbfb188fa8d6d697c.png)

按照下图进行配置即可，SOCKS 代理在本文不做介绍，后续在介绍。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/6c39921b93ab44009d153c73715ab131.png)
OK，到这里，我们的电脑端配置就完成了。
## Android 证书安装
在 Android 7.0 之后，用户安装的证书默认在用户的模块下面，而不是安装到系统下面。所以我们还需要将证书移动到系统证书中去。本文将借助 Magisk 的模块去实现这个功能。
1. 导出 charles 的证书
  按照下图的操作导出证书![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/d7d98b7b2bcd443dbf6d3fee4926a958.png)
导出证书的时候，选择导出为 cer 证书即可。
2. 将证书安装到手机上

    使用 adb 将证书推送到手机上。例如：

        adb push charles.cer /storage/emulated/0/Download/charles.cer

    这样在 Download 文件夹下就会有一个 charles.cer 文件。此时是无法直接进行安装的，一般都是需要在手机的设置搜索证书，然后在弹出来的证书窗口中进行选择安装，在 K40 中如下所示：
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/fc5a4414380548439d0b27d2d07a6a43.jpeg)
选择CA证书，进入到证书安装界面
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f2fd7c7d644a447e9b76e253e6497623.jpeg)
然后安装证书
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/78d7a40b25e84dcd867b3dfe869dd7cc.jpeg)
然后选择 Download 目录下刚才导入的证书，进行安装。安装之后在信任的凭据中可以进行查看
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/511091d7a5164c1691d67c1b8b808387.jpeg)
可以看到，我们为用户安装了一个证书，这个证书不是系统的。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a266d52f227148d99bdb94fcfe9635c7.jpeg)
3. Magisk 模块
    使用 GitHub 上的 https://github.com/ys1231/MoveCertificate 提供的 Magisk 模块来安装一个可以将证书安装到系统的模块。目前是支持 Android 7-15的。

    在 [releases](https://github.com/ys1231/MoveCertificate/releases) 中下载 Magisk 模块，然后通过 adb 将模块推送到手机上。

    之后，打开手机上的　Magisk 软件，在模块中选择从本地安装，选择我们刚刚推送上去的模块，安装完成之后，然后重启手机。在你的系统证书列表中，就会有一个 charles.cer 证书了。

![!\[在这里插入图片描述\](https://i-blog.csdnimg.cn/direct/39aec57a226c4333850c8d482ddfa643.jpe](https://i-blog.csdnimg.cn/direct/a3da537ba59844a9afff20093f60ad7b.jpeg)
## 抓包
保证你的手机和电脑连接同一个 WIFI，配置手机的 WIFI 代理为你的电脑的IP地址，端口是8888。就是我们刚才在 charles 中配置的。之后，我们在 charles 中进行抓包，就可以抓到手机浏览器的包了。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/0005ab53d26744aab09243b2d92d48d4.png)
图中的1是打开SSL代理，也就是让我们可以抓到 HTTPS的包，2是开始监听流量报文。
我们在手机上的浏览器中搜索奶龙，就可以看到抓包工具中显示的奶龙搜索结果

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7c48e1ce3265400b9ea4f39a257988c8.png)


