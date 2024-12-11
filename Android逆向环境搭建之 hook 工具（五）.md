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

ok, 对手机端的操作先到这里，接下来我们该考虑如何编写 hook 代码了。


