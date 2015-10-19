---
layout: post
title: ArchLinux 上使用shadowsocks服务
---

###服务端 

安装shadowsocks    
```bash
pacman -S shadowsocks
```

创建一个配置文件config.json，在/etc/shadowsocks 目录下有个examle.json作为模板  
```
{
    "server":"my_server_ip",
    "server_port":8388,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"mypassword",
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open": false,
    "workers": 1
}
```

作为服务端的话，local_address和local_port不用管他，在server和server_port填上ip和端口，password设置密码，timeout设置超时时间，method设置加密方式，推荐适用ChaCha20，速度比较快，也可以使用默认的 aes-256-cfb。

然后在config的目录下运行服务  
```bash
ssserver
```

如果要在后台运行  
```bash
nohup ssserver > log &
```

###客户端

可以直接安装aur中的shadowsocks-qt5，按照服务端的设置进行配置即可。

###可以使用的加密方式

* aes-256-cfb: 默认加密方式
* aes-128-cfb
* aes-192-cfb
* aes-256-ofb
* aes-128-ofb
* aes-192-ofb
* aes-128-ctr
* aes-192-ctr
* aes-256-ctr
* aes-128-cfb8
* aes-192-cfb8
* aes-256-cfb8
* aes-128-cfb1
* aes-192-cfb1
* aes-256-cfb1
* bf-cfb
* camellia-128-cfb
* camellia-192-cfb
* camellia-256-cfb
* cast5-cfb
* chacha20
* idea-cfb
* rc2-cfb
* rc4-md5
* salsa20
* seed-cfb

要使用chacha20和salsa20的话需要安装libsodium  
```bash
pacman -S libsodium
```
