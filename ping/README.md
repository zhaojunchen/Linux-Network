# Ping程序设计

[toc]
[github: https://github.com/zhaojunchen/Linux-Network/tree/master/ping ](https://github.com/zhaojunchen/Linux-Network/tree/master/ping)

## 0. 设计框架

ping程序一般按照下图的框架进行设计。主要分为

1. 发送数据
2. 接收数据
3. 计算时间差

发送数据对组织好的数据进行发送，接收数据从网络上接收数据并判断其合法性，例如判断是否本进程发出的报文等，计算时间差反应网络的延时。

<img src="https://zhaojunchen-website-1259455842.cos.ap-shanghai.myqcloud.com/markdown/20200519225647.png" alt="image-20200519210917147" style="zoom:50%;" />



## 1. 实验须知

### ICMP协议

ICMP协议消息类型有很多种、常见的有如下的几种：

| ICMP消息类型 | 用途说明                                                     |
| ------------ | ------------------------------------------------------------ |
| 回显请求     | Ping工具通过发送ICMP回显消息检查特定节点的IPv4连接以排查网络问题。类型值为0 |
| 回显应答     | 节点发送回显答复消息响应ICMP回显消息。类型值为8              |
| 重定向       | 路由器发送“重定向”消息，告诉发送主机到目标IPv4地址更好的路由。类型值为5 |
| 源抑制       | 路由器发送“源结束”消息，告诉发送主机它们的IPv4数据报将被丢弃——因为路由器上发生了拥塞。于是，发送主机将以较低的频度发送数据报。类型值为4 |
| 超时         | 这个消息有两种用途。第一，当超过IP生存期时向发送系统发出错误信息。第二，如果分段的IP数据报没有在某种期限内重新组合，这个消息将通知发送系统。类型值为11 |
| 无法到达目标 | 路由器和目标主机发送“无法到达目标”消息，通知发送主机它们的数据无法传送。类型值为3 |

其中ping发送数据的时候类型为 **回显请求**  接收数据的时候类型为 **回显应答**



## 2. 发送数据

发送数据对组织好的数据进行发送、送达到目标主机。

具体包含以下的几个部分：

### 1、 ICMP报文填充

ICMP报头的格式设置如下所示:

![image-20200519211520398](README.assets/aHR0cHM6Ly96aGFvanVuY2hlbi13ZWJzaXRlLTEyNTk0NTU4NDIuY29zLmFwLXNoYW5naGFpLm15cWNsb3VkLmNvbS9tYXJrZG93bi8yMDIwMDUxOTIyNDk1OC5wbmc.jpg)

**各字段说明**

- 类型：占一字节，标识ICMP报文的类型，目前已定义了14种，从类型值来看ICMP报文可以分为两大类。第一类是取值为1~127的差错报文，第2类是取值128以上的信息报文。
- 代码：占一字节，标识对应ICMP报文的代码。它与类型字段一起共同标识了ICMP报文的详细类型。
- **校验和**：这是对**包括ICMP报文数据部分在内的整个ICMP数据报的校验和（数据部分和报头部分）**，以检验报文在传输过程中是否出现了差错。其计算方法与在我们介绍IP报头中的校验和计算方法是一样的。
- 标识：占两字节，用于标识本ICMP进程，但仅适用于回显请求和应答ICMP报文，对于目标不可达ICMP报文和超时ICMP报文等，该字段的值为0。



ping程序请求远程主机所使用的ICMP的类型为**回显请求**

**回显请求**的报文格式如下：

![image-20200519212611266](README.assets/aHR0cHM6Ly96aGFvanVuY2hlbi13ZWJzaXRlLTEyNTk0NTU4NDIuY29zLmFwLXNoYW5naGFpLm15cWNsb3VkLmNvbS9tYXJrZG93bi8yMDIwMDUxOTIyNTAwMi5wbmc.jpg)

就是上面所说的**头部（没有选型字段）+ 数据部分**

ICMP回显请求报文的数据填充就是根据上图的格式填写正确的内容

**报文填充如下：**

**报头部分**

- ICMP回显请求的类型为8，即 ICMP ECHO。
- ICMP回显请求的代码值为0。
- ICMP回显请求的校验和后面会讲到
- ICMP回显请求的序列号是一个16位的值，通常由一个递增的值生成。
- ICMP回显请求的ID用于区别，通常用进程的PID填充进行ICMP头部校验
- ICMP回显请求头部的选项部分未设置

**数据部分**

- ICMP回显的数据部分可以任意设置，但是以太网包的总长度不能小于以太网的最小值，即总长度不能小于46.由于IP头部为20字节，ICMP头部为8个字节，以太网头部占用14个字节，因此ICMP回显包的最小值为46-20-8-14=4个字节。   **本程序中ICMP的报文长度设置为64（8字节头部+其余56字节的 任意数据）**





```
/**
 **设置ICMP报文
 * @param icmph 发送的报文
 * @param seq   序列号（依次递增）
 * @param tv    时间戳信息
 * @param length icmp报文(头部+数据部分)总长度  程序中设置为64  
 */
static void icmp_pack(struct icmp *icmph, int seq, struct timeval *tv, int length) {
    unsigned char i = 0;
    /*设置ICMP的报头*/
    // 请求类型
    icmph->icmp_type = ICMP_ECHO;    /*ICMP回显请求*/
    // 请求码
    icmph->icmp_code = 0;            /*code值为0*/
    // 校验和 
    icmph->icmp_cksum = 0;      /*先将cksum值填写0，便于之后的cksum计算*/
    // 序列号
    icmph->icmp_seq = seq;            /*本报的序列号*/
    icmph->icmp_id = pid & 0xffff;    /*填写PID*/
    
    // 设置ICMP的数据部分
    // length-8 = 56字节的数据部分填充
    for (i = 0; i < length - 8; i++)
        icmph->icmp_data[i] = i;
    /*计算校验和*/
    icmph->icmp_cksum = icmp_cksum((unsigned char *) icmph, length);
}
```



### 2、 ICMP校验

由于ICMP必须使用原始套接字进行设计，要手动设置ICMP的头部校验和。这是对**包括ICMP报文数据部分在内的整个ICMP数据报的校验和（数据部分和报头部分）**，以检验报文在传输过程中是否出现了差错。

```
// 使用校验的部分：（icmp报文填充的时候）
/*计算校验和*/
    icmph->icmp_cksum = icmp_cksum((unsigned char *) icmph, length);


/**
CRC16校验和计算icmp_cksum
参数：
	data:数据  注意的是ICMP校验包含了头部和数据部分  程序的ICMP总长度为64字节
	len:数据长度
返回值：
	计算结果，short类型
*/
static unsigned short icmp_cksum(unsigned char *data, int len) {
    int sum = 0;                            /*计算结果*/
    int odd = len & 0x01;                    /*是否为奇数*/
    /*将数据按照2字节为单位累加起来*/
    while (len & 0xfffe) {
        sum += *(unsigned short *) data;
        data += 2;
        len -= 2;
    }
    /*判断是否为奇数个数据，若ICMP报头为奇数个字节，会剩下最后一字节*/
    if (odd) {
        unsigned short tmp = ((*data) << 8) & 0xff00;
        sum += tmp;
    }
    sum = (sum >> 16) + (sum & 0xffff);    /*高低位相加*/
    sum += (sum >> 16);                    /*将溢出位加入*/

    return ~sum;                            /*返回取反值*/
}
```



### 3、 发送ICMP报文

发送报文函数是一个线程，每隔1s向目的主机发送一个ICMP回显请求报文，它在整个程序处于激活状态（ alive为1）时一直发送报文（alive是一个全局变量）

发送报文的逻辑如下：

1. 获得当前的时间值，按照序列号 packet_send将ICMP报文打包到缓冲区 send buff中后，发送到目的地址。发送成功后，记录发送报文的状态
   - 序号seq为 packet_send
   - 标志fag为1，表示已经发送但是没有收到响应
   - 发送时间为之前获得的时间
2. 每次发送成功后序号值会增加1，即 packet send++。
3. 在线程开始进入主循环 while（ alive）之前，将整个程序的开始发送时间记录下来，用于在程序退出的时候进行全局统计，即 gettimeofday（& ctv begin,NULL），将时间保存在变量 tv_begin中。

```
/*发送ICMP回显请求包*/
static void *icmp_send(void *argv) {
    /*保存程序开始发送数据的时间*/
    gettimeofday(&tv_begin, NULL);
    while (alive) {
        int size = 0;
        struct timeval tv;
        gettimeofday(&tv, NULL);            /*当前包的发送时间*/
        /*在发送包状态数组中找一个空闲位置*/
        
        // 寻找一个pingm_pakcet结构、记录当前ping包的时间戳信息和序列号等信息
        pingm_pakcet *packet = icmp_findpacket(-1);
        if (packet) {
        	// 设置序列号
            packet->seq = packet_send;        /*设置seq*/
            packet->flag = 1;                /*已经使用*/
            // 记录当前ping包的开始时间  结束时间在接收部分处理
            gettimeofday(&packet->tv_begin, NULL);    /*发送时间*/
        }
        // 填充报文数据到发送缓冲区  icmp_pack填充 ICMP响应请求的报头和数据部分
        icmp_pack((struct icmp *) send_buff, packet_send, &tv, 64);
        
        // 原始套接字 发送 到目标主机
        // rawsock：ICMP的原始套接字
        // send_buff ICMP报文 
        // 64 是报文的长度 dest是服务器地址
        size = sendto(rawsock, send_buff, 64, 0,        /*发送给目的地址*/
                      (struct sockaddr *) &dest, sizeof(dest));
        if (size < 0) {
            perror("sendto error");
            continue;
        }
        // packet_send就是发送的序列号
        packet_send++;                    /*计数增加*/
        /*每隔1s，发送一个ICMP回显请求包*/
        sleep(1);
    }
}
```

**备注** 

1. 在C语言中可以使用函数gettimeofday()函数来得到精确时间。它的精度可以达到微妙，是C标准库的函数. gettimeofday()会把目前的时间用timeval 结构体返回，当地时区的信息则放到tz所指的结构中

2. timeval 结构体如下

   ```
   struct timeval
   {
     __time_t tv_sec;    /* Seconds.  */
     __suseconds_t tv_usec;   /* Microseconds.  */
   };
   #endif
   ```



## 3. 接收数据

### 1、 剥离ICMP接收报文头部

**函数 icmp_unpack处理来自目标地址的响应应答** icmp_unpack会剥离应答内容IP头部，分析ICMP头部的值。判断是否为正确的ICMP报文，并打印结果。

参数buf为剥去了以太网部分数据的IP数据报文，len为数据长度。可以利用IP头部的参数快速地跳到ICMP报文部分，IP结构的iph标识I头部的长度，由于ih标识的是4字节单位，所以需要乘以4来获得IMP段的地址。

获得ICMP数据段后，判断其类型是否为 ICMP ECHOREPLY，并核实其标识是否为本进程的PID。由于需要判断数据报文的往返时间，在本程序中需要先查找这个包发送时的时间，与当前时间进行计算后，可以得出本地主机与目标主机之间网络ICMP回显报文的差值！



```
/*解压接收到的包，并打印信息*/
static int icmp_unpack(char *buf, int len) {
    int iphdrlen;
    struct ip *ip = NULL;
    struct icmp *icmp = NULL;
    int rtt;

    ip = (struct ip *) buf;                    /*IP头部*/
    iphdrlen = ip->ip_hl * 4;                    /*IP头部长度*/
    // icmp指向ICMP报文的起始地址
    icmp = (struct icmp *) (buf + iphdrlen);        /*ICMP段的地址*/
    len -= iphdrlen;
    // len是ICMP报文的长度
    /*判断长度是否为ICMP包*/
    if (len < 8) {
        printf("ICMP packets\'s length is less than 8\n");
        return -1;
    }
    /*ICMP类型为ICMP_ECHOREPLY并且为本进程的PID*/
    if ((icmp->icmp_type == ICMP_ECHOREPLY) && (icmp->icmp_id == pid)) {
        struct timeval tv_internel, tv_recv, tv_send;
        /*在发送表格中查找已经发送的包，按照seq*/
        // 已经使用的ping包（flag =1 存在seq）会被seq寻找到
        pingm_pakcet *packet = icmp_findpacket(icmp->icmp_seq);
        if (packet == NULL)
            return -1;
        packet->flag = 0;    /*取消标志*/
        tv_send = packet->tv_begin;            /*获取本包的发送时间*/
        gettimeofday(&tv_recv, NULL);        /*读取此时间，计算时间差*/
        // timeval 时间差计算
        tv_internel = icmp_tvsub(tv_recv, tv_send);
        // timeval 得到rtt往返时间
        rtt = tv_internel.tv_sec * 1000 + tv_internel.tv_usec / 1000;
        /*打印结果，包含
        *  ICMP段长度
        *  源IP地址
        *  包的序列号
        *  TTL
        *  时间差
        */
        // 输出当前报文的信息
        printf("%d byte from %s: icmp_seq=%u ttl=%d rtt=%d ms\n",
               len,
               inet_ntoa(ip->ip_src),
               icmp->icmp_seq,
               ip->ip_ttl,
               rtt);
		// 统计受到响应的ping包
        packet_recv++;                        /*接收包数量加1*/
    } else {
        return -1;
    }
    return 0;
}

// 附上时间计算函数
static struct timeval icmp_tvsub(struct timeval end, struct timeval begin) {
    struct timeval tv;
    /*计算差值*/
    tv.tv_sec = end.tv_sec - begin.tv_sec;
    tv.tv_usec = end.tv_usec - begin.tv_usec;
    /*如果接收时间的usec值小于发送时的usec值，从usec域借位*/
    if (tv.tv_usec < 0) {
        tv.tv_sec--;
        tv.tv_usec += 1000000;
    }

    return tv;
}
```

### 2、 接收报文

与发送函数一样，接收报文也用一个线程实现，使用 select轮询等待报文到来。当接收到一个报文后，使用函数 icmp _npack（来解包和查找报文之前发送时的记录，获取发送时间，计算收发差值并打印信息)
（1）接收成功后将合法的报文记录重置为没有使用，fag为0
（2）接收报文数量增加1。
（3）为了防止丢包， select的轮询时间设置的比较短。

接收来自目标地址的响应数据、并且将其送入imcp_packet解包

```
/*接收ping目的主机的回复*/
static void *icmp_recv(void *argv) {
    /*轮询等待时间*/
    struct timeval tv;
    tv.tv_usec = 200;
    tv.tv_sec = 0;
    fd_set readfd;
    /*当没有信号出一直接收数据发*/
    while (alive) {
        int ret = 0;
        FD_ZERO(&readfd);
        FD_SET(rawsock, &readfd);
        ret = select(rawsock + 1, &readfd, NULL, NULL, &tv);
        switch (ret) {
            case -1:
                /*错误发生*/
                break;
            case 0:
                /*超时*/
                break;
            default: {
                /*接收数据*/
                int size = recv(rawsock, recv_buff, sizeof(recv_buff),
                                0);
                if (errno == EINTR) {
                    perror("recvfrom error");
                    continue;
                }
                /*解包，并设置相关变量*/
                ret = icmp_unpack(recv_buff, size);
                if (ret == -1) {
                    continue;
                }
            }
                break;
        }

    }
}
```



## 4. 主程序流程

ping程序的实现使用了两个线程，一个线程 icmp sendo用于发送请求，另一个线程icmp recv用于接收远程主机的响应。当变量 alive为0时，两个线程退出。

### Ping数据结构

程序使用类型为结构 struct ping_packet的变量 pingpacket用于保存发送数据报文的状态。

- tv_begin用于保存发送的时间。
- ty_end用于保存数据报文接收到的时间。
- seq是序列号，用于标识报文，作为索引。
- flag用于表示本单元的状态，表示数据报文已经发送，但是没有收到回应包；0表示已经接收到回应报文，这个单元可以再次用于标识发送的报文。

```
typedef struct pingm_pakcet {
    struct timeval tv_begin;    /*发送的时间*/
    struct timeval tv_end;        /*接收到的时间*/
    short seq;                    /*序列号*/
    int flag;        /*1，表示已经发送但没有接收到回应包0，表示接收到回应包*/
} pingm_pakcet;

```





### 主函数流程

0.ICMP协议类型设置

```
 struct protoent *protocol = NULL;
 char protoname[] = "icmp";
 /*获取协议类型ICMP*/
 protocol = getprotobyname(protoname);
```

1.初始化ICMP的raw sock

```
 /*socket初始化*/
    rawsock = socket(AF_INET, SOCK_RAW, protocol->p_proto);
```

2.系列全局变量初始化（不多赘述）

```
/*复制目的地址字符串*/
    memcpy(dest_str, argv[1], strlen(argv[1]) + 1);
    memset(pingpacket, 0, sizeof(pingm_pakcet) * 128);
    /*为了与其他进程的ping程序区别，加入pid*/
    pid = getuid();
    /*增大接收缓冲区，防止接收的包被覆盖*/
    setsockopt(rawsock, SOL_SOCKET, SO_RCVBUF, &size, sizeof(size));
    bzero(&dest, sizeof(dest));
```

3.目的IP地址解析（域名或者IP）

```
	dest.sin_family = AF_INET;
    /*输入的目的地址为字符串IP地址*/
    inaddr = inet_addr(argv[1]);
    if (inaddr == INADDR_NONE) {
        /*输入的是DNS地址*/
        host = gethostbyname(argv[1]);
        if (host == NULL) {
            perror("gethostbyname");
            return -1;
        }
        /*将地址复制到dest中*/
        memcpy((char *) &dest.sin_addr, host->h_addr, host->h_length);
    } else        /*为IP地址字符串*/
    {
        memcpy((char *) &dest.sin_addr, &inaddr, sizeof(inaddr));
    }
    /*打印提示*/
    inaddr = dest.sin_addr.s_addr;
```

4.开启发送线程和接收线程

```
pthread_t send_id, recv_id;        /*建立两个线程，用于发送和接收*/
int err = 0;
err = pthread_create(&send_id, NULL, icmp_send, NULL);        /*发送*/
if (err < 0) {
    return -1;
}
err = pthread_create(&recv_id, NULL, icmp_recv, NULL);        /*接收*/
if (err < 0) {
    return -1;
}
/*等待线程结束*/
    pthread_join(send_id, NULL);
    pthread_join(recv_id, NULL);
```

5.清理并打印统计结果

```
/*清理并打印统计结果  程序使用SIGINT退出子线程、控制权转到主线程*/
close(rawsock);
icmp_statistics();
```

### 注意点

在主程序中需要注意如下几点：

- 使用 getprotobyname函数获得icmp对应的ICMP协议值。对输入的目的主机地址兼容域名和IP地址，使用 gethostbynameo函数来获得DNS对应的IP地址， inet addo函数获得字符串类型的P地址对应的整型值。
- 为了防止远程主机发送过大的包或者本地来不及接收现象的发生， setsockopt（ rawsock、 SOL SOCKET、 SO RCVBUF、&size、 sizeof（size）函数将 socket的接收缓冲区设置为128K
- 对信号 SIGINT进行了截取、设置alive为0 让子线程主动退出，用以来结束函数
- 建立两个线程 icmp_sendo和 icmp_recv进行接收和发送，然后主程序等待两个线程结束。
- ICMP原始报文发送需要root权限



### 运行结果

![image-20200519224549793](README.assets/aHR0cHM6Ly96aGFvanVuY2hlbi13ZWJzaXRlLTEyNTk0NTU4NDIuY29zLmFwLXNoYW5naGFpLm15cWNsb3VkLmNvbS9tYXJrZG93bi8yMDIwMDUxOTIyNTAwOS5wbmc-1589903769198.jpg)

## 参考文献

\[ 1 ] : Linux网络程序设计