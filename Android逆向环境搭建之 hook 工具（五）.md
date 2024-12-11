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
