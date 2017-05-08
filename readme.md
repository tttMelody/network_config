# Response
```
{
  "result": {
      请求的资源,为空则不显示
  },
  "success": true|false,
  "message": "error or success message",
  "code": 错误码
}
```

# API
1. GET /network/config 获取数据库中的网络配置
    - Response
           
          200:获取数据库配置成功
          500:获取数据库配置失败
          
2. GET /network/init 初始化网络,删除所有的bonds ,bridges, vlans,只开启一个管理口
3. GET /network/apply 应用数据库中的网络配置到系统

## Bond部分
1. POST /network/bond 新增Bond

    - Params:
    
          name: Bond的名字;
          mode: Bond的模式,取值为0~7,默认是0:
                    0:BALANCE_RR
                    1:ACTIVE_BACKUP
                    2:BALANCE_XOR
                    3:BROADCAST
                    4:802_3AD
                    5:BALANCE_TLB
                    6:BALANCE_ALB;
          dev: 组成Bond的接口,用方括号括起,多个接口用逗号隔开;
    - Example
    
          {"name":"bond0", "mode":"4", "devs": ["eth00","eth11"]}
          {"name":"bond1", "mode":"0", "devs": ["eth3"]}
          
    - Response
           
          201:在数据库中创建bond成功
          500:在数据库中创建bond失败,可能的原因有
              1. 从数据库中获取配置失败
              2. 用户输入校验未通过(name或者dev被占用)
              3. 把配置放入数据库失败
          具体的失败原因在响应的message字段表示.
          
2. DELETE /network/bond/name
Params:

3. PUT /network/bond
Params:
name
mode
dev

# Bridge部分
POST /network/bridge
Params:

DELETE /network/bridge/name
Params:

PUT /network/bridge
Params:

# Vlan部分
POST /network/vlan
Params:

DELETE /network/vlan/name
Params:

PUT /network/vlan
Params:

# IP部分
POST /network/ip
Params:

DELETE /network/ip
Params:

主要代码在src/interface.go,src/api_server.go

测试代码在src/interface_test.go

项目目录:/root/work/network_config

运行测试:sh /root/work/network_config/bin/test.sh
***
# 一.用户正常修改
1. 用户修改配置,api server收到具体的修改请求,比如/network/BridgeAdd
2. api server校验配置,比如名字是否重复,master是否重复
3. 校验通过修改到数据源
4. 用户点立即应用,api server收到这个请求后,把数据源中的配置应用到系统,直接返回给用户成功与否

# 二.配置应用到系统流程(数据源->系统)
0. 关闭接口
1. 删除现有网桥
2. 删除现有VLAN虚拟接口
3. 删除现有链路聚合设备
4. 创建链路聚合设备(若需要)
    1. 初始化bonding模块(bond设备数量及模式)
    2. 依次组建各bonding设备
5. 创建VLAN虚拟接口(若需要)
6. 创建网桥(若需要)
7. 删除现有接口绑定的所有IP
8. 为每个接口绑定IP地址和子网掩码

# 三.说明
1. 一旦系统收到 "立即应用信号",不管配置有无改动,都把当前配置应用到系统(走一遍上面的流程),所以尽量修改完所有配置再点应用
2. 若是切换数据源,系统就默认执行一次立即应用,从新的数据源中取出配置应用到系统,完成配置切换
3. api sever主要负责**用户修改**到**数据源**,apply是负责**数据源**到**系统**

# 四.API
Apply(config Config)

**以下所有API都是修改数据源中的配置**
BridgeAdd(name string, dev []string, mtu int)

BridgeUpdate(name string, dev []string, mtu int) // can not modify name

BridgeDel(name string)

BondAdd(name string, mode int, dev []string)

BondUpdate(name string, mode int, dev []string) // can not modify name

BondDel(name string)

VlanAdd(name string, tag int, parent string)

VlanUpdate( name string, tag int, parent string) // can not modify name

VlanDel(name string)

AssignIP(name string, ip string, mask string)

DelIP(name string)

# 五.相关设置json
配置json
```
{
    "hostId":"",//集群中的哪台waf
    "config": [
        {
            "bridge": [
                {"name":"br1", "dev":"eth0 eth1", "mtu":1500, "stp":"off",[addr{"IP":"1.1.1.1", "Mask":"ffffff00"}]},
                {"name":"br2", "dev":"eth2 eth3", "mtu":1500, "stp":"off",[addr{"IP":"1.1.1.2", "Mask":"ffffff00"}]}        
            ],
            "bond": [
                {"name":"bond1", "dev":"eth4", "mode":"1",[addr{"IP":"1.1.1.3", "Mask":"ffffff00"}]}
            ],
            "vlan" :[
                {"name":"eth0.100", "parent":"eth5", "tag":"110",[addr{"IP":"1.1.1.4", "Mask":"ffffff00"}]}
            ]
        }
    ]
}
```

## 结构体
```
type Config struct {
	HostId  string
	Devices []Device
	Bonds   []Bond
	Bridges []Bridge
	Vlans   []Vlan
	//后期想到上面新的配置项可以加在这里
}

type Device struct {
	Index int
	Name  string
	Ips   []IPNet
}

type Bond struct {
	Index int
	Name  string
	Mode  netlink.BondMode
	Devs  []string
	Ips   []IPNet
}

type Bridge struct {
	Index int
	Name  string
	Devs  []string
	Ips   []IPNet
	Mtu   int
	Stp   string
}

type Vlan struct {
	Index  int
	Name   string
	Tag    int
	Parent string
	Ips    []IPNet
}

type IPNet struct {
	IP   net.IP
	Mask string // network mask
	mask net.IPMask
}
```

## FAQ
1. validate哪些东西? 目前是接口的name不能重名,一个接口只能有一个master(bond和bridge)
2. 什么才算是不能再拆的状态? 目前是系统中没有bond bridge和vlan
3. 更新失败是否回滚? 目前先不回滚,回滚的一种方案是把配置回退到上一版本(就是应用之前的那个版本),然后再对这个版本应用更改.但是由于一般执行失败可能是由于硬件原因,所以这样还是有很大可能性执行失败.
4. 直接返回执行是否成功给用户,系统不再记录状态