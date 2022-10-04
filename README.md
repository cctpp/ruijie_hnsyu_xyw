# ruijie_syu_xyw

HNSYU（适用于OpenWrt路由的Shell脚本）实现用路由器连接校园网的项目，实现多设备上网。  
原来的校园网支持PPPOE拨号，检测网络共享也不算很严，随便接个由器接上就能用。今年校园网进行全光改造，认证方式也改成网页认证。  
然后关于防检测的可以参考 [这里](https://www.sunbk201.site/posts/crack-campus-network.html)本文不再赘述
### 需要准备的
1. 一个能刷OpenWrt的路由器
2. 带有有线网口的PC一台
3. 有线的校园网账号一份(无线套餐的，你都办无线了肯定也不需要共享设备了)
4. 2条千兆网线（一条用于调试）
5. [WinSCP](https://winscp.net/eng/docs/lang:chs)软件；用于上传脚本
6. [putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/)软件
## 开始
1. 编译带有```UA2F```插件的对应路由器型号的固件。一般使用[云编译](https://github.com/cctpp/Actions-lede-UA2F)(本地编译报错太多)
2. 刷入OpenWrt固件,需要用到```breed Web```(一般在网上买的都已经刷好了)按住RESET键再插电，路由器灯闪烁再松开按键然后用电脑网线接路由器lan口接着浏览器访问 ```192.168.1.1```刷入即可。刷好后路由器后台地址默认为```192.168.1.1``` 用户名 ```root``` 密码 ```password```
   ![breed web](https://s1.ax1x.com/2022/10/04/xlpfsS.png)
   ![后台管理地址](https://s1.ax1x.com/2022/10/04/xlp7in.png)
3. 配置校园网登录脚本
4. 配置防检测
### 配置校园网登录脚本
1. 电脑用网线连接锐捷的AP面板，随便打开网页会自动跳转校园网登录页面，输入账号密码勾选记住密码登录成功后，下线，回到登录页面，这时候就需要复制网址```.jsp?```后的所有内容。记住电脑的mac地址后面有用。
2. 下载```xyw.sh```并编辑第27行
   ```
   queryString="wlanuserip="
   ```
   把双引号里的内容替换成刚才复制的内容并且保存。拔掉PC与锐捷面板的连接改为 用PC连接路由器```LAN```，用路由器```WAN```连接锐捷面板。
3. 把改好的```xyw.sh```用```WinSCP```上传到路由器的```/root/ ```目录  
   ![winscp登录](https://s1.ax1x.com/2022/10/04/xlpG5R.png)
   ![上传.sh](https://s1.ax1x.com/2022/10/04/xlpwrD.png)
   
   打开目录```/etc/config/```打开文件```network```编辑```config device 'wan_eth0_2_dev'```和```config interface 'wan'```下```option macaddr 'F8:75:A4:3D:69:25'```把``里的内容换成刚才电脑的mac编辑并保存  
   ![修改mac1](https://s1.ax1x.com/2022/10/04/xl9MWt.png)
   ![修改mac2](https://s1.ax1x.com/2022/10/04/xl91Qf.png)
   ![电脑mac](https://s1.ax1x.com/2022/10/04/xl9YwQ.png)  
注意电脑mac地址用```-```连接  
需要用```:```替换。   
重启路由器后进入后台管理页面点开```网络-->接口``` 看到mac地址已经和电脑是一样就算成功了。
1. 打开```putty```登录到```192.168.1.1```账户名```root``` 密码```password```执行命令  
![登录ssh](https://s1.ax1x.com/2022/10/04/xlCC7Q.png)
   ```
   sh xyw.sh 校园网账号 密码
   ```
 
回车，会返回一个提示。成功后，就能通过路由器联网了。  

![成功界面](https://s1.ax1x.com/2022/10/04/xl970e.png)
   
1. 配置自动联网脚本```系统-->计划任务```添加代码
   ```
   */5 * * * * sh xyw.sh 校园网账号 密码
   ```
   提交
   保存配置后路由器就每5分钟运行一次脚本。
2. 配置无线```网络-->>无线```  
   自己配置，总不会连wifi名称和密码都不会自己设置吧
3. 添加开机启动无线网络```系统-->启动项```拉到最下面添加代码
   ```
   ifconfig rai0 up
   ifconfig ra0 up
   brctl addif br-lan rai0
   brctl addif br-lan ra0
   ```
   提交
### 配置防检测
1. 访问路由器后台，进入管理页面鼠标移动上方选项卡```网络```-->``` 防火墙```-->```自定义规则```让后把下方代码复制粘贴上去点击```重启防火墙```
```
iptables -t nat -A PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 53
iptables -t nat -A PREROUTING -p tcp --dport 53 -j REDIRECT --to-ports 53
uci set ua2f.enabled.enabled=1
uci commit ua2f
# 通过 rkp-ipid 设置 IPID
# 若没有加入rkp-ipid模块，此部分不需要加入
iptables -t mangle -N IPID_MOD
iptables -t mangle -A FORWARD -j IPID_MOD
iptables -t mangle -A OUTPUT -j IPID_MOD
iptables -t mangle -A IPID_MOD -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A IPID_MOD -d 127.0.0.0/8 -j RETURN
#由于本校局域网是A类网，所以我将这一条注释掉了，具体要不要注释结合你所在的校园网
# iptables -t mangle -A IPID_MOD -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A IPID_MOD -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A IPID_MOD -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A IPID_MOD -d 255.0.0.0/8 -j RETURN
iptables -t mangle -A IPID_MOD -j MARK --set-xmark 0x10/0x10

# 防时钟偏移检测
iptables -t nat -N ntp_force_local
iptables -t nat -I PREROUTING -p udp --dport 123 -j ntp_force_local
iptables -t nat -A ntp_force_local -d 0.0.0.0/8 -j RETURN
iptables -t nat -A ntp_force_local -d 127.0.0.0/8 -j RETURN
iptables -t nat -A ntp_force_local -d 192.168.0.0/16 -j RETURN
iptables -t nat -A ntp_force_local -s 192.168.0.0/16 -j DNAT --to-destination 192.168.1.1

# 通过 iptables 修改 TTL 值
iptables -t mangle -A POSTROUTING -j TTL --ttl-set 64
```
2. 进入```网络-->Turbo ACC 网络加速```关掉所有  
![网络-->Turbo ACC 网络加速](https://s1.ax1x.com/2022/10/04/xQzHTs.png)
3. 打开```putty```登录后运行命令
   ```
   service ua2f enable
   ```
4. 重启路由器后访问[http://ua.233996.xyz/](http://ua.233996.xyz/)如果出现下图就算成功了
   ![防止检测成功图](https://s1.ax1x.com/2022/10/04/xlCRHg.png)
## 结束
如有错误之处欢迎指出  
本文所有文件 均来自网络 ,感谢以下大佬的无私奉献。  
[锐捷校园网自动认证路由脚本](https://blog.csdn.net/u010102747/article/details/124639593)  
[关于某大学校园网共享上网检测机制的研究与解决方案](https://www.sunbk201.site/posts/crack-campus-network.html)  
[Zxilly/UA2F](https://github.com/Zxilly/UA2F)  
[一个云编译UA2F固件的项目](https://github.com/MoorCorPa/Actions-lede-UA2F)  


