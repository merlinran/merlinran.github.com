---
layout: default
title: EMC Symmetrix产品的GateKeeper介绍
---
EMC Symmetrix磁盘阵列的管理有两个途径，一是通过Service Processor，主要由工程师做更换、检查、升级等维护操作，另一种方式就是主机上的各种工具，如symcli、Solution Enabler、EMC Control Center等等。而主机和阵列的通讯，只能通过数据访问的链路，如Fibre Channel或者iSCSI来完成，主机向LUN发起一些特殊的SCSI命令，就实现了管理的功能。

理论上说，主机识别到的任何一个LUN，都能做为管理的载体。但某些管理命令，可能需要较长时间才能完成，后续的数据IO只能排队，增大了延迟。作为高端阵列，管理监控的指标很多，为了避免对性能的影响，通常单独建很小的LUN，管理功能只通过它们进行，和数据访问完全独立。这些很小的LUN，就称作GateKeeper。 

  + GateKeeper LUN的大小通常是2880KB，DMX2和以前是6个柱面，DMX3和以后是3个柱面（注1）

  + 小于8MB的LUN，会被自动认为是GateKeeper，在syminq的输出中，会把LUN的类型标识为GK

  + 如果没有配置专用GateKeeper，会强制使用数据LUN（影响性能）。到底选择哪些LUN，可以在安装Solution Enabler时用gkavoid和gkselect这两个文件来干预。

  + 每个GateKeeper只能给一台主机使用

  + 如果没有配置Consistency Group，6个GateKeeper足够了。

  + 建议把GateKeeper用PowerPath管理起来，这样一条路径有问题了，仍然可以通过另外的路径去访问。但其它的多路径软件不支持那些特别的SCSI命令，所以不能用它们来管理。

更多信息，请在EMC PowerLink上查找Primus emc255976

*注一：LUN大小＝柱面数 * 每柱面磁道数(15) * 每磁道扇区数(DMX2和以前是64，之后是128)  * 512 (每扇区字节数)*

