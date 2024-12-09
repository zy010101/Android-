# 常见Android反逆向技术
Android 反向工程技术分为以下几类常见的技术。
## 虚拟环境检测
虚拟环境检测是指识别应用是否运行在模拟器、虚拟机或受控环境中，这种环境通常用于调试、逆向分析或欺骗应用逻辑（例如绕过地域限制、支付验证等）。模拟器通常由开发者或攻击者使用，如 Android Studio 自带的模拟器、Genymotion、BlueStacks，以及某些定制的虚拟设备。一般会综合使用下面的方式来检测：
1. 系统属性检测

    虚拟环境的系统属性通常带有特定标志。
    ```java
    private boolean isEmulator() {
        return (Build.FINGERPRINT.startsWith("generic") ||  // 通用系统指纹
                Build.FINGERPRINT.startsWith("unknown") ||  // 未知指纹
                Build.MODEL.contains("google_sdk") ||       // Google SDK 模拟器
                Build.MODEL.contains("Emulator") ||         // "Emulator" 标志
                Build.MODEL.contains("Android SDK built for x86") || // x86 设备
                Build.BRAND.startsWith("generic") ||        // 通用品牌
                Build.DEVICE.startsWith("generic") ||       // 通用设备
                "google_sdk".equals(Build.PRODUCT));        // Google 产品标志
    }
    ```
2. CPU架构检测

    虚拟机通常使用 x86 架构，而大多数实际设备使用 ARM 架构。
    ```java
    private boolean isCpuEmulator() {
    String abi = Build.CPU_ABI.toLowerCase();
        return abi.contains("x86") || abi.contains("intel");
    }
    ```
3. 硬件特性检测

    虚拟设备的硬件特性和真实设备不同，例如硬件名称、传感器缺失等。
    ```java
    private boolean isHardwareEmulator() {
    String hardware = Build.HARDWARE.toLowerCase();
    return hardware.contains("goldfish") ||   // Android SDK 模拟器
           hardware.contains("ranchu") ||     // Android Emulator
           hardware.contains("qemu");         // QEMU 虚拟机
    }
    ```

4. 检测虚拟设备文件
    
    某些虚拟设备会包含特定文件或目录，例如：

    ```java
    private boolean hasEmulatorFiles() {
        String[] paths = {
            "/dev/qemu_pipe",        // QEMU 模拟器
            "/dev/socket/genyd",     // Genymotion
            "/dev/socket/baseband"   // 基带调试工具
        };
        for (String path : paths) {
            if (new File(path).exists()) {
                return true;
            }
        }
        return false;
    }
    ```
5. 检测传感器数量
    
    真实设备通常拥有多种传感器，而模拟器可能缺少部分传感器。
    ```java
    private boolean hasFewSensors(Context context) {
    SensorManager sensorManager = (SensorManager) context.getSystemService(Context.SENSOR_SERVICE);
    int sensorCount = sensorManager.getSensorList(Sensor.TYPE_ALL).size();
        return sensorCount < 5;  // 模拟器通常传感器较少
    }
    ```

6. 基站信息检测

    只提供手机 APP 的可以检测这一项，如果提供平板APP，则不能检测这一项。
    ```java
    private boolean isNetworkEmulator(Context context) {
    TelephonyManager tm = (TelephonyManager) context.getSystemService(Context.TELEPHONY_SERVICE);
    String networkOperator = tm.getNetworkOperatorName().toLowerCase();
        return networkOperator.equals("android") || networkOperator.contains("emulator");
    }
    ```
7. 显示分辨率和 DPI 检测
    
    模拟器通常使用标准化分辨率和 DPI，这可能与真实设备的参数不符。

    ```java
    private boolean isScreenEmulator(Context context) {
        DisplayMetrics metrics = context.getResources().getDisplayMetrics();
        return metrics.densityDpi == DisplayMetrics.DENSITY_MEDIUM ||  // 模拟器分辨率通常为 MDPI
            metrics.densityDpi == DisplayMetrics.DENSITY_HIGH;
    }
    ```

检测是否是真机环境的手动还有其他一些，过模拟器检测一般比较费劲。咱们使用真机就可以解决的问题，就不要在搞怎么过模拟器了。后续如果真的必须在模拟器上搞，那就回头在研究。
## 防止抓包
### 代理检测
代理检测是识别应用是否通过代理服务器（如 VPN、HTTP 代理、SOCKS 代理等）访问网络的一种保护机制。这对于防止用户伪造网络环境（例如进行欺诈或攻击）非常重要。以下是一些常用的代理检测方法：

1. 检测系统代理设置

    通过检查系统是否配置了 HTTP 或 HTTPS 代理来识别代理使用情况。

    ```java
    private boolean isProxySet() {
        String proxyAddress = System.getProperty("http.proxyHost");
        String proxyPort = System.getProperty("http.proxyPort");
        return proxyAddress != null && proxyPort != null;
    }

    // Android 9 (API 28) 及以上直接检查 ProxyInfo。
    private boolean isProxyConfigured(Context context) {
        ConnectivityManager cm = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
        if (cm != null && cm.getDefaultProxy() != null) {
            return true;  // 检测到代理
        }
        return false;
    }
    ```
2. 检测本地代理端口

    代理软件（如 Shadowsocks 或 VPN）通常会监听本地端口，可以扫描常见端口。


    ```java
    private boolean isLocalProxyRunning() {
        int[] proxyPorts = {8080, 8888, 3128, 1080};  // 常见代理端口
        for (int port : proxyPorts) {
            try {
                Socket socket = new Socket("127.0.0.1", port);
                socket.close();
                return true;  // 检测到代理
            } catch (Exception ignored) {
            }
        }
        return false;
    }
    ```
3. 检测 VPN
    
    VPN 通常会更改路由表，可以通过以下方式检查是否存在 VPN。

    ```java
    private boolean isVPNActive(Context context) {
        ConnectivityManager cm = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
        if (cm != null) {
            Network[] networks = cm.getAllNetworks();
            for (Network network : networks) {
                NetworkCapabilities caps = cm.getNetworkCapabilities(network);
                if (caps != null && caps.hasTransport(NetworkCapabilities.TRANSPORT_VPN)) {
                    return true;  // 检测到 VPN
                }
            }
        }
        return false;
    }
    ```
4. 检测系统是否安装了自签名证书

    中间人代理通常会通过在设备上安装自签名证书来解密 HTTPS 流量。你可以通过以下方式检测设备上是否存在可疑的用户证书。
    ```java
    private boolean isCustomCertificateInstalled(Context context) {
        try {
            KeyStore keyStore = KeyStore.getInstance("AndroidCAStore");
            keyStore.load(null);

            Enumeration<String> aliases = keyStore.aliases();
            while (aliases.hasMoreElements()) {
                String alias = aliases.nextElement();
                if (alias.contains("Fiddler") || alias.contains("Charles") || alias.contains("Burp")) {
                    return true;  // 检测到自签名证书
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }
    ```

5. 使用第三方服务检测

    一些第三方服务可以提供证书校验的支持，帮助快速检测代理的存在。例如：

    SSL Labs API：检测 HTTPS 连接的证书有效性。

    Certificate Transparency Logs：检查证书是否来自未授权的颁发机构。

    ```java
    // 示例：请求 CT 日志服务验证证书是否合法
    private boolean checkCertificateWithCTLogs(String certFingerprint) {
        String apiUrl = "https://ct-checker.example.com/api?fingerprint=" + certFingerprint;
        try {
            URL url = new URL(apiUrl);
            HttpURLConnection connection = (HttpURLConnection) url.openConnection();
            connection.connect();

            BufferedReader reader = new BufferedReader(new InputStreamReader(connection.getInputStream()));
            String response = reader.readLine();
            return response.contains("valid");  // 如果返回 "valid"，说明证书合法
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }
    ```
### 证书校验
1. 单向认证（服务器证书校验）

    在单向 TLS/SSL 认证中，客户端验证服务器的证书，确保通信连接的服务器是受信任的，并且通信不会被篡改。

    **防止抓包的策略：**

    验证证书链：服务器证书应由受信任的 CA 签发。如果证书链无效，或者证书不在客户端的信任存储中，连接会被拒绝，防止中间人攻击。

    域名匹配：客户端会检查服务器证书中的 CN（Common Name）或 SAN（Subject Alternative Name）是否与访问的域名匹配。如果不匹配，连接将会失败，从而防止恶意证书的使用。

    证书有效期：客户端会检查证书的有效期，确保服务器的证书没有过期。

2. 双向认证（客户端证书校验）

    在双向 TLS/SSL 认证中，除了验证服务器证书，服务器还会要求客户端提供证书，并验证客户端的身份。

    **防止抓包的策略：**

    服务器证书验证：服务器验证客户端证书是否由受信任的 CA 签发，确保客户端的身份是合法的。

    客户端证书验证：客户端提供的证书必须是合法的，并且必须由服务器信任的 CA 签发。此操作可以防止恶意用户通过中间人攻击伪造合法客户端。

3. 证书钉扎（Certificate Pinning）

    证书钉扎是一种增强的证书校验机制，客户端硬编码服务器的证书或公钥指纹，在建立 TLS 连接时，客户端与服务器证书的指纹进行匹配，确保服务器的证书没有被篡改或被中间人替换。

    防止抓包的策略：
    硬编码证书指纹：客户端在应用中存储并校验服务器证书的指纹（SHA-256），确保与服务器证书一致，即使攻击者控制了网络流量并更换了服务器证书，也无法通过钓鱼证书绕过校验。

    防止代理：通过强制证书钉扎，抓包工具（如 Burp Suite）无法替换证书并进行中间人攻击。
## 逆向检测
1. 使用 Debug.isDebuggerConnected 检测调试器
    ```java
    if (Debug.isDebuggerConnected() || Debug.waitingForDebugger()) {
        // 检测到调试器，执行反制操作，例如退出程序
        System.exit(1);
    }
    ```


2. 使用 ptrace 防调试，通过 ptrace 系统调用检测调试行为。

    ```c
    #include <sys/ptrace.h>

    void anti_debug() {
        if (ptrace(PTRACE_TRACEME, 0, 0, 0) == -1) {
            // 如果调用失败，可能说明程序正在被调试
            exit(1);
        }
    }
    ```

3. 检测 Hook 工具，通过检测常见 Hook 工具（如 Frida、Xposed）来阻止逆向分析
    ```java
    private boolean detectHookFramework() {
        String[] knownHookFiles = {
            "/data/app/de.robv.android.xposed.installer-1/base.apk",
            "/data/app/de.robv.android.xposed.installer-2/base.apk"
        };

        for (String file : knownHookFiles) {
            if (new File(file).exists()) {
                return true;
            }
        }

        return false;
    }
    ```
4. 使用动态加载技术，将敏感逻辑放置在加密的 SO 库或外部文件中，通过 JNI 或反射动态加载.
    ```java
    System.loadLibrary("native-lib");  // 加载加密的 SO 文件
    ```
