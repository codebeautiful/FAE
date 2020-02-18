# Wi-Fi通用模组透传协议开发指南

## 概述

小米通用模组提供一套可读的串口文本命令，供外部芯片调用，一般适用于MIIO模块不是主控芯片的情况，即模组只负责网络通讯，而不关心业务逻辑。

**文本命令采用“一问一答”的方式，每条命令，无论执行成功与否，都会返回结果或错误提示。** 所有文本命令，如无特殊说明，都使用小写字母。

下文中，示例里的“↑”和“↓”分别代表MCU发出的命令和模组返回的结果。有时，也称二者为“上行”信息和“下行”信息。因为在涉及到云通信的命令里，“上”代表发往云端，“下”代表从云端发来。

一个典型的上行例子(温度传感器上报温度)：

1. 传感器探知当前温度为26度

2. 传感器调用MIIO芯片的props命令，上报温度属性:props temp 26 

一个典型的下行例子(手机app打开电源开关)：

1. 手机app发出打开开关指令到服务器
2. MIIO芯片接收到服务器的下行消息，并翻译为文本命令:set_power on
3. 主控MCU不停的使用get_down命令轮询MIIO芯片，得到了set_power on命令 
4. 主控MCU完成电源打开的操作
5. 主控MCU调用result ”ok”，告知MIIO芯片成功完成
6. MIIO芯片给MCU回复ok，告知MCU结果已收到

## 串口指令要求

串口指令为由mcu实现，通过串口发送给模组的请求或命令。

1. 串口配置:115200,8,1,N,N

2. 命令长度:小于800个字符。

3. 命令格式:cmd_name [cmd_params] ... [cmd_params]'\r'， 

   - 分隔符:采取空格分隔的方式，非字符串中的空格需要将空格转义成 \u0020

   - 结束符:回车符’\r’(即CR或0x0D)
   - 命令与参数应由合法字符构成，包括字母、数字、下划线。 


4. 字符串:
   - 本地配置命令没有字符串类型，不需要对参数额外添加“” 
   - 服务器的通信命令基于JSON，服务器通信命令参数中的字符串类型需要添加“”

## 上行串口命令

上行串口指令为由mcu实现，通过串口发送给模组的请求或命令。

### 基础配置指令

**基础配置指令需mcu上电即发送，建议按mcu_version、model、ble_config（仅双模模组需要）顺序发送。**

#### model

- 参数:无 或 <model_name_string> 
- 返回:无参数时，返回当前model字符串。带参数时，将芯片model设为参数字符串，并返回ok。如果参数字符串不合法，则返回error。

- 说明:产品model通过开发者平台创建产品申请。合法的model字符串有3部分构成:公司名、 产品类别名、产品版本。3部分以“.”连接，总长度不能超过23个字符。**MCU上电后应第一时间上报model。**

- 举例:

  ↑model

  ↓xiaomi.dev.v1

  ↑model xiaomi.prod.v2

  ↓ok

  ↑model

  ↓xiaomi.prod.v2

  ↑model company.product_name_that_is_too_long.v2 

  ↓error

  ↑model plug

  ↓error

#### mcu_version

- 参数:<mcu_version_string> 

- 返回:如果参数合法，则返回ok，非法则返回error

- 说明:上报MCU固件版本。要求必须是4位数字。**MCU应该在应用固件引导后，第一时间调用该命令上报版本。**

- 举例: 

  ↑mcu_version 0001 

  ↓ok

  ↑mcu_version A001

  ↓error

  ↑mcu_version 1 

  ↓error

**双模模组初始化配置，除了设置model和mcu_version外，还需通过ble串口指令配置pid。**

#### ble_config

- 参数:set <pid> <mcu_version>   dump

- 示例:

  ↑ ble_config set 156 0001

  ↓ ok

  ↑ ble_config dump

  ↓ ["product id":190,"version":1.3.0_0000]

- 即时返回:若参数为dump则返回模组当前PID及固件版本，否则返回ok/error 
- 说明:查看、设置模组ID及固件版本号。

### 服务器通信指令

#### get_down

- 参数:无
- 返回:down <method_name> <arg1>,<arg2>,<arg3> ...

- 说明:获取下行指令。如果有下行指令，则返回该指令，如没有，则返回none。**获取到下行指令，需处理完成后再进行get_down，如果对上一个方法还没有给出结果，则返回error。 MCU在获得下行方法后，有1s时间处理，并用result/error命令上传结果。超过1s后，则wifi模 块认为调用失败。需注意，处理下行指令的窗口期内，模组仅接收result/error、指令，其余均 返回error。**

- 返回值说明:如果没有下行命令，则返回down none。如果有下行命令，则返回命令名、命令 参数(如果有参数)。命令名和参数之间用空格隔开，多个参数之间用逗号隔开。参数可以是 双引号括起的字符串，或是数字。

- 举例:

  ↑get_down

  ↓down none

  ↑get_down

  ↓down power_on "light_on",60,0,1 

  ↑result "ok"

  ↓ok

  ↑get_down 

  ↓down power_off 

  ↑get_down 

  ↓error

#### result

- 参数:<value1> <value2> ... 返回:成功返回ok，失败返回error

- 说明:发送下行指令的执行结果，如果有多个返回值，则以空格分隔。发送成功后，模组返回 ok。

- 举例:

  ↑result 123 "abc" 

  ↓ok

#### error

- 参数:<message> <error_code> 返回:成功返回ok，失败返回error

- 说明:如果下行指令执行出错，可用该命令发送错误信息。发送成功后，返回MIIO芯片回复 ok。message必须是双引号括起的字符串，错误码必须是-9999 到 -5000之间的整数(含边界 值)。

- 举例:

  ↑error "memory error" -5003

  ↓ok

  ↑error "stuck" -5001 

  ↓ok

#### props

- 参数:<prop_name_1> <value_1> <prop_name_2> <value_2> ... 
- 返回:ok 或 error
- 说明:参数至少要有一个名值对。

- 举例:

  ↑props temp 21 hum 50 location "home"

  ↓ok

  ↑props location office (格式非法)

  ↓error 
  
  限制:属性名的字符个数不得多于31个(可等于31)，一条指令最多上报15对kv。

#### event

- 参数:<event_name> <value_1> <value_2> ... 
- 返回:ok 或 error
- 说明:发送上行事件。

- 举例: 

  ↑event overheat 170 

  ↓ok

  ↑event button_pressed

  ↓ok

  ↑event fault "motor stuck" "sensor error" 

  ↓ok
  ↑event fault motor stuck (格式非法) 

  ↓error

  ↑event fault "motor stuck"

  ↓ok ... (短时间内大量event)

  ↑event fault "motor stuck" (此时，事件队列已满) 

  ↓error

- 限制:
   a. 事件名的字符个数不得多于31个(可等于31)
   b. 如果MIIO芯片断网，事件会在芯片里缓存最长10分钟，如果这段时间内网络未恢复正常， 则事件会被丢弃
   c. MIIO芯片最多缓存8条事件，如果短时间内大量调用event命令，则会造成缓存列表填满，后续事件会被丢弃

### 控制指令

#### reboot

- 参数:无
- 返回:ok 说明:MIIO接收到该命令后，将在0.5秒内重启。

- 举例:

  ↑reboot 

  ↓ok

#### restore

- 参数:无
- 返回:ok 说明:MIIO接收到该命令后，将清除wifi配置信息，并在0.5秒内重启。

- 举例:

  ↑restore 

  ↓ok

#### factory

- 参数:无

- 返回:ok

- 说明:MIIO芯片收到该命令后，在0.5秒内重启，并进入厂测模式。该模式下，芯片会按照预 设置信息连接路由器。预设路由器SSID:miio_default，密码:0x82562647。需注意，工厂模 式需保证仅可在工厂触发。

- 举例:

  ↑factory 

  ↓ok

#### set_low_power

- 参数:"on"/"off"
- 返回:ok/error 说明:打开模组的低功耗功能。目前只有MHCW03P、ESP-WROOM-02D/U模组上有实现。

### 获取模组信息指令

#### net

- 参数:无

- 返回: offline 或 local 或 cloud 或 updating 或 uap 或 unprov 

- 说明:询问网络状态。返回值分别代表:连接中(或掉线)、连上路由器但未连上服务器、连上小米云服务器、固件升级中、uap模式等待连接、关闭wifi(半小时未快连)。

  | 返回值   | 说明                     |
  | -------- | ------------------------ |
  | offline  | 连接中(或掉线)           |
  | local    | 连上路由器但未连上服务器 |
  | cloud    | 连上小米云服务器         |
  | updating | 固件升级中               |
  | uap      | uap模式等待连接          |
  | unprov   | 关闭wifi(半小时未快连)   |

- 举例:

  ↑net 

  ↓offline 

  ↑net 

  ↓local 

  ↑net 

  ↓cloud

#### time

- 参数:无 或 posix 
- 返回:当前日期和时间，格式见举例。时间为UTC+8时间，即中国的时间。

- 举例:

  ↑time

  ↓2015-06-04 16:58:07 

  ↑time posix 

  ↓1434446397

#### mac

- 参数:无 

- 返回:当前模组的mac地址。

- 举例:

  ↑mac 

  ↓34ce00892ab7

### 串口工具调试指令

#### echo

- 参数:on 或 off 
- 返回:ok

- 说明:打开串口回显功能。该功能开启时，MIIO芯片的串口输出会对输入进行回显。这在使用 串口工具(比如SecureCRT或Xshell)时，提供方便。

- 注:通过串口工具进行串口调试时，不打开回显功能可能会无法正常调试。

#### help

- 参数:无
- 返回:所有支持的命令和参数格式

### 保留的下行命令

MIIO芯片保留了一些命令名，用来通知主控MCU特定的事件。这类命令同样可用get_down得到， 需要主控MCU做出相应响应。

#### MIIO_net_change

- **表示MIIO芯片的网络连接状态发生了变化**。其后的参数代表了芯片最新的网络状态。网络状态 请参考net命令。可根据此命令改变wifi状态灯状态。

- 举例:

  ↑get_down

  ↓down MIIO_net_change cloud

#### update_fw

- 说明:通过串口给主控MCU升级指令，详见<固件ota升级>。不同平台升级MCU固件大小限 制不同，MHCW03P模组不大于400KB，MHCWB3P模组不大于600KB。

- 返回:mcu可返回“ready”和“busy”。返回“ready”，则进入正常升级流程;返回“busy”，表示 mcu当前状态无法升级，向服务器返回升级失败。

- 注:详细介绍请见串口固件OTA升级
