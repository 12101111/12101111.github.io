+++
title = "逆向Speedtest.net 的API和协议"
date = 2019-09-26
[taxonomies]
categories = ["Network"]
tags = ["5G","Rust"]
+++

目前在国内乃至全球范围内能跑满5G移动网络全速下载的服务器也没有几个,然而Speedtest.net是一个例外.

处于营销等目的,全国各地移动联通电信架设了多个(63个)Speedtest的测速服务器,其中命名包含5G的有15个(而且全球仅中国的服务器如此),可见运营商大概率在宣传5G网速时也会使用自己架设的Speedtest服务器,以接近理论上的峰值速度.

处于自动化测速的考虑,需要研究Speedtest的测速机制并且命令行脚本化.

<!-- more -->

## 对官网的抓包

打开Speedtest.net,打开Chrome的开发人员工具,切换到`Network`标签页,点击筛选图标,发现如下API:

### `https://www.speedtest.net/api/js/servers?engine=js`

    可选参数: `https_functional=1` 加和不加返回的值似乎不太一样
    方法: `GET`
    返回格式: JSON
    内容: 距客户端最近的20个测速服务器的参数的数组

格式:

```JSON
[{
    "url": "http:\/\/5gtest.hl.chinamobile.com\/speedtest\/upload.php",
    "lat": "45.7500",
    "lon": "126.6333",
    "distance": 11,
    "name": "Harbin",
    "country": "China",
    "cc": "CN",
    "sponsor": "ChinaMobile-HeiLongjiang",
    "id": "26656",
    "preferred": 0,
    "host": "5gtest.hl.chinamobile.com:8080"
}, {
    "url": "http:\/\/speedtest2.jl.chinamobile.com:8080\/speedtest\/upload.php",
    "lat": "43.8800",
    "lon": "125.3228",
    "distance": 134,
    "name": "Changchun",
    "country": "China",
    "cc": "CN",
    "sponsor": "China Mobile,Jilin",
    "id": "16375",
    "preferred": 0,
    "https_functional": 1,
    "host": "speedtest2.jl.chinamobile.com:8080"
},...]
```

### wss://测速服务器host/ws

    例如: wss://speedtest2.jl.chinamobile.com:8080/ws

消息内容:

客户端|服务器|解释
-|-|-
HI|HELLO 2.7 (2.7.4) 2019-09-17.2311.4b191db|服务器软件版本号
GETIP|YOURIP 111.40.48.197|获得客户端IP
CAPABILITIES|CAPABILITIES SERVER_HOST_AUTH UPLOAD_STATS|获取服务器能力?
PING |PONG 1569507473389|(注意PING后有一个空格)服务器返回UNIX时间戳
PING 1569507472890|PONG 1569507473389
PING 1569507472912|PONG 1569507473411
PING 1569507472935|PONG 1569507473434|客户端会发起3次PING,最终被选中的服务器还会发起多次PING

### https://ipv6-api.speedtest.net/getip

方法: `GET`

返回了客户端的IPv6地址

### https://测速服务器host/hello

方法: `GET`

参数:

    nocache=随机UUID
    guid=SessionGUID

实际测试服务器不检查nocache=UUID(正如其名,防止运营商缓存内容)

guid用于提交结果,应在一次测试中不变.

响应:`hello 2.7 (2.7.4) 2019-09-17.2311.4b191db`即服务器版本号

### https://测速服务器host/download

方法: `GET`

参数:

    nocache=随机UUID
    guid=SessionGUID
    size=25000000

size会出现在响应头的Content-Length上,建议保持默认25MB.

下载的内容为随机的字符串

当使用多线程时,网页会以4-6线程发起GET请求,当23.8MiB下载完成时再次请求内容,但当15s固定时间到时会停止所有的下载.

### https://测速服务器host/upload

方法: `POST`

参数

    nocache=随机UUID
    guid=SessionGUID

建议Header:

    Content-type: application/octet-stream
    Content-Length: 目标上传大小

POST body中可以是任意随机字符串

响应为:

    size=收到的字节数

### https://www.speedtest.net/api/results.php

方法: `POST`

表格数据: 

```
serverid: 16167
testmethod: wss,xhrs,xhrs
hash: c6de8e4d1142ae6368c474247913822f
source: st4-js
configs[latencyProtocol]: ws
configs[downloadProtocol]: xhr
configs[uploadProtocol]: xhr
configs[host]: speedtest1.ln.chinamobile.com.prod.hosts.ooklaserver.net
configs[port]: 8080
configs[serverVersion]: 2.7.4
configs[serverBuild]: 2019-09-17.2311.4b191db
ping: 18
jitter: 0.6666666666666666
upload: 760127
uploadSpeeds[remote][combined]: 95464417
uploadSpeeds[remote][average]: 91299851.36765888
uploadSpeeds[remote][mst_66_20]: 95125068
uploadSpeeds[remote][mst_66_30]: 95487869.89473684
uploadSpeeds[remote][mst_75_30]: 94791001.52380952
uploadSpeeds[remote][superspeed]: 96913920
uploadSpeeds[local][combined]: 95015855
uploadSpeeds[local][average]: 88915587.36009651
uploadSpeeds[local][mst_66_20]: 95222898.44444445
uploadSpeeds[local][mst_66_30]: 95015855.26315789
uploadSpeeds[local][mst_75_30]: 94903589.14285715
uploadSpeeds[local][superspeed]: 96372459.86666666
download: 186022
downloadSpeeds[local][combined]: 23252766
downloadSpeeds[local][average]: 22400172.935841
downloadSpeeds[local][mst_66_20]: 23357582.111111112
downloadSpeeds[local][mst_66_30]: 23252766.10526316
downloadSpeeds[local][mst_75_30]: 23397916
downloadSpeeds[local][superspeed]: 23957890.133333333
guid: 153842d5-114b-4004-a282-3de942a6b66f
closestPingDetails: [{"server":9484,"jitter":2,"latency":27},{"server":16375,"jitter":1,"latency":24},{"server":16167,"jitter":1,"latency":16},{"server":25204,"jitter":2,"latency":154},{"server":6316,"jitter":3,"latency":244},{"server":10177,"jitter":0,"latency":371},{"server":7403,"jitter":0,"latency":413},{"server":6375,"jitter":1011.5,"latency":475},{"server":3805,"error":{"message":"Error: failed latency test"}},{"server":20951,"jitter":2.5,"latency":494}]
clientip: 111.40.48.197
ip6Address: 2001::....
connections[isVpn]: false
connections[selectionMethod]: auto
connections[mode]: multi
```

## TCP && Websocket 协议

根据一篇[文章](https://gist.github.com/sdstrowes/411fca9d900a846a704f68547941eb97),Speedtest除了具有HTTP XHR接口还存在TCP协议.

TCP协议和Websocket协议经测试基本相同,因此一同介绍.

此协议为文字协议(就像HTTP 1一样),因此可以使用命令行访问,TCP应使用netcat `nc` 程序,Websocket应使用`wscat`程序.

`wscat`可以使用`npm install -g wscat`安装.

连接到测速服务器:

```bash
nc 5gtest.hl.chinamobile.com 8080
# 或者
wscat -c wss://speedtest2.jl.chinamobile.com:8080/ws
```

客户端|服务器|解释
-|-|-
HI|HELLO 2.7 (2.7.4) 2019-09-17.2311.4b191db|服务器软件版本号
GETIP|YOURIP 111.40.48.197|获得客户端IP
CAPABILITIES|CAPABILITIES SERVER_HOST_AUTH UPLOAD_STATS|获取服务器能力?
PING+空格+任意字符串或为空|PONG 1569507473389|服务器返回UNIX时间戳
DOWNLOAD 字节数|DOWNLOAD 随机字符串|下载内容,服务器会返回要求的字节数(包括DOWNLOAD本身)
DOWNLOAD 5|DOWN\n|因此不足8字节时会返回不完整的单词
UPLOAD 随机字符串|OK 字节数 时间戳|字节数包括UPLOAD以及换行符
QUIT|仅用于TCP下,用于断开TCP链接

## XML数据

Speedtest还具有两个XML接口的API

    https://www.speedtest.net/speedtest-config.php 客户端配置
    http://www.speedtest.net/speedtest-servers-static.php 所有的服务器参数列表

## 客户端

根据TCP协议,目前编写/改造了一个命令行客户端:https://github.com/12101111/speedtestr
