---
layout: default
title: 把魅族M8备份的XML格式通讯录恢复到酷派8150手机
---
魅族M8的屏幕坏了，赶紧跑移动办了个酷派的8150，但通讯录里边几百个联系人，没法导。只有之前备份出来的Contact.xml文件，而8150只支持vCard导入。

本打算用Ruby写个小转换程序，后来发现这位同学用Python写过一个，直接借来改了改，主要是把vCard加工成酷派吃得下的东西，另外把公司名和分组信息也导进去。好吧，这是我第二个有用的Python程序，可能也适用于酷派的其它Android型号。

使用方法：

酷派手机上有个“酷派备份”的工具，备份时只选通讯录。

USB连接到电脑，假设电脑识别出来是F盘，在F:\COOLPAD\BACKUPSD\下面就是生成的备份文件

打开contacts.zip，里面有contacts.vcf，用我们的程序生成的vcf文件替换它

contacts.zip里面有个groups.vcf文件，打开是这个样子

    BEGIN:VCARD  
    VERSION:2.1  
    X_GROUPINFO;CHARSET=UTF-8:朋友;1;friend;;;;  
    X_GROUPINFO;CHARSET=UTF-8:家人;1;family;;;;  
    X_GROUPINFO;CHARSET=UTF-8:同事;1;confrere;;;;  
    X_GROUPINFO;CHARSET=UTF-8:同学;1;schoolmate;;;;  
    END:VCARD  

把M8中用到的组全部加进这个文件。一定要按照相同的格式。注意后面的英文名称，也要写上

8150从电脑上断开，选择备份文件，点击恢复

下面是对应的代码

    #!/usr/bin/env python  
    # coding: utf-8  
      
    from xml.etree import ElementTree as ET  
      
    out = file("contacts.vcf", "wb")  
    root = ET.parse(file("Contact.xml", "r")).getroot()  
      
    for e in root.findall('Contact'):  
      
      out.write('BEGIN:VCARD\r\nVERSION:2.1\r\n')  
      fn = e.findtext('FileAs', '').encode('utf8').encode('quopri')  
      out.write('N;CHARSET=UTF-8;ENCODING=QUOTED-PRINTABLE:;%s;;;\r\n' % fn)  
      out.write('FN;CHARSET=UTF-8;ENCODING=QUOTED-PRINTABLE:%s\r\n' % fn)  
      
      category = e.findtext('Category')  
      if category is not None:  
        out.write('X_GROUP;CHARSET=UTF-8;ENCODING=QUOTED-PRINTABLE:%s\r\n' % category[0:-1].encode('utf8').encode('quopri'))  
      
      company = e.findtext('Company')  
      if company is not None:  
        out.write('ORG;CHARSET=UTF-8;ENCODING=QUOTED-PRINTABLE:%s\r\n' % company.encode('utf8').encode('quopri'))  
        out.write('X_CPOTR;CHARSET=UTF-8:false;;;0;0;;\r\n')  
      else:  
        out.write('X_CPOTR;CHARSET=UTF-8:0;;;0;;;\r\n')  
      
      for ee in e.find('Phone').findall('PhoneElement'):  
        primary = int(ee.get('IsPrimary'))  
        phone_type = int(ee.get('Type'))  
        out.write('TEL%s%s:%s\r\n' %  
          ( (';WORK' if phone_type == 1 else ';CELL'), (';PREF' if primary else ''), ee.get('Value')))  
      
      out.write('X_SECRET_COOLPAD:0\r\n')  
      out.write('END:VCARD\r\n')  
      
    out.close()  

