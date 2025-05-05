# dhcpsnooping-lite
在openwrt基于ebtables实现轻量级的dhcpsnooping

## 先决条件
### openwrt 要么基于dsa的交换机 
判断openwrt是否使用 dsa 
grep -s DEVTYPE=dsa /sys/class/net/*/uevent

### 对于非dsa的设备
解除默认的vlan配置,然后为每个网口分配独立vlan 再然后将这些独立网卡桥接到网桥 br-lan 
否则ebtables无法起作用  
路由器一共5个网口 包含 wan lan 1-4  这是一台作为交换机使用的openwrt路由器 将 5个网口全部用于交换机接口  
这种配置对cpu几乎没有性能损失 
 
/etc/config/network 配置如下  



config device  
	option name 'br-lan'  
	option type 'bridge'  
	option stp '1'  
	option igmp_snooping '1'  
	list ports 'eth0.11'  
	list ports 'eth0.22'  
	list ports 'eth0.33'  
	list ports 'eth0.44'  
	list ports 'eth0.55'  
  
config switch  
	option name 'switch0'  
	option reset '1'  
	option enable_vlan '1'  
  
config switch_vlan  
	option device 'switch0'  
	option vlan '1'  
	option vid '11'  
	option ports '0t 1'  

config switch_vlan  
	option device 'switch0'  
	option vlan '2'  
	option vid '22'  
	option ports '0t 2'  
  
config switch_vlan  
	option device 'switch0'   
	option vlan '3'  
	option vid '33'  
	option ports '0t 3'  
  
config switch_vlan  
	option device 'switch0'  
	option vlan '4'  
	option vid '44'  
	option ports '0t 4'  
  
config switch_vlan  
	option device 'switch0'  
	option vlan '5'  
	option vid '55'  
	option ports '0t 5'  

## 在openwrt 安装 ebtables
opkg install ebtables ebtables-utils kmod-ebtables kmod-ebtables-ipv4 kmod-ebtables-ipv6 

 
## 使用ebtables 抑制 非信任dhcp服务端的报文
 实现原理 通过mac地址匹配 使记录的dhcp服务器报文通过 
 没有报文结构解析  当dhcpd 不使用67作为源端口时 规则不起作用
 不是基于 switch port ,而是网桥 如 openwrt的 br-lan
 当局域网内存在多个dhcpd服务器时 仅允许那些被记录到ebtables规则的mac地址通过 没有记录的被DROP
 
 ebtables -t filter -D OUTPUT --logical-out br-lan  -p ipv4 --ip-protocol udp  --ip-source-port  67 --among-src ! 11:22:33:44:55:66 -j DHCPSNOOPING_IN
 
 ebtables -t filter -A DHCPSNOOPING_IN --among-src "$DHCPDMAC" -j RETURN
 
 ebtables -t filter -A DHCPSNOOPING_IN -j DROP
 

## 脚本 /etc/init.d/dhcpsnooping-lite 实现了这个操作

 在 start 函数 添加规则  
 on 网卡 你的mac地址  
 如 on br-lan 11:22:33:44:55:66  

在stop 函数 删除规则  
off br-lan  

 


 
 

 
