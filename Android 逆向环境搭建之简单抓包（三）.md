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

![image](https://github.com/user-attachments/assets/8e0f5ce5-a378-4934-9351-ec01604e7406)


按照下图进行配置即可，SOCKS 代理在本文不做介绍，后续在介绍。

![image](https://github.com/user-attachments/assets/e413e6d5-5f67-4855-8703-bb93e2e45ef8)

OK，到这里，我们的电脑端配置就完成了。
## Android 证书安装
在 Android 7.0 之后，用户安装的证书默认在用户的模块下面，而不是安装到系统下面。所以我们还需要将证书移动到系统证书中去。本文将借助 Magisk 的模块去实现这个功能。
1. 导出 charles 的证书

   ![image](https://github.com/user-attachments/assets/93bd8b6a-c610-462c-976c-03846321176c)

    导出证书的时候，选择导出为 cer 证书即可。
2. 将证书安装到手机上

    使用 adb 将证书推送到手机上。例如：

        adb push charles.cer /storage/emulated/0/Download/charles.cer

    这样在 Download 文件夹下就会有一个 charles.cer 文件。此时是无法直接进行安装的，一般都是需要在手机的设置搜索证书，然后在弹出来的证书窗口中进行选择安装，在 K40 中如下所示：
    
    ![image](https://github.com/user-attachments/assets/626fe661-8f54-417b-922d-ba7c4145900e)

选择CA证书，进入到证书安装界面

![image](https://github.com/user-attachments/assets/b3fa40e3-47d1-452e-ad2c-dddf18d44f75)

然后安装证书

![image](https://github.com/user-attachments/assets/82604b1e-575d-4476-bbb0-f6d55873cd5d)

然后选择 Download 目录下刚才导入的证书，进行安装。安装之后在信任的凭据中可以进行查看

![image](https://github.com/user-attachments/assets/b42bae37-d209-4167-8bc7-f53d7882d1f5)

可以看到，我们为用户安装了一个证书，这个证书不是系统的。

![image](https://github.com/user-attachments/assets/00bfe3c9-0512-4261-abd4-fc4620c9f8c5)

3. Magisk 模块
    使用 GitHub 上的 https://github.com/ys1231/MoveCertificate 提供的 Magisk 模块来安装一个可以将证书安装到系统的模块。目前是支持 Android 7-15的。

    在 [releases](https://github.com/ys1231/MoveCertificate/releases) 中下载 Magisk 模块，然后通过 adb 将模块推送到手机上。

    之后，打开手机上的　Magisk 软件，在模块中选择从本地安装，选择我们刚刚推送上去的模块，安装完成之后，然后重启手机。在你的系统证书列表中，就会有一个 charles.cer 证书了。

    ![image](https://github.com/user-attachments/assets/7f9f651e-561b-4171-a320-da7d8ebfd373)

## 抓包
保证你的手机和电脑连接同一个 WIFI，配置手机的 WIFI 代理为你的电脑的IP地址，端口是8888。就是我们刚才在 charles 中配置的。之后，我们在 charles 中进行抓包，就可以抓到手机浏览器的包了。

![image](https://github.com/user-attachments/assets/3cfc990b-a7da-4640-8f57-525f6ce81201)

图中的1是打开SSL代理，也就是让我们可以抓到 HTTPS的包，2是开始监听流量报文。
我们在手机上的浏览器中搜索奶龙，就可以看到抓包工具中显示的奶龙搜索结果

![image](https://github.com/user-attachments/assets/8e4c5709-6334-45eb-a340-02e2ce1b4cf2)

## 配置 charles 外部代理
通过配置 Charles 的外部代理，我们可以在 burp suite 等其他软件上对 Charles 抓取的包进行操作。
![image](https://github.com/user-attachments/assets/b462d61f-cedc-49d3-b777-ae379d373bb8)

![image](https://github.com/user-attachments/assets/2b64cb58-2780-400c-ad52-bd2c56c948b9)

进行上述配置之后，在burp suite 中就可以可以很方便的操作我们抓到的包了。

**如果burp suite再次代理的时候，app的包出现问题，那么请检查 burp suite 的默认设置是不是对请求和响应做了修改，Charles 是不会对请求和响应做修改的。** 最重要的就是下面两处的配置，他们会对部分包造成影响
![image](https://github.com/user-attachments/assets/09c69155-293e-4dd1-aa30-aaf2adb3b99b)
![image](https://github.com/user-attachments/assets/8de61631-9bb6-4a16-8a7d-7b304c59f5c4)
