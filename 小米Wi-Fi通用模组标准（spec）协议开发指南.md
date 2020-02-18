# Wi-Fi通用模组标准（spec）协议开发指南

## 概述

MIIO芯片提供一套可读的串口文本命令，供外部芯片调用，一般适用于MIIO模块不是主控芯片的 情况，即MIIO芯片只负责网络通讯，而不关心业务逻辑。

**文本命令采用“一问一答”的方式，每条命令，无论执行成功与否，都会返回结果或错误提示。** 所有文本命令，如无特殊说明，都使用小写字母。

下文中，示例里的“↑”和“↓”分别代表MCU发出的命令和模组返回的结果。有时，也称二者为“上 行”信息和“下行”信息。因为在涉及到云通信的命令里，“上”代表发往云端，“下”代表从云端发来。

## 串口指令要求

串口指令为由MCU实现，通过串口发送给模组的请求或命令。

- 串口配置：115200,8,1,N,N
- 命令长度：小于800个字符。
- 命令格式：cmd_name [cmd_params] ... [cmd_params]'\r' 
- 分隔符：采取空格分隔的方式，非字符串中的空格需要将空格转义成 \u0020 
- 结束符：回车符'\r'(即CR或0x0D) 命令与参数应由合法字符构成，包括字母、数字、下划线。 
- 本地配置命令没有字符串类型，不需要对参数额外添加“” 
- 服务器的通信命令基于JSON，服务器通信命令参数中的字符串类型需要添加“”

## 上行串口指令

上行串口指令为由mcu实现，通过串口发送给模组的请求或命令。 

串口层默认配置为115200,8,N,1。 

指令传输长度不固定，依据格式传输即可，但每条长度最大限制为512个字符。

命令采取空格分隔的方式，第一个连续字符串为命令名，其后可跟若干参数，最后以回车符‘\r’结束 
(即CR或0x0D)，模组返回接口同样以回车符‘\r’结尾。命令与参数应由合法字符构成，包括字 母、数
字、下划线。参数中若包含字符串值，需要用双引号括起。

### 基础功能指令

**基础配置指令需mcu上电即发送，建议按mcu_version、model、ble_config（仅双模模组需要）顺序发送。**

#### model

- 参数：无 或 <model_name_string>

- 返回：无参数时，返回当前model字符串。带参数时，将芯片model设为参数字符串，并返回 ok。如果参数字符串不合法，则返回error。

- 说明：**产品model通过MIOT平台创建产品申请**。合法的model字符串有3部分构成:公司名、 产品类别名、产品版本。3部分以“.”连接，总长度不能超过23个字符。**MCU上电后应第一时间 上报model。**

- 举例：

  ```
  ↑model
  
  ↓xiaomi.dev.v1
  
  **↑model xiaomi.prod.v2**
  
  **↓ok**
  
  ↑model
  
  ↓xiaomi.prod.v2
  
  ↑model company.product_name_that_is_too_long.v2 ↓error
  
  ↑model plug
  
  ↓error
  ```
  
  

#### mcu_version

- 参数：<mcu_version_string> 返回:如果参数合法，则返回ok，非法则返回error

- 说明： 上报MCU固件版本。要求必须是4位数字。**MCU应该在应用固件引导后，第一时间调用该命令上报版本。另外，如果下行命令收到MIIO_mcu_version_req，也应该立即上报版本**。

- 举例：

  ```
  **↑mcu_version 0001**
  
  **↓ok**
  
  ↑mcu_version A001
  
  ↓error
  
  ↑mcu_version 1 
  
  ↓error
  ```

#### ble_config(仅双模模组需要)

- 参数：set <pid> <mcu_version> 或 dump 返回:若参数为dump则返回模组当前PID及固件版本，否则返回ok/error 

- 说明：查看、设置模组PID及固件版本号。

- 举例：

  ```
  ↑ ble_config set 156 0001
  
  ↓ ok
  
  ↑ ble_config dump
  
  ↓ ["product id":190,"version":1.3.0_0000]
  ```

### 服务器通信指令

#### get_down

- 参数：无

- 返回：down <method_name> <arg1>,<arg2>,<arg3> ... 

- 说明：获取下行指令。如果有下行指令，则返回该指令，如没有，则返回none。**获取到下行指令，需处理完成后再进行get_down，如果对上一个方法还没有给出结果，则返回error。 MCU在获得下行方法后，有1s时间处理，并用result/error命令上传结果。超过1s后，则wifi模 块认为调用失败。需注意，处理下行指令的窗口期内，模组仅接收result/error、指令，其余均 返回error。**

- 返回值说明：如果没有下行命令，则返回down none。如果有下行命令，则返回命令名、命令参数(如果有参数)。命令名和参数之间用空格隔开，多个参数之间用逗号隔开。参数可以是 双引号括起的字符串，或是数字。返回的命令名对应相应的设备功能定义，需要MCU根据设备实例定义处理对应操作。

- 举例：

  ```
  ↑get_down

  ↓down none

  ↑get_down

  ↓down set_properties 1 1 10 

  ↑result 1 1 0

  ↓ok

  ↑get_down 

  ↓down action 1 1 

  ↑get_down 
  
  ↓error
  ```

#### result

- 参数：根据下行指令不同格式不同

- 返回：ok 或 error 

- 说明：返回下行指令的执行结果。

- 格式参考文末**spec串口指令调用方式**

- 举例：

  ```
  ↑get_down

  ↓down set_properties 2 1 10 

  ↑result 2 1 0

  ↓ok ↑get_down

  ↓down get_properties 2 1 

  ↑result 2 1 10

  ↓ok 

  ↑get_down

  ↓down action 2 1

  ↑result 2 1 0
  
  ↓ok
  ```

#### properties_changed

- 参数: properties_changed <siid> <piid> <value> ... <siid> <piid> <value>

- 返回:ok 或 error 

- 说明:**用于属性上报，当都某个属性变化时，上报当前变化的属性，在联网成功后，上报当前的所有属性**。参数至少要有一组属性iid及值。

- 举例:

  ```
  ↑properties_changed 1 1 17 1 2 "hi" 

  ↓ok

  ↑properties_changed 1 1
  
  ↓error
  ```

#### event_occured

- 参数: event_occured <siid> <eiid> <piid> <value> ... <piid> <value> 

- 返回:ok 或 error 

- 说明:用于时间上报。

- 举例:

  ```
  ↑event_occured 1 1 1 17 2 "hi" 

  ↓ok

  ↑event_occured 1

  ↓error
  ```


#### error

- 参数：<message> <error_code>

- 返回：成功返回ok，失败返回error

- 说明：**如果下行指令异常，可用该命令发送错误信息**。发送成功后，返回MIIO芯片回复ok。 message必须是双引号括起的字符串，错误码范围为是-9999 到 -5000之间的整数(含边界 值)。

- 举例：

  ```
  ↑getdown
  
  ↓down sxx
  
  ↑error "undefined command" -9999 
  
  ↓ok
  ```

### 控制指令

#### reboot

- 参数：无

- 返回：ok 

- 说明：MIIO接收到该命令后，将在0.5秒内重启。

- 举例：

  ```
  ↑reboot 
  
  ↓ok
  ```

#### restore

- 参数：无 

- 返回：ok

- 说明：MIIO接收到该命令后，将清除wifi配置信息，并在0.5秒内重启。

- 举例:

  ```
  ↑restore 
  
  ↓ok
  ```

#### factory

- 参数：无

- 返回：ok

- 说明：MIIO芯片收到该命令后，在0.5秒内重启，并进入厂测模式。该模式下，芯片会按照预 设置信息连接路由器。预设路由器SSID:miio_default，密码:0x82562647。需注意，工厂模 式需保证仅可在工厂触发。

- 举例：

  ```
  ↑factory 
  
  ↓ok
  ```

### 获取模组信息指令

#### net

- 参数：无 

- 返回： 见下表

- 说明：询问网络状态。

  | 返回值   | 连接中(或掉线)           |
  | -------- | ------------------------ |
  | offline  | 连接中(或掉线)           |
  | local    | 连上路由器但未连上服务器 |
  | cloud    | 连上小米云服务器         |
  | updating | 固件升级中               |
  | uap      | uap模式等待连接          |
  | unprov   | 关闭wifi(半小时未快连)   |

- 举例：

  ```
  ↑net 
  
  ↓offline 
  
  ↑net 
  
  ↓local 
  
  ↑net 
  
  ↓cloud
  ```

#### time

- 参数:无 或 posix 

- 返回:当前日期和时间，格式见举例。时间为UTC+8时间，即中国的时间。

- 举例:

  ```
  ↑time
  
  ↓2015-06-04 16:58:07 
  
  ↑time posix 
  
  ↓1434446397
  ```

#### mac

- 参数:无 

- 返回:当前模组的mac地址。

- 举例:

  ```
  ↑mac 
  
  ↓34ce00892ab7
  ```

#### version

- 参数:无

- 返回:模组固件版本号 

- 说明:某些模组返回串口协议(本协议)的版本，2.x.x版本后返回模组固件版本

- 举例:

  ```
  ↑version 
  
  ↓2.0.0
  ```

### 串口工具调试指令

#### echo

- 参数：on 或 off 

- 返回：ok

- 说明:打开串口回显功能。该功能开启时，MIIO芯片的串口输出会对输入进行回显。这在使用串口工具(比如SecureCRT或Xshell)时，提供方便。

- 注:通过串口工具进行串口调试时，不打开回显功能可能会无法正常调试。

#### help

- 参数:无
- 返回:所有支持的命令和参数格式

### 下行串口指令

#### MIIO_net_change

- **表示MIIO芯片的网络连接状态发生了变化**。其后的参数代表了芯片最新的网络状态。网络状态 请参考net命令。**可根据此命令改变wifi状态灯状态。**

- 举例：

  ```
  ↑get_down
    
  ↓down MIIO_net_change cloud
  ```
#### get_properties

- 参数：`get_properties <siid> <piid> ... <siid> <piid>`

- 返回：`<siid> <piid> <code> [value] ... <siid> <piid> <code> [value]`

- 说明：查询属性，支持同时查询多个属性。

- 举例：

  ```
  ↑ get_down
    
  ↓ down get_properties 1 2 1 3
    
  ↑ result 1 2 0 10 1 3 0 "hello"
    
  或
    
  ↑ result 1 2 0 10 1 3 -4001 
  ```
#### set_properties

- 参数：`set_properties <siid> <piid> <value> ... <siid> <piid> <value>`

- 返回：`<siid> <piid> <code> ... <siid> <piid> <code>`

- 说明：设置属性值，支持同时设置多个属性。

- 举例：

  ```
  ↑ get_down

  ↓ down set_properties 1 1 10 1 88 "hello"
  
  ↑ result 1 1 0 1 88 -4003
  ```

#### action 

- 参数：`action <siid> <aiid> <piid> <value> ... <piid> <value>`

- 返回：`<siid> <aiid> <code> <piid> <value> ... <piid> <value>`

- 说明：设置属性值，支持同时设置多个属性。

- 举例：

  ```
  ↑ get_down
  
  ↓ down action 1 1 1 10
  
  ↑ result 1 1 0 3 10
  
  或
  
  ↑ result 1 1 -4004
  ```

### code状态码定义

| code  | 描述                         |
| ----- | ---------------------------- |
| 0     | 成功                         |
| 1     | 接收到请求，但操作还没有完成 |
| -4001 | 属性不可读                   |
| -4002 | 属性不可写                   |
| -4003 | 属性、方法、事件不存在       |
| -4004 | 其他内部错误                 |
| -4005 | 属性 value错误               |
| -4006 | 方法in参数错误               |
| -4007 | did错误                      |
