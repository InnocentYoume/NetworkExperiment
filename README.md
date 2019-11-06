 # 网络综合实验
 ## 实验设备
* 模拟器 : HCL
* 路由器 : MSR36-20
* 交换机 : S5820V2-54QS-GE
* 操作系统 : ComwareV7.0
---
## 交换机
---
### TELNET方式登录交换机
使用如下命令:
```
[SW]int vlan {vlanid}
[SW-Vlan-interface1]ip address {IP地址} {子网掩码}
[SW-Vlan-interface1]telnet server enable
[SW]user-interface vty 0 4
[SW-line-vty0-4]auth password
[SW-line-vty0-4]set auth pass simple {密码}
[SW-line-vty0-4]user level-3
```
如上,之后在PC使用如下指令即可:
```
telnet {之前设置的ip地址}
```
如果使用交换机,记得设置交换机ip地址

---
### 链路聚合
首先,要新建立一个聚合组
```
[SW]interface bridge-aggregation {聚合端口号 0-1024}
```
<br>注意 : 同一个聚合下的端口必须都工作在全双工模式下且速率相同
<br>配置端口模式
```
[SW-GigabitEthernet1/0/1]duplex {工作模式: fall|half|auto}
[SW-GigabitEthernet1/0/1]speed {速率: 10|100|1000|auto}
```
加入一个端口
```
[SW]interface Ethernet {端口号,以1/0/1为例}
[SW-GigabitEthernet1/0/1]port link-aggregation {聚合端口号}
```
如上,端口 1/0/1 被 聚合 到 组2 内,之后,可以看一下结果
<br>显示聚合信息:
```
[SW]display link-aggregation summary
```
---

### 生成树协议配置
首先,为了使各交换机使用相同的生成树协议标准,先在个交换机执行以下命令
```
[SW]stp mode rstp
[SW]stp patchcost-standard legacy
```
因为STP默认开启,所以此时STP已打开
---
### VLAN配置
创建vlan将端口加入
```
[SW]vlan {vlan id}
[SW-vlan2]prot GigabitEthernet {起始端口号} to {终止端口号}
```
注意 : 也可以只写一个port来加入,这样创建的port类型为缺省值access port
要建立trunk port,需要首先对目标端口进行链路聚合(哪怕只有一个端口)
之后,进行如下设置
```
[SW-bridge-aggregation1]port link-type trunk
[SW-bridge-aggregation1]port trunk permit vlan {源Vlan id} to {目标vlan id}
```
---
## 路由器
---
### 使用TELNET登录路由器
```
[RT]telnet server enable  /缺省情况下，Telnet服务处于关闭状态
[RT]line vty 0   /进入一个或多个VTY用户线视图
[RT-line-vty0]authentication-mode scheme  /设置登录用户的认证方式为通过AAA认证
[RT]local-user test class manage（创建用户名）
[RT-luser-manage-test]password simple {密码}
[RT-luser-manage-test]service-type telnet
[RT-luser-manage-test]authorization-attribute user-role network-admin（设置登陆权限是超级用户最高权限）
```
---
### 路由协议设置
静态路由
```
ip route-static {dest ip addr} {mask} {next-hoop}
```
显示路由表
```
dsiplay ip routing-table
```
rip
```
[RT]rip
[RT-rip1]network {指定网段地址} {反子网掩码}
```
注意:需要添加所有需要的网段地址,包括自己

<br>
ospf

```
[RT]ospf
[RT-ospf-1]area 0
[RT-ospf-1-area-0.0.0.0]network {要启用的ip} {反子网掩码}
```

---
### 广域网协议设置
#### PAP
<br>pap为双向认证,需要双方都开启
<br>
<br>验证方
```
[RA-Serial1/0]ppp authentication-mode pap
[RA]local-user {用户名}
[RA-luser-manage-ra]service-type ppp
[RA-luser-manage-ra]password simple {密码}
[RA-Serial1/0]shutdown
[RA-Serial1/0]undo shutdwon
```
被验证方
```
[RB-Serial1/0]ppp pap local-user {用户名} password simple {密码} 
```
注意:此处用户名与密码须与上述相同
<br>此时只实现了单向验证,双方是无法ping通的,需要实现双向认证

---
#### CHAP
<br>验证方
```
[RA-Serial1/0]ppp authentication-mode chap
[RA-Serial1/0]ppp chap user {对方用户名}
[RA]local-user {自己用户名}
[RA-luser-manage-ra]service-type ppp
[RA-luser-manage-ra]password simple {密码}
[RA-Serial1/0]shutdown
[RA-Serial1/0]undo shutdwon
```
被验证方
```
[RA-Serial1/0]ppp chap user {对方用户名}
[RA]local-user {自己用户名}
[RA-luser-manage-ra]service-type ppp
[RA-luser-manage-ra]password simple {密码}
[RA-Serial1/0]shutdown
[RA-Serial1/0]undo shutdwon
```
---

### 防火墙
基于basic ACL
```
[RT]acl basic {acl编号2000~} match-order auto
[RT-acl-inv4-adv-2001]rule {permit|deny} ip {source|dest} {反子网掩码}
[RT-Eth/Ser]packet-filter 2001 {inbound|outbound}
```
基于advanced ACL
```
[RT]acl advanced {acl编号3000~} match-order auto
[RT-acl-inv4-adv-3001]rule {permit|deny} ip source {源ip地址|any} {反掩码|前为any时不写} destination {目的ip地址|any} {反掩码|前为any时不写}
[RT-Eth/Ser] packet-filter 3001 {inbound|outbound}
```

### NAT
首先要建立地址池
```
[RT]nat address-group {id} 
[RT-address-group-1]address {被映射ip组} {目标映射IP组}
```
同时要建立acl,如防火墙
<br>最后,应用在端口
```
[RT-Serial1/0]nat outbound {acl id} address-group {group id}
```
另外,对于具体到某个端口的nat映射,有如下指令
```
[RT-Serial1/0]nat server protocol tcp global {映射ip} {映射端口号} inside {被映射的ip} {被映射端口号}
```
