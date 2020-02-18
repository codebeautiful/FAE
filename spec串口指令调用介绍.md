# spec串口指令调用介绍

## 概述

| method             | 设备miot-spec功能定义 |
| ------------------ | --------------------- |
| get_properties     | 可读**属性**          |
| set_properties     | 可写**属性**          |
| properties_changed | 变化上报**属性**      |
| action             | **方法**定义          |
| event_occured      | **事件**定义          |

## spec**串口指令定义** 

#### 读属性：get_properties

**命令格式**

```
get_properties <siid> <piid> ... <siid> <piid>
```

**应答格式**

```
<siid> <piid> <code> [value] ... <siid> <piid> <code> [value]
```

**示例：**

```
↑ get_down
↓ down get_properties 1 2 1 3
↑ result 1 2 0 10 1 3 0 "hello"
或
↑ result 1 2 0 10 1 3 -4001
```

#### 写属性：set_properties命令格式

**命令格式**

```
 set_properties <siid> <piid> <value> ... <siid> <piid> <value>
```

**应答格式**

```
 <siid> <piid> <code> ... <siid> <piid> <code>
```

**示例：**

```
↑ get_down
↓ down set_properties 1 1 10 1 88 "hello"
↑ result 1 1 0 1 88 -4003
```

#### 执行方法：action 

**命令格式**

```
action <siid> <aiid> <piid> <value> ... <piid> <value>
```

**应答格式**

```
<siid> <aiid> <code> <piid> <value> ... <piid> <value>
```

**示例**

```
↑ get_dow
↓ down action 1 1 1 10
↑ result 1 1 0 3 10
或
↑ result 1 1 -4004
```

#### 属性上报：properties_changed

**命令格式**

```
properties_changed <siid> <piid> <value> ... <siid> <piid> <value>
```

**应答格式**

```
ok/error
```

**示例**

```
↑ properties_changed 1 1 17 1 1 "hi"
↓ ok
或
↓ error
```

#### 事件上报：event_occured

**命令格式**

```
event_occured <siid> <eiid> <piid> <value> ... <piid> <value>
```

**应答格式**

```
ok/error
```

**示例**

```
↑ event_occured 1 1 1 17 2 "hi"
↓ ok
或
↓ error
```

#### code状态码定义

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