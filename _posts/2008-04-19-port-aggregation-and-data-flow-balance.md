---
layout: post
title: 端口聚合与流量分担
---
上班正在赶着写存储过程，做报表，以前铁通的同事突然在QQ上找我，说她那里有一台Cisco 2970交换机，上行连Cisco 4507，用了两个端口做聚合，开始没注意，现在流量满了，才发现端口聚合没生效，上行流量一直只走一个端口。

以前没搞过这个东东，但研究的劲头来了。我让她把两边交换机的配置发给我看，2970这边的相关部分如下：

    interface GigabitEthernet0/23
     description To-IDC4507R
     switchport trunk encapsulation dot1q
     switchport mode trunk
     speed 1000
     duplex full
     channel-group 1 mode active
    !
    interface GigabitEthernet0/24
     description To-IDC4507R
     switchport trunk encapsulation dot1q
     switchport mode trunk
     speed 1000
     duplex full
     channel-group 1 mode active
    !

本来配端口聚合挺简单的。这个配置也看不出啥问题。第一个想法就是比较两边的配置，但4507那边的配置她一直没发给我，也就没法比较。

于是一直沿着这条唯一的配置命令往下找：

    channel-group 1 mode active

心想是不是两边的模式配错了呢？让她检查，也没问题。

后来问她，这个交换机是不是接各个VOD服务器？得到肯定回答后，一下子明白了。

原来Cisco交换机做端口聚合时，默认的负载均衡方式是按源MAC地址，VOD服务器只有几台，也就意味着源MAC地址只有几个，负载肯定难以做到均衡了。让她把负载均衡方式改为按目的IP地址，这样，所有在线看电影的宽带用户的IP地址数以万计，轻易就能达到完美的均衡。相关命令如下：

    port-channel load-balance {dst-ip | dst-mac | src-dst-ip | src-dst-mac | src-ip | src-mac}

改完后，GE0/24的流量马上增加了。她这才回头去看4507的配置，告诉我说那边配的是：

    IDC#show etherchannel load-balance 
    Source XOR Destination IP address

这种方式，要多做一次内存读取，加上一次异或操作，会轻微地影响转发性能，我让她改为按src-ip，同样把数以万计的铁通宽带IP地址拿来做均衡。
