
## 装饰模式（Decorator Pattern）


**装饰模式**是一种结构型设计模式，旨在在不改变原有对象结构的情况下动态地为对象添加功能。通过将对象封装到一系列装饰器类中，可以以灵活和透明的方式扩展功能。


如果要扩展功能，装饰模式提供了`比继承`更有弹性的替代方案，装饰模式强调的是功能的扩展和灵活`组合`。


**装饰模式强调**的是`扩展对象的功能`及`扩展功能的组合`。比如，对象A，需要扩展X、Y的功能，这些扩展功能按不同的顺序组合可以实现不同的效果，先执行X再执行Y扩展功能或者先执行Y再执行X扩展功能。而继承实现的只是单一的顺序扩展功能，并且继承是单一的。看后面的例子就很好理解了。


## 结构


通常包含`一个核心对象`和`若干装饰对象`，这些装饰对象通过引用同一接口或抽象类实现功能的叠加。


装饰模式包含以下主要角色：


1. **Component（组件接口）**
定义核心对象的公共接口，允许动态添加行为。
2. **ConcreteComponent（具体组件）**
具体实现了 `Component` 接口的类，表示被装饰的核心对象。
3. **Decorator（装饰器抽象类）**
实现 `Component` 接口，同时持有一个 `Component` 的引用，表示对该对象的包装。
4. **ConcreteDecorator（具体装饰器）**
在装饰器抽象类的基础上，添加具体的功能。


## 代码示例


实现对`Socket`报文的`多层加密`，加密顺序可`随意组合`。示例中使用到了国密SM4加密和Base64编码。Socket 报文可以先加密再编码，也可以先编码在加密，跟根据不同组合来实现不同的增强。


### 类图


![image](https://img2024.cnblogs.com/blog/1209017/202412/1209017-20241226222951313-171300243.png)


### 定义核心接口



```
public interface SocketStream {
    void writeData(String data) throws IOException;
    String readData() throws IOException;
    void close() throws IOException;
}

```

### 被装饰的核心对象



```
public class SimpleSocketStream implements SocketStream {
    private Socket socket;
    private OutputStream outputStream;
    private InputStream inputStream;
    public SimpleSocketStream(Socket socket){
        this.socket = socket;
        try {
            this.outputStream = socket.getOutputStream();
            this.inputStream = socket.getInputStream();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }


    @Override
    public void writeData(String data) throws IOException {
        outputStream.write(data.getBytes(StandardCharsets.UTF_8));
        outputStream.flush();
        socket.shutdownOutput();
    }

    @Override
    public String readData() throws IOException {
        // 读取客户端请求并解密
        BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
        String message = reader.readLine();
        return message;
    }

    @Override
    public void close() throws IOException {
        if(inputStream!=null)
            inputStream.close();
        if(outputStream!=null)
            outputStream.close();
        if(socket!=null)
            socket.close();
    }
}

```

### 抽象装饰器



```
public abstract class SocketStreamDecorator implements SocketStream{
    private SocketStream socketStream;

    public SocketStreamDecorator(SocketStream socketStream){
        this.socketStream = socketStream;
    }

    @Override
    public void writeData(String data) throws IOException {
        socketStream.writeData(data);
    }

    @Override
    public String readData() throws IOException {
        return socketStream.readData();
    }

    @Override
    public void close() throws IOException {
        socketStream.close();
    }
}

```

### 具体装饰器1\-\-SM4 加密处理



```
public class SM4CipherSocketStreamDecorator extends SocketStreamDecorator{

    public SM4CipherSocketStreamDecorator(SocketStream socketStream) {
        super(socketStream);
    }

    @Override
    public void writeData(String data) throws IOException {
        // 加密
        String sm4Encrypt = SM4EncryptUtil.sm4Encrypt(data);
        System.out.println("--SM4加密数据："+sm4Encrypt);
        super.writeData(sm4Encrypt);
    }

    @Override
    public String readData() throws IOException {
        String readData = super.readData();
        // 解密
        System.out.println("--SM4解密前数据："+readData);
        String sm4Decrypt = SM4EncryptUtil.sm4Decrypt(readData);
        return sm4Decrypt;
    }
}

```

**工具类**



```
public class SM4EncryptUtil {
    // 秘钥
    private static final String key = "1234567812345678";

    static {
        // 添加安全提供者（SM2，SM3，SM4等加密算法，CBC、CFB等加密模式，PKCS7Padding等填充方式，不在Java标准库中，由BouncyCastleProvider实现）
        Security.addProvider(new BouncyCastleProvider());
    }

    /**
     * 输入：待加密的字符串，16或24或32位字符串密码
     * 输出：16进制字符串或Base64编码的字符串密文（常用）
     */
    public static String sm4Encrypt(String encrypt) {
        String cipherString = null;
        try {
            // 指定加密算法
            String algorithm = "SM4";
            // 创建密钥规范
            SecretKeySpec secretKeySpec = new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), algorithm);
            // 获取Cipher对象实例（BC中SM4默认使用ECB模式和PKCS5Padding填充方式，因此下列模式和填充方式无需指定）
            Cipher cipher = Cipher.getInstance(algorithm + "/ECB/PKCS5Padding");
            // 初始化Cipher为加密模式
            cipher.init(Cipher.ENCRYPT_MODE, secretKeySpec);
            // 获取加密byte数组
            byte[] cipherBytes = cipher.doFinal(encrypt.getBytes(StandardCharsets.UTF_8));
            // 输出为Base64编码
            cipherString = Base64.getEncoder().encodeToString(cipherBytes);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return cipherString;
    }

    /**
     * 输入：密文，16或24或32位字符串密码
     * 输出：明文
     */
    public static String sm4Decrypt(String cipherString) {
        String plainString = null;
        try {
            // 指定加密算法
            String algorithm = "SM4";
            // 创建密钥规范
            SecretKeySpec secretKeySpec = new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), algorithm);
            // 获取Cipher对象实例（BC中SM4默认使用ECB模式和PKCS5Padding填充方式，因此下列模式和填充方式无需指定）
            Cipher cipher = Cipher.getInstance(algorithm + "/ECB/PKCS5Padding");
            // 初始化Cipher为解密模式
            cipher.init(Cipher.DECRYPT_MODE, secretKeySpec);
            // 获取加密byte数组
            byte[] cipherBytes = cipher.doFinal(Base64.getDecoder().decode(cipherString));
            // 输出为字符串
            plainString = new String(cipherBytes);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return plainString;
    }

}

```

### 具体装饰器2\-\-Base64 编码处理



```
public class Base64SocketStreamDecorator extends SocketStreamDecorator{

    public Base64SocketStreamDecorator(SocketStream socketStream) {
        super(socketStream);
    }

    @Override
    public void writeData(String data) throws IOException {
        String encode = this.encode(data);
        System.out.println("--base64编码后的数据：" + encode);
        super.writeData(encode);
    }

    @Override
    public String readData() throws IOException {
        String readData = super.readData();
        System.out.println("--base64解密前的数据：" + readData);
        String decode = this.decode(readData);
        return decode;
    }
    /**
     * base64编码
     */
    public String encode(String str) {
        if (str == null || str.isEmpty()) {
            return "";
        }
        // String 转 byte[]
        byte[] bytes = str.getBytes(StandardCharsets.UTF_8);
        // 编码（base64字符串）
        return Base64.getEncoder().encodeToString(bytes);
    }

    /**
     * base64解码
     */
    public String decode(String base64Str) {
        if (base64Str == null || base64Str.isEmpty()) {
            return "";
        }
        // 编码
        byte[] base64Bytes = Base64.getDecoder().decode(base64Str);
        // byte[] 转 String（解码后的字符串）
        return new String(base64Bytes, StandardCharsets.UTF_8);
    }

}

```

### 测试代码


测试Socket 服务端代码



```
public class SM4SocketServer {

    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(9999);
        System.out.println("Server started...");

        while (true) {
            try (Socket socket = serverSocket.accept()) {
                System.out.println("Client connected...\n");

                // 加密数据流
                SocketStream socketStreamDecorator = new Base64SocketStreamDecorator(
                                                        new SM4CipherSocketStreamDecorator(
                                                          new SimpleSocketStream(socket)));
                // 读取客户端请求并解密
                String message = socketStreamDecorator.readData();
                System.out.println("服务端读取数据: " + message);

                // 处理请求并加密响应
                String response = "我来自服务端！";
                System.out.println("服务端发送数据: " + response);
                socketStreamDecorator.writeData(response);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}

```

测试Socket 客户端代码



```
public class SM4SocketClient {

    public static void main(String[] args) throws IOException {
        // 连接到服务器
        try (Socket socket = new Socket("localhost", 9999)) {
            System.out.println("Connected to server...\n");

            // 读取服务器的响应并解密
            SocketStream socketStreamDecorator = new Base64SocketStreamDecorator(
                                                    new SM4CipherSocketStreamDecorator(
                                                      new SimpleSocketStream(socket)));
            // 要发送的消息
            String message = "我来自客户端！";
            System.out.println("客户端发送数据: " + message);

            // 使用加密流将消息发送到服务器
            socketStreamDecorator.writeData(message);

            // 读取客户端请求并解密
            String readData = socketStreamDecorator.readData();
            System.out.println("客户端读取数据: " + readData);
            socketStreamDecorator.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

### 测试结果分析


先启动Socket服务，再发起Socket请求。


客户端输出结果



> Connected to server...
> 
> 
> 客户端发送数据: 我来自客户端！
> 
> 
> \-\-base64编码后的数据：5oiR5p2l6Ieq5a6i5oi356uv77yB
> 
> 
> \-\-SM4加密数据：anTSZv18wcQOxQtA82w9ap2j8DjagsX9QqRVjzFCjWU\=
> 
> 
> \-\-SM4解密前数据：5azwyKP7MB6jSyG3eDlJmfOYx5y\+woud2WgE2Jjf5lA\=
> 
> 
> \-\-base64解密前的数据：5oiR5p2l6Ieq5pyN5Yqh56uv77yB
> 
> 
> 客户端读取数据: 我来自服务端！


服务端输出结果



> Server started...
> 
> 
> Client connected...
> 
> 
> \-\-SM4解密前数据：anTSZv18wcQOxQtA82w9ap2j8DjagsX9QqRVjzFCjWU\=
> 
> 
> \-\-base64解密前的数据：5oiR5p2l6Ieq5a6i5oi356uv77yB
> 
> 
> 服务端读取数据: 我来自客户端！
> 
> 
> 服务端发送数据: 我来自服务端！
> 
> 
> \-\-base64编码后的数据：5oiR5p2l6Ieq5pyN5Yqh56uv77yB
> 
> 
> \-\-SM4加密数据：5azwyKP7MB6jSyG3eDlJmfOYx5y\+woud2WgE2Jjf5lA\=


这行代码new的先后顺序决定代码执行的先后顺序：`new Base64SocketStreamDecorator(new SM4CipherSocketStreamDecorator(new SimpleSocketStream(socket)));` ，所以文中的案例测试执行顺序如下：


**数据流方向**


* **写入数据时**：外层到内层逐步加工数据。
	+ 数据先经过 `Base64` 编码，再经过 `SM4` 加密，最后写入 `SimpleSocketStream`。
* **读取数据时**：内层到外层逐步还原数据。
	+ 数据从 `SimpleSocketStream` 读取后，先解密（SM4），再解码（Base64）。


**代码层面的调用顺序**


* **构造阶段**：
	+ `SimpleSocketStream(socket)` → `SM4CipherSocketStreamDecorator` → `Base64SocketStreamDecorator`。
* **方法调用阶段**（如 `write(data)` 或 `read()`）：
	+ 调用 `Base64SocketStreamDecorator.write(data)`。
	+ 该方法内部会调用 `SM4CipherSocketStreamDecorator.write(data)`。
	+ 最终调用 `SimpleSocketStream.write(data)` 执行实际的数据写入。


### 调整组合顺序测试


如果哪天有需求要改为先执行SM4加密，再执行Base64编码，则代码只需要修改为`new SM4CipherSocketStreamDecorator(new Base64SocketStreamDecorator(new SimpleSocketStream(socket)));`


测试结果变为：


客户端输出结果



> 客户端发送数据: 我来自客户端！
> 
> 
> \-\-SM4加密数据：YoNCZhEm1shIjPWO5vyhsFRWq\+XUdHPggFiE325GCKc\=
> 
> 
> \-\-base64编码后的数据：WW9OQ1poRW0xc2hJalBXTzV2eWhzRlJXcStYVWRIUGdnRmlFMzI1R0NLYz0\=
> 
> 
> \-\-base64解密前的数据：OUJTQ0ZLUVFIa2dxSGs0WTd2enB1RlJXcStYVWRIUGdnRmlFMzI1R0NLYz0\=
> 
> 
> \-\-SM4解密前数据：9BSCFKQQHkgqHk4Y7vzpuFRWq\+XUdHPggFiE325GCKc\=
> 
> 
> 客户端读取数据: 我来自服务端！


服务端输出结果变为



> \-\-base64解密前的数据：WW9OQ1poRW0xc2hJalBXTzV2eWhzRlJXcStYVWRIUGdnRmlFMzI1R0NLYz0\=
> 
> 
> \-\-SM4解密前数据：YoNCZhEm1shIjPWO5vyhsFRWq\+XUdHPggFiE325GCKc\=
> 
> 
> 服务端读取数据: 我来自客户端！
> 
> 
> 服务端发送数据: 我来自服务端！
> 
> 
> \-\-SM4加密数据：9BSCFKQQHkgqHk4Y7vzpuFRWq\+XUdHPggFiE325GCKc\=
> 
> 
> \-\-base64编码后的数据：OUJTQ0ZLUVFIa2dxSGs0WTd2enB1RlJXcStYVWRIUGdnRmlFMzI1R0NLYz0\=


## 装饰模式的应用


Java标准库中的装饰模式之一：**IO流体系**


示例代码



```
BufferedReader bufferedReader = new BufferedReader(
                                          new InputStreamReader(
                                            new FileInputStream(
                                              new File("test.txt"))));

```

**逐步增强功能**：IO流的增强组合有顺序要求


* 每一层包装器都增加了功能：
	+ `File`：创建一个表示文件的对象 `test.txt`；
	+ `FileInputStream`：从文件中读取字节；
	+ `InputStreamReader`：将字节流转换为字符流；
	+ `BufferedReader`：提供字符缓冲功能，支持高效读取和按行读取。


## 总结


装饰模式是一种灵活的结构型模式，允许我们在不修改现有类的情况下，动态地添加功能。它尤其适合于功能可以按需组合叠加、扩展的场景，但需要权衡复杂性与性能开销。


![image](https://img2024.cnblogs.com/blog/1209017/202412/1209017-20241226223007703-965492480.gif)


[什么是设计模式？](https://github.com)


[单例模式及其思想](https://github.com)


[设计模式\-\-原型模式及其编程思想](https://github.com)


[掌握设计模式之生成器模式](https://github.com)


[掌握设计模式之简单工厂模式](https://github.com)


[掌握设计模式之工厂方法模式](https://github.com)


[超实用的SpringAOP实战之日志记录](https://github.com):[楚门加速器](https://shexiangshi.org)


[2023年下半年软考考试重磅消息](https://github.com)


[通过软考后却领取不到实体证书？](https://github.com)


[计算机算法设计与分析（第5版）](https://github.com)


[Java全栈学习路线、学习资源和面试题一条龙](https://github.com)


[软考证书\=职称证书？](https://github.com)


[软考中级\-\-软件设计师毫无保留的备考分享](https://github.com)


