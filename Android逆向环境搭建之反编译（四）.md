![image](https://github.com/user-attachments/assets/361e1b8a-fbf1-47a9-a59b-a06a732e77d8)# Android逆向环境搭建之反编译
在简单抓包中，我们实际上可以抓到一些 APP 的包，但是这些包中携带的数据是加密的，不是明文的，我们无法通过修改抓到的包来进行测试。比如，我这里以小猿口算APP为例，在账号密码登录的时候，抓到的包如下图所示：

![image](https://github.com/user-attachments/assets/61318c5b-cdc1-4547-95f8-d36ef55e081b)

我们可以看到 `/leo-gateway/android/auth/password` 这个 URL 大概率就是使用账号密码登录的 API 的 URL，经常查看其他的请求，发现它们返回的响应都没有响应体，因此可以断定，该 API 就是用来账号密码登录的 API。我把完整的报文放在下面：

    POST /leo-gateway/android/auth/password?YFD_U=5712412029357391116&_productId=611&platform=android33&version=3.97.3&vendor=xiao_mi&deviceCategory=phone&av=5&isBackground=0&sign=b9bc7f57dd152206c7fa5bda46a4e89d HTTP/2
    Host: xyks.yuanfudao.com
    Leo-Client-Trace-Id: 7aw3t31nrlh8ew3tb5ct
    Default-Namespace-Sw8: MQ==-N2F3M3QzMW5ybGg4ZXczdGI1Y3Q=-MA==-0-X19PX1JfVF9f-UF9J-UF9F-SV9Q
    User-Agent: Leo/3.97.3 (RedmiM2012K11AC; Android 13; Scale/2.75)
    X-Xyks-Req-Timestamp: 1733843038630
    X-Xyks-Req-Network-Env: wifi
    X-Shepherd-Did: DUf7rsjrGNP-ECPjhpeluIhxJYDnB6FYhzg7
    Ci: 2
    Content-Type: application/x-www-form-urlencoded
    Content-Length: 392
    Accept-Encoding: gzip, deflate, br
    Cookie: ks_persistent=jXUi05a5kSoQoDcHRmp+e1HZ9q/RgkCcPI7ihWUFAQTRC3MllsBNczOaVLhthzPxc2+TeowlsB17P4PIpfXNQNgaBluClWXsgnAeqO6H3Uo=; sid=2446924330984759097; sess=x/qO0oR79WOnHPbIrYO7HBpKgEu4pnJlOuttyVNnJpPIzcvPtkwL7KpfYwcB2YbiUhVDjrybATMrRPG7/ZqIcNru9KvDS8GeuG0vNFAJqlI=; userid=1055361641; g_sess=x/qO0oR79WOnHPbIrYO7HBpKgEu4pnJlOuttyVNnJpPIzcvPtkwL7KpfYwcB2YbiUhVDjrybATMrRPG7/ZqIcA2ncdmptJwPxlbbUj3fDd67cWFPREUKCMSRhDdaENK1; __sub_user_infos__=/24T7Ovgtuud9LSYuz4IE/a06WwvM8rPZQO5PFO1a+2LNiZ95kgk1BpjYanVG9EveEiqnYyK7mMQugDnIaPolQ==; g_loc=vPeteFfrRL2jJ0VNIL3TWQ==; ks_sess=Lf7qQBSDsFG1HGWjWi3ZUx1dQgaz+LYTEGgFKM9McjWH+hR1deOYLksDQA8eR4XB; ks_deviceid=283347246
    
    phone=H2k%2FdHIVIsBVp6ZMWWtidqHw7k%2Bu3bjrseJLVV3TUGOVVwo9ckL2%2BA5vNJUlu7YTYWO3kUc%2Bkb8o%2Fd5tWnr6CtAQ%2B11acrNIUaK8nbxvp3fBZAUilRUpwJY7tr%2FGZzyFzx7evHBNkQzQx0fzbNBzJJyOoi1f6khO78epE3b2kQQ%3D&password=T%2BCEcKdQk41l1QskCvF8d1cEqDyVFc6AReASO7X8QS%2FH%2BWoIs00tYeDCck%2Bq7KTmhEiBF3qHkPujXOXa5dRYyhL4v7dlqTWjbVVgw%2FJTn99ZnMngQYgbF0%2Bc878XDJiEFcuUcjnjaUWH7sBSzT2fRCsoa2p9Q83If2O7ozcB%2Bog%3D

这个包传递了两个参数，phone 和 password，这下可以完全确定，这就是登录接口。可以看到，在请求报文中，是对这两个关键参数做了加密的，经过尝试使用 url decode 之后，再进行 base64 decode，解不出这两个值，那么很明显，这两个值是进行了应用层加密。我们猜不到加密算法，这个时候，只能进行反编译去找到登录方法，然后找到使用的加密算法，才能完全解开这个 API 接口。
## jadx 安装
jadx 是 Dex 到 Java 的反编译器，用于从 Android Dex 和 Apk 文件生成 Java 源代码的命令行和 GUI 工具。这款反编译工具是开源的，在 GitHub 上可以搜索到该工具，官方仓库地址：https://github.com/skylot/jadx，可以在这里查看 jadx 的更多介绍。现在我们在 releases 中下载最新版本的 jadx 即可。为了避免和本地的 JDK 环境混乱，一般我们下载带有 JRE 的版本即可。如下图：

![image](https://github.com/user-attachments/assets/e5b69adb-09ff-4918-b186-b696fe4db50b)

下载之后，直接解压，得到的文件如下图所示：

![image](https://github.com/user-attachments/assets/421b019e-87fb-4f28-9db8-3aaa7ab09c5a)

直接点击 exe 文件运行即可，之后选择打开文件，选择小猿口算 APP 的 apk 文件即可，等待反编译。（反编译比较费时间和资源，如果电脑配置较低，可能会反编译失败，主要是吃内存，CPU不行的话，反编译会较慢，因此一台高性能的电脑还是很有必要的）

等待反编译完成之后，我们可以按照下图中点击搜索操作，搜索我们刚才抓包获取的那个 URL （/leo-gateway/android/auth/password）

![image](https://github.com/user-attachments/assets/6900d7fd-521d-4443-bde8-d965036db935)

总共搜索到两个结果，如下图所示：

![image](https://github.com/user-attachments/assets/02b87496-affc-429f-88a9-c1830b42aef7)

第一个搜索结果看着非常像登录接口所在的地方，我们直接点击第一个结果，跳转到反编译之后的代码中，如下所示：

![image](https://github.com/user-attachments/assets/5ab39d28-e605-48f6-9b16-b5b6b6472cd4)

在这个文件中还包含了其他的登录方式的代码。不过这段代码很难看懂。首先这是一段 kotlin 代码，而不是 Java 代码。
