# wx_revoke
微信消息防撤回


# 用python实现微信消息防撤回

这个脚本是基于 [itchat](https://github.com/littlecodersh/ItChat) 库来实现的。
> itchat是一个开源的微信个人号接口，使用python调用微信从未如此简单。
使用不到三十行的代码，你就可以完成一个能够处理所有信息的微信机器人。
当然，该api的使用远不止一个机器人，更多的功能等着你来发现，比如这些。
该接口与公众号接口itchatmp共享类似的操作方式，学习一次掌握两个工具。
如今微信已经成为了个人社交的很大一部分，希望这个项目能够帮助你扩展你的个人的微信号、方便自己的生活。

### 准备环境：
+ 安装itchat库： `pip install itchat`

### 代码实现：
```
   1 #!/usr/bin/env python3
   2 # -*- coding: utf-8 -*-
   3
   4 import os
   5 import re
   6 import shutil
   7 import time
   8 import itchat
   9 from itchat.content import *
  10
  11
  12 # 定义一个字典，保存消息的信息。
  13 msg_dict = {}
  14
  15 # 创建一个目录，用于存放消息临时文件。
  16 rec_tmp_dir = "/Users/matianhe/itchat/rec_tmp/"
  17 if not os.path.exists(rec_tmp_dir):
  18     os.mkdir(rec_tmp_dir)
  19
  20
  21 face_bug = None
  22
  23
  24 # 注册消息接收器
  25 @itchat.msg_register([TEXT, PICTURE, MAP, CARD, SHARING, RECORDING,
  26                       ATTACHMENT, VIDEO])
  27 def handler_receive_msg(msg):
  28     global face_bug
  29     msg_time_rec = time.strftime("%Y-%m-%d %H:%M%S", time.localtime())
  30     msg_id = msg['MsgId']
  31     msg_time = msg['CreateTime']
  32     msg_from = (itchat.search_friends(userName=msg['FromUserName']
  33                                       ))["NickName"]
  34     msg_content = None
  35     msg_share_url = None
  36
  37     if msg['Type'] == 'Text' or msg['Type'] == 'Friends':
  38         msg_content = msg['Text']
  39     elif msg['Type'] == 'Recording' or msg['Type'] == 'Attachment' \
  40             or msg['Type'] == 'Video' or msg['Type'] == 'Picture':
  41         msg_content = r"" + msg['FileName']
  42         msg['Text'](rec_tmp_dir + msg['FileName'])
  43     elif msg['Type'] == 'Card':
  44         msg_content = msg['RecommendInfo']['NickName'] + r" 的名片"
  45     elif msg['Type'] == 'Map':
  46         x, y, location = re.search("<location x=\"(.*?)\" y=\"(.*?\".*lable= \
  47                                    \"(.*?)\".*", msg['OriContent']).group(1, 2,
  48                                                                           3)
  49         if location is None:
  50             msg_content = r"纬度->" + x.__str__() + " 经度->" + y.__str__()
  51         else:
  52             msg_content = r"" + location
  53     elif msg['Type'] == 'Sharing':
  54         msg_content = msg['Text']
  55         msg_share_url = msg['Url']
  56     face_bug = msg_content
  57
  58     msg_dict.update({
  59         msg_id: {
  60             "msg_from": msg_from, "msg_time": msg_time,
  61             "msg_time_rec": msg_time_rec, "msg_type": msg["Type"],
  62             "msg_content": msg_content, "msg_share_url": msg_share_url
  63         }
  64     })
  65
  66
  67 @itchat.msg_register([NOTE])
  68 def send_msg_helper(msg):
  69     global face_bug
  70     if re.search(r"\<\!\[CDATA\[.*撤回了一条消息\]\]\>", msg['Content']) \
  71             is not None:
  72         old_msg_id = re.search("\<msgid\>(.*?)\<\/msgid\>", \
  73                                 msg['Content']).group(1)
  74         old_msg =msg_dict.get(old_msg_id, {})
  75         if len(old_msg_id) < 11:
  76             itchat.send_file(rec_tmp_dir + face_bug,
  77                                 toUserName='filehelper')
  78             os.remove(rev_tmp_dir + face_bug)
  79         else:
  80             msg_body = "有人撤回消息" + "\n" \
  81                 + old_msg.get('msg_from') + " 撤回了 " \
  82                 + old_msg.get('msg_type') + " 消息" + "\n" \
  83                 + old_msg.get('msg_time_rec') + "\n" \
  84                 + r"" + old_msg.get('msg_content')
  85             if old_msg['msg_type'] == "Sharing":
  86                 msg_body += "\n就是这个连接->" + old_msg.get('msg_share_url')
  87             itchat.send(msg_body, toUserName='filehelper')
  88             if old_msg["msg_type"] == "Picture" \
  89                     or old_msg["msg_type"] == "Recording" \
  90                     or old_msg["msg_type"] == "Video" \
  91                     or old_msg["msg_type"] == "Attachment":
  92                 file = '@fil@%s' % (rec_tmp_dir + old_msg['msg_content'])
  93                 itchat.send(msg=file, toUserName='filehelper')
  94                 os.remove(rec_tmp_dir + old_msg['msg_content'])
  95             msg_dict.pop(old_msg_id)
  96
  97
  98 if __name__ == "__main__":
  99     itchat.auto_login(hotReload=True, enableCmdQR=2)
 100     itchat.run()
```

### 实现思路：
注册两个消息接收器，一个接收平时消息，并把消息信息存到字典里。另一个监听是否有撤回操作，如果有撤回操作则取出第一个监听器内的消息，用文件传输助手发送到手机上。
