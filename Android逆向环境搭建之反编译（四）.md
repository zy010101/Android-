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

在这个文件中还包含了其他的登录方式的代码。不过这段代码很难看懂。首先这是一段 kotlin 代码，而不是 Java 代码。没关系，我们有 AI 大模型，我们不懂的它懂。

我们右键一下这个 passwordLogin ，然后点击查找用例，就会跳转到调用的地方，如下图所示，我们可以看到 a12, a11 实际上对应的就是 phone 和 password.

![image](https://github.com/user-attachments/assets/c8dae1b9-b8ef-4681-95ef-e79e73a68651)

剩下的代码我们完全看不懂，不知道在干什么，现在把这段代码丢到大模型，让他解释一下。

![image](https://github.com/user-attachments/assets/c537bb07-0e60-4301-b5c2-cacb84e8e97f)

大模型告诉我们的结果比较模糊，我们并不明白 `String a11 = wg.i.a(((RichInputCell) aVar2.x(aVar2, com.fenbi.android.leo.business.user.c.password_cell, RichInputCell.class)).getInputText());` 是在干什么？单独把这行丢给大模型，继续询问，大模型给出回复如下：

![image](https://github.com/user-attachments/assets/5206476a-80d9-4b0e-b432-5c77a28c7848)

大模型告诉我们， 获取密码输入框中的文本，并将其加密后存储在 a11 变量中。我们可以在下一篇的 hook 中验证一下大模型的判断是否正确。

既然 `wg.i.a` 可能是加密算法，我们点击这个 a 直接跳转到对应的方法，然后看一下具体的代码。我把代码贴在这里

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
简单扫一下这段代码，确实是在加密，而且是使用了 RSA 加密（RSA/ECB/PKCS1PADDING），这段代码看起来很别扭，我们交给 AI 大模型，让它把这段代码翻译成 python 代码。下面是大模型给出的 python 代码
```python
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding
import base64

# 公钥 Base64 编码字符串（与 Java 代码中的 f69169a 相同）
public_key_pem = """
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDSovT1rrwzrGoMCFb6z8e+5lzVdAD5o8krGIwdfxrVE2OnMijUZdkQk7etPJvZ2JOVXghthAGUUJkDUE8n2ZMNFKPjMrQJI49ewVzqWOKOvgU6Iu60Sn0xpeietP1wWXBkszdV1WfNBJUo2hhPDnIPMGzzdfLW5rMu+tczeUriJQIDAQAB
-----END PUBLIC KEY-----
"""

# 加载公钥
public_key = serialization.load_pem_public_key(public_key_pem.encode('utf-8'))

# 加密方法
def rsa_encrypt(message: str, public_key):
    # 将消息转换为字节
    message_bytes = message.encode('utf-8')

    # 使用RSA公钥加密
    encrypted = public_key.encrypt(
        message_bytes,
        padding.PKCS1v15()  # 使用PKCS1填充
    )

    # 将加密后的字节转换为Base64编码的字符串
    encrypted_base64 = base64.b64encode(encrypted).decode('utf-8')

    return encrypted_base64

# 使用RSA公钥加密字符串
encrypted_message = rsa_encrypt("15925833452", public_key)
print("Encrypted message:", encrypted_message)
```

我们可以使用这段代码作为之后生成phone 和 password 的代码。到这里的时候，笔者已经迫不及待的想去尝试重放报文，然后就重复失败了。因为这个报文中的 query 参数中的 sign 是每次都变化的，还有请求头中的 Leo-Client-Trace-Id 和 Default-Namespace-Sw8 中的 MQ 也是变化的，还有 cookie 中的 ks_sess 也是每次都变化的。我们现在还不能确定到底是哪个参数或者哪些参数的变化导致了报文重放失败。我们先来看一下 query 中的 sign 参数吧！看看它是如何生成的。

直接启动搜索大法，在 jadx 反编译之后的结果中搜索关键字 sign，得到的结果如下所示：

![image](https://github.com/user-attachments/assets/3639be83-7329-4bb1-9ab1-76d033e81a69)

注意到有一个静态变量的值是 sign ，我们直接跳转到这里。

![image](https://github.com/user-attachments/assets/e39b9944-819b-46e8-84a1-c291e968b140)

直接查找用例，果然找到了使用的地方

![image](https://github.com/user-attachments/assets/87791f6c-9346-46f3-8666-e729bd1d9830)

![image](https://github.com/user-attachments/assets/10497021-fec6-4d56-aea4-9f64f564aad7)

果然，这段代码里设置了 query 参数 sign, `host.addQueryParameter(SIGN, b11).build()).build()`，b11 就是 sign 的值，而 b11 是 `b11 = h3.c().b(request.url().encodedPath())) == null` 生成的，我们先去看 h3.c().b 这个方法干了什么？

![image](https://github.com/user-attachments/assets/f9901c90-29ec-4d95-bfa2-c41dd1aab957)

![image](https://github.com/user-attachments/assets/0f26c784-1939-4f76-8fe6-8bd228f79d7f)

OK，zcvsd1wr2t 是个 native 方法，是 RequestEncoder 这个库实现的。 如果你不懂 Java 的话，还是要回头补习 Java 的。看到这段代码顿感大事不妙，这是通过 JNI 调用了本地共享库 RequestEncoder 中的代码，sdwioxccsd() 和 zcvsd1wr2t() 都是本地方法，它们的实现并不在 Java 中。我们打开 jadx 左边的导航栏，可以看到有个 lib, 里面就放着本地共享库。

![image](https://github.com/user-attachments/assets/45e32dc0-afe7-4a99-b7ab-5708f5c4ba6c)

在这里，能找到一个名为 libRequestEncoder.so 的文件，导出这个文件即可。至于如何处理这个库文件，我们之后在学习。下一章节，我们先来 hook 验证一下我们之前的 phone 和 password 的实现代码是否真的正确，看看 `String a11 = wg.i.a(((RichInputCell) aVar2.x(aVar2, com.fenbi.android.leo.business.user.c.password_cell, RichInputCell.class)).getInputText());` 这段代码是否获取的就是明文的数据。
