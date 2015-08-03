---
layout: post
title: SO_REUSERPORT负载均衡
---

在kernel 3.9中，添加了SO_REUSEPORT选项，支持多个进程或线程绑定在同一个IP和端口上，可以对端口流量进行负载均衡，提升server性能。本文以UDP为例，分析该选项查找socket的算法。

UDP中查找socket的主要函数为 __udp4_lib_lookup，在就之前的版本中，该函数的主要代码如下
```cpp
result = NULL;
badness = -1;
sk_nulls_for_each_rcu(sk, node, &hslot->head) {
	score = compute_score(sk, net, saddr, hnum, sport,
			      daddr, dport, dif);
	if (score > badness) {
		result = sk;
		badness = score;
	}
}
```
该函数对socket链表进行遍历，找到匹配ip地址和端口的socket并返回
加入SO_REUSEPORT选项后，代码如下
```cpp
result = NULL;
badness = 0;
sk_nulls_for_each_rcu(sk, node, &hslot->head) {
    score = compute_score(sk, net, saddr, hnum, sport,
                  daddr, dport, dif);
    if (score > badness) {
        result = sk;
        badness = score;
        reuseport = sk->sk_reuseport;
        if (reuseport) {
            hash = inet_ehashfn(net, daddr, hnum,
                        saddr, htons(sport));
            matches = 1;
        }
    } else if (score == badness && reuseport) {
        matches++;
        if (((u64)hash * matches) >> 32 == 0)
            result = sk;
        hash = next_pseudo_random32(hash);
    }
}
```
在匹配到合适的目的ip和目的端口后，又加入了源ip和源端口计算hash，通过hash确定socket的位置。
这样的优点在于
1. 对于同一个源ip和源端口发出的数据，可以保证被同一个socket接受
2. 对于不同的源，流量会被均衡到多个socket中，提升服务性能 

同样存在一些不足
1. 服务运行过程中要避免socket的数量发生变化而导致socket位置发生变法，使接受的数据错乱
2. 网上有人指出在高负荷的情况会导致负载不均衡，不过我自己测试中并没有发生这种现象。

一个负载均衡udp服务的实例
```cpp
#include <sys/types.h>
#include <string.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <sys/uio.h>

#define LPORT 8000
#define NUM_PROCESSES 2
#define BUFFSIZE 1024
#define SO_REUSEPORT 15


int new_sock(char* str_addr)
{
	int sockfd;
	int optval;
	struct sockaddr_in seraddr;
	sockfd = socket(AF_INET, SOCK_DGRAM, 0);
	setsockopt(sockfd, SOL_SOCKET, SO_REUSEPORT, &optval, sizeof(optval));
	bzero(&seraddr, sizeof(seraddr));
	seraddr.sin_family= AF_INET;
	seraddr.sin_addr.s_addr = inet_addr(str_addr);
	seraddr.sin_port = htons(LPORT);
	if (bind(sockfd, (struct sockaddr*)&seraddr, sizeof(seraddr)) == -1)
	{
		perror("bind error");
		exit(1);
	}
	return sockfd;
}

void recv_print(int sockfd, int num)
{
	int n;
	char buff[BUFFSIZE];
	struct sockaddr clientaddr;
	socklen_t addrlen = sizeof(clientaddr);
	for(;;)
	{
		memset(buff, 0, BUFFSIZE);
		n = recvfrom(sockfd, buff, BUFFSIZE, 0, &clientaddr, &addrlen);
		printf("process %d\n", num);
		n = sendto(sockfd, buff, n, 0, &clientaddr, addrlen);
	}
}


void process_func(int sockfd, int num)
{
	if (sockfd > 0)
		recv_print(sockfd, num);
}



pid_t new_process(int sockfd, int num)
{
	pid_t pid = fork();
	if (pid > 0)
		return pid;
	else if (pid == 0)
		process_func(sockfd, num);
	else
		exit(1);
	return pid;
}

void start(char* str_addr)
{
	int i;
	for (i = 0; i < NUM_PROCESSES; i++)
	{
		int sockfd = new_sock(str_addr);
		new_process(sockfd, i);
	}
}

int main(int argc, char* argv[])
{
	if (argc == 2)
	{
		char* str_addr = argv[1];
		start(argv[1]);
	}
	return 0;
}
```

