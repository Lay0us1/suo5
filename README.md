<h1 align="center">Suo5</h1>

<p align="center">一款高性能 HTTP 代理隧道工具</p>

<div align="center">

![License](https://img.shields.io/github/license/zema1/suo5)
![Workflow Status](https://img.shields.io/github/actions/workflow/status/zema1/suo5/release.yml?label=release)
![Workflow Status](https://img.shields.io/github/actions/workflow/status/zema1/suo5/test.yml?label=test)
![Latest release](https://img.shields.io/github/v/release/zema1/suo5?label=latest)

</div>

![experience](./tests/img/suo5.gif)

[中文](./README.md) | [English](./README_EN.md)

----

`suo5` 是一个全新的 HTTP 代理隧道，基于 `HTTP/1.1` 的 `Chunked-Encoding`
构建。相比 [Neo-reGeorg](https://github.com/L-codes/Neo-reGeorg) 等传统隧道工具, `suo5`
的性能可以达到其数十倍。查看 [性能测试](./tests)

其主要特性如下：

- 同时支持全双工与半双工模式，传输性能接近 FRP
- 支持在 Nginx 反向代理和负载均衡场景使用
- 完善的连接控制和并发管理，使用流畅丝滑
- 支持 Tomcat Jetty Weblogic Resin 等主流中间件
- 支持 Java4 ~ Java 19 全版本
- 同时提供提供命令行和图形化界面

原理介绍 [https://koalr.me/posts/suo5-a-hign-performace-http-socks/](https://koalr.me/posts/suo5-a-hign-performace-http-socks/)

> 免责声明：此工具仅限于安全研究，用户承担因使用此工具而导致的所有法律和相关责任！作者不承担任何法律责任！

## 安装运行

前往 [Releases](https://github.com/zema1/suo5/releases) 下载编译好的二进制，其中带 `gui` 的版本是界面版，不带 `gui`
的为命令行版。所有编译由 Github Action 自动构建，请放心使用。

使用时需上传 [suo5.jsp](./assets/) 到目标环境中并确保可以执行。

### 界面版

界面版基于 [wails](https://github.com/wailsapp/wails) 实现，依赖 Webview2 框架。Windows 11 和 MacOS
已自带该组件，其他系统会弹框请允许下载安装，否则无法使用。

![gui.png](tests/img/gui.jpg)

### 命令行

```text
USAGE:
   suo5 [global options] command [command options] [arguments...]

VERSION:
   v0.6.0

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --target value, -t value                               set the remote server url, ex: http://localhost:8080/tomcat_debug_war_exploded/
   --listen value, -l value                               set the listen address of socks5 server (default: "127.0.0.1:1111")
   --method value, -m value                               http request method (default: "POST")
   --redirect value, -r value                             redirect to the url if host not matched, used to bypass load balance
   --no-auth                                              disable socks5 authentication (default: true)
   --auth value                                           socks5 creds, username:password, leave empty to auto generate
   --mode value                                           connection mode, choices are auto, full, half (default: "auto")
   --ua value                                             the user-agent used to send request (default: "Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.1.2.3")
   --header value, -H value [ --header value, -H value ]  use extra header, ex -H 'Cookie: abc'
   --timeout value                                        http request timeout in seconds (default: 10)
   --buf-size value                                       set the request max body size (default: 327680)
   --proxy value                                          use upstream socks5 proxy
   --debug, -d                                            debug the traffic, print more details (default: false)
   --no-heartbeat, --nh                                   disable heartbeat to the remote server which will send data every 5s (default: false)
   --help, -h                                             show help
   --version, -v                                          print the version
```

命令行版本与界面版配置完全一致，可以对照界面版功能来使用，最简单的只需指定连接目标

```bash
$ ./suo5 -t https://example.com/proxy.jsp
```

使用 `GET` 方法发送请求，有时可以绕过限制

```bash
$ ./suo5 -m GET -t https://example.com/proxy.jsp
```

自定义 socks5 监听在 `0.0.0.0:7788`，并自定义认证信息为 `test:test123`

```bash
$ ./suo5 -t https://example.com/proxy.jsp -l 0.0.0.0:7788 --auth test:test123
```

负载均衡场景下将流量转发到某一个固定的 url 解决请求被分散的问题 (需要尽可能的在每一个后端服务中上传 suo5)

```bash
$ ./suo5 -t https://example.com/proxy.jsp -t http://172.0.3.2/code/proxy.jsp
```

### 特别提醒

`User-Agent` (`ua`) 的配置本地端与服务端是绑定的，如果修改了其中一个，另一个也必须对应修改才能连接上。

## 常见问题

1. 什么是全双工和半双工?

   **全双工** 仅需发送一个 HTTP 请求即可构建出一个 HTTP 隧道, 实现 TCP 的双向通信。可以理解成这个请求既是一个上传请求又是一个下载请求，只要连接不断开
   ，就会一直下载，一直上传, 便可以借此做双向通信。

   **半双工** 在部分场景下不支持 `全双工` 模式（比如有反代），可以退而求其次做半双工，即发送一个请求构建一个下行的隧道，同时用短链接发送上行数据一次来完成双向通信。

2. `suo5` 和 `Neo-reGeorg` 怎么选？

   如果目标是 Java 的站点，可以使用 `suo5` 来构建 http 隧道，大多数情况下 `suo5` 都要比 `neo` 更稳定速度更快。但 `neo`
   提供了非常多种类的服务端支持，兼容性很好，而且也支持一些 `suo5` 当前还在开发的功能，也支持更灵活的定制化。

## 接下来

- [x] 支持配置上游 socks 代理
- [x] 支持负载均衡的场景
- [ ] 支持 .Net 的类型

## 参考

- [https://github.com/L-codes/Neo-reGeorg](https://github.com/L-codes/Neo-reGeorg)
- [https://github.com/BeichenDream/Chunk-Proxy](https://github.com/BeichenDream/Chunk-Proxy)

## 其他

扫码加我拉进讨论群，请备注申请理由为：问题反馈与讨论，否则不通过

![wechat](https://user-images.githubusercontent.com/20637881/229296623-24a7cc5d-8f33-4120-ae77-bbe2610ce5bb.jpg)
