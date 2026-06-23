# 红温的demo

1. 去区块链浏览器（<https://www.oklink.com/zh-hans/sepolia>）里查合约<code><font style="color:rgb(0, 0, 0);">0x032C7E5Bba57BF5947129436eE3F8Ad8966938A9</font></code>事件

![1760618558406-5ee850ec-e875-47fc-8a80-40170f87c0a1.png](./img/eoJaPIKZZHQSTi4Q/1760618558406-5ee850ec-e875-47fc-8a80-40170f87c0a1-337669.png)

2. 去事件栏，查看输出的网址h，这个要选文本格式，可以读到<http://154.221.31.223:9000/>
3. 查看输出的地址key，得到一串地址，把多余的0都去掉，即`0xa0Ee7A142d267C1f36714E4a8F75612F20a79720`

![1760618549065-0d847b04-49ad-4215-9d48-e25b10498f2c.png](./img/eoJaPIKZZHQSTi4Q/1760618549065-0d847b04-49ad-4215-9d48-e25b10498f2c-569252.png)

4. 进入h，把去掉多余0的key贴到demo栏4的to里，点check，demo修复成功变成绿色，nonce栏里弹出flag：DLNUFCG{Th1s\_Is\_7he\_F1ag}

![1758974677225-6f44bdd0-7c2c-423b-8a31-9d7c09692b0a.png](./img/eoJaPIKZZHQSTi4Q/1758974677225-6f44bdd0-7c2c-423b-8a31-9d7c09692b0a-359570.png)![1758974677267-47601fc3-bc6f-4eb2-90bc-3cdf4c36dda6.png](./img/eoJaPIKZZHQSTi4Q/1758974677267-47601fc3-bc6f-4eb2-90bc-3cdf4c36dda6-486795.png)![1758974677226-e5456554-dc2d-49ec-8c2b-cfdae31396ae.png](./img/eoJaPIKZZHQSTi4Q/1758974677226-e5456554-dc2d-49ec-8c2b-cfdae31396ae-570621.png)

```solidity
==================================================================
Congratulations! Installed successfully!
=============注意：首次打开面板浏览器将提示不安全=================

 请选择以下其中一种方式解决不安全提醒
 1、下载证书，地址：https://dg2.bt.cn/ssl/baota_root.pfx，双击安装,密码【www.bt.cn】
 2、点击【高级】-【继续访问】或【接受风险并继续】访问
 教程：https://www.bt.cn/bbs/thread-117246-1-1.html
 mac用户请下载使用此证书：https://dg2.bt.cn/ssl/mac.crt

========================面板账户登录信息==========================

 【云服务器】请在安全组放行 37757 端口
 外网ipv4面板地址: https://154.221.24.170:37757/35169840
 内网面板地址:     https://154.221.24.170:37757/35169840
 username: co4yfiwo
 password: a0e805d7

 浏览器访问以下链接，添加宝塔客服
 https://www.bt.cn/new/wechat_customer
==================================================================
Time consumed: 6 Minute!
root@yisu-689167f03c23a:~#
```

[https://154.221.31.223:37757/](https://154.221.31.223:37757/files)35169840


> 更新: 2025-10-16 20:42:39  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/dg3zgz59p0og483a>