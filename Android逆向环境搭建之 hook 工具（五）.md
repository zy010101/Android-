# Android逆向环境搭建之 hook 工具
首先，我们来介绍一下到底什么是 hook？
## Hook 的工作原理
拦截函数调用：你可以指定应用程序中的某个函数或方法进行拦截，该应用程序在执行该函数时会被我们插入的 hook 代码所替换，然后执行我们的自定义代码。在 hook 后，你可以修改被拦截函数的输入、输出，或者干脆阻止其继续执行。通过 hook，可以动态地记录函数的调用过程，捕捉到程序内部的状态，帮助开发人员分析程序的行为，或者验证反编译结果。
## Frida 介绍和安装
Frida 是一个面向开发人员、逆向工程师和安全研究人员的动态检测工具包。它允许您将 `JavaScript` 代码片段或您自己的库注入到 Windows、macOS、GNU/Linux、iOS、watchOS、tvOS、Android、FreeBSD 和 QNX 上的 `原生应用` 中。我们直接开始安装 Frida，通过实际的操作来进一步理解 hook 是什么。
## 安装 Frida
Firda通常是通过 python 的 pip 直接进行安装，为了避免环境混乱，建议使用 python 的虚拟环境（可以通过 conda 或者 venv 等方式来创建虚拟环境）
  
    pip install frida-tools
安装完成之后，我们使用 pip 来查找 frida 的版本

    pip freeze
    
![image](https://github.com/user-attachments/assets/381f3f78-562d-4ab8-a965-62a3d399fdc7)

得到安装的 frida 版本是 16.5.7，之后我们要去手机端安装 Frida Server，Frida 开源在 Github 上，我们可以在其下载页面上下载和电脑上安装的同版本的 Frida Server，但是 Frida Server 支持的架构非常多，那么如何确认我们使用哪一个？可以通过 adb 命令来查看当前手机的架构信息。

    adb shell getprop ro.product.cpu.abi
    
其实也没什么看的，现在几乎都是 arm64，K40当然也不例外。那我们就下载 Android 平台的 arm64 的 Frida Server 即可。下载之后，推送到手机上，流程如下：

    adb push frida-server /data/local/tmp/      # 将frida-server 推送到该目录下
    adb shell                                   # 启动 adb shell
    su                                          # 切换到 root 权限
    cd /data/local/tmp/                         # 进入到存放 Frida Server的目录下
    chmod +x frida-server                       # 赋予可执行权限
    ./frida-server                              # 运行 frida-server

到这里，我们手机端的 frida server 就运行起来了。然后我们需要转发两个端口到PC上。

**27042 端口**

这是 Frida 的 控制端口。Frida 客户端（例如使用 frida 命令行工具或 Frida Python API）通过此端口连接到目标设备（可能是物理设备或模拟器），并与 Frida Server 进行通信。通过该端口，Frida 客户端发送命令并接收来自 Frida Server 的响应。
通常，Frida 客户端通过此端口执行如加载脚本、挂钩函数、注入 JavaScript 代码等操作。

**27043 端口**

这是 Frida 的 数据端口。它用于传输从目标设备捕获到的数据或执行结果，如函数调用日志、内存信息、变量值等。Frida Server 会在目标应用程序执行过程中将数据流发送到该端口，Frida 客户端则可以接收并分析这些信息。

为了方便起见，我们直接将这两个端口转发的操作写成一个 python 程序，这样之后每次执行该 python 即可。
```python
import subprocess

# 如果连接多台设备，则需要通过-s指定设备，使电脑上的端口和设备上的端口建立连接，前面是电脑上的端口号
    # 比如：adb -s 127.0.0.1:62001 forward tcp:27042 tcp:27042
    # 比如：adb -s 127.0.0.1:62025 forward tcp:27042 tcp:27042

subprocess.getoutput('adb forward tcp:27042 tcp:27042')
subprocess.getoutput('adb forward tcp:27043 tcp:27043')
```

ok, 对手机端的操作先到这里，接下来我们该考虑如何 hook 了。
## hook 代码编写
Frida 的核心是用 C 编写的，并将QuickJS 注入目标进程。所以核心的 hook 部分的代码需要使用 js 实现。只需要会 JavaScript 的基本语法就可以完成 hook 任务。所以 js 也得会一些。

我们先拷贝一份 hook 代码的模板，然后进行修改即可。修改完成之后的代码如下所示：
```python
import frida
import sys

### 以后这个代码不需要动### ### ###
rdev = frida.get_remote_device()
session = rdev.attach("小猿口算")  # 写上要hook的app的名字，attach方案
### 以后这个代码不需要动### ###


# 要改动的地方,js实现 hook
scr = """
Java.perform(function () {
    // 找到类 反编译的包名+类名
    var encode = Java.use("wg.i");
    
    // 替换类中的方法，方法有几个参数，就要传几个参数
    // 这里的a就是我们需要修改的方法，而implementation是固定写法
    encode.a.implementation = function(str){
        console.log("参数：",str);      // 传入的参数打印了，我们猜是明文数据
        var res = this.a(str);         //调用原来的函数
        console.log("返回值：",res);    // 打印出正常执行这个方法，返回的结果
        return res;
    }
});
"""


####下面代码完全不需要动
script = session.create_script(scr)
def on_message(message, data):
    print(message, data)
script.on("message", on_message)
script.load()
sys.stdin.read()
```
放上之前的反编译代码，对比看一下就明白了。我们 hook 的是 wg 这个包下面的 i 这个类中的方法 a, 这个方法只有一个参数，所以我们之前的 js 代码中 hook 的时候，函数的参数也是 1 个。
```java
package wg;

import java.io.UnsupportedEncodingException;
import java.security.InvalidKeyException;
import java.security.KeyFactory;
import java.security.NoSuchAlgorithmException;
import java.security.PublicKey;
import java.security.Security;
import java.security.spec.InvalidKeySpecException;
import java.security.spec.X509EncodedKeySpec;
import javax.crypto.BadPaddingException;
import javax.crypto.Cipher;
import javax.crypto.IllegalBlockSizeException;
import javax.crypto.NoSuchPaddingException;

/* loaded from: classes3.dex */
public class i {

    /* renamed from: a, reason: collision with root package name */
    public static String f69169a = "MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDSovT1rrwzrGoMCFb6z8e+5lzVdAD5o8krGIwdfxrVE2OnMijUZdkQk7etPJvZ2JOVXghthAGUUJkDUE8n2ZMNFKPjMrQJI49ewVzqWOKOvgU6Iu60Sn0xpeietP1wWXBkszdV1WfNBJUo2hhPDnIPMGzzdfLW5rMu+tczeUriJQIDAQAB";

    /* renamed from: b, reason: collision with root package name */
    public static PublicKey f69170b;

    static {
        KeyFactory keyFactory;
        try {
            keyFactory = KeyFactory.getInstance(com.alipay.sdk.encrypt.d.f17015a);
        } catch (NoSuchAlgorithmException unused) {
            keyFactory = null;
        }
        try {
            f69170b = keyFactory.generatePublic(new X509EncodedKeySpec(a.a(f69169a, 2)));
        } catch (InvalidKeySpecException unused2) {
        }
    }

    public static String a(String str) throws NoSuchAlgorithmException, NoSuchPaddingException, InvalidKeyException, UnsupportedEncodingException, IllegalBlockSizeException, BadPaddingException {
        Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1PADDING", Security.getProvider("BC"));
        cipher.init(1, f69170b);
        return a.f(cipher.doFinal(str.getBytes()), 2);
    }
}
```
