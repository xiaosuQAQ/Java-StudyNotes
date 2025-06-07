Servlet是用于Web开发的工具，简而言之Web开发就是需要编写在服务器上运行的代码，可以接受浏览器请求、响应返回数据，也就是典型的B/S架构。

首先来看一下使用网络编程中学到的内容来编写一个程序，使得服务器在收到浏览器请求时可以自动解析请求并返回对应的结果。需要一个ServerSocket来监听指定的接口、如果收到请求就响应得到一个Socket对象，通过这个Socket对象可以获得输入流、输出流，从而实现请求的解析和响应，考虑到多线程访问服务器，因此这里通过线程对象来实现网络流的处理。那么这部分的代码可以写作：

```Java
public class Server {
    public static void main(String[] args) throws IOException {
        ServerSocket ss = new ServerSocket(8080);	// 监听8080接口
        System.out.println("server is running");
        for(;;) {
            Socket sock = ss.accept();			// 收到请求就返回Socket对象
            System.out.println("connected from " + sock.getInetAddress());
            Thread t = new Handler(sock);		// 线程对象来解析、响应请求
            t.start();
        }
    }
}
```

接下来是线程对象的实现，线程对象会接受一个Socket对象，通过获得输入流实现请求的解析，通过输出流来实现返回响应，整体的思路很简单，需要注意解析时要根据**请求标头**来判断是否合法。

那么如果我们向服务器发送一个请求，典型的请求标头见下：

![典型请求标头](imgs/请求标头.png)
