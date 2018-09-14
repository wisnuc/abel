# BLE API DOC

### BLE GATT Characteristic
BLE 通过`GATT Characteristic`显示只读字段，包括

| Characteristic | UUID |
| -------------- |:-------------:|
| Firmware Version | 2A26 |
| Manufacturer Name | 2A29 |
| Station Id | BEF1 |
| Station Status | BEF2 |
| Binding Progress | BEF3 |

+ **Station Id**:
  0x00: 未获得station id
  !0x00: station id，具体规则待定

+ **Station Status**: 用于表示station的配置状态
  0x00: 未获得station状态
  0x01: 已配置
  0x02: 待配置
  0x03: 待配置，且磁盘有数据
  0x04: 检测不到设备磁盘
  0x05: 其他情况，异常状态

+ **Binding Progress**: 首配过程中用于表示当前过程
  0x00: 首配未开始
  0x01: 已配置完成
  0x02: 设置与连接wifi中
  0x03: 连接云服务中
  0x04: 要求用户按下按键
  0x05: 配置失败

### SPS通讯
客户端通过蓝牙与station通讯，该协议下蓝牙只是转发

+ 通讯字段为BLE的Data characteristic (UUID : F000C0E1-0451-4000-B000-00000000-0000 )
+ 客户端发起的请求内容以JSON为格式，以`\n`结尾
+ SPS每次通讯最多为20bits，所以要大部分情况需要分段，如
```javascript
{
  session: '1234',
  op: 'binding',
  awsalb: 'blabla',
  essid: 'wifi-essid',
  password: 'wifi-password',
  csr: 'blabla'
}
```
需要分段为以下包，最后发送`0x0A`表示一个请求发送完毕
```
{"session":"1234","o
p":"binding","awsalb
":"blabla","essid":"
wifi-essid","passwor
d":"wifi-password","
csr":"blabla"}
```
+ station以相同的方式返回客户端的请求

### bled.js与蓝牙的通讯

+ 固件版本控制: 检测BLE状态，Bootloader还是application，
  1. bled发送`0x55 0x55` 
  2. 返回`0x00 0xCC`: BLE处于Bootloader且建立了通讯，进行刷写蓝牙固件的操作
  3. 返回`0x00 0xAA`: BLE处于application
  4. 返回其他内容或超时，重启蓝牙并重试过程

+ bled.js到BLE的命令的通用格式为 `0x00` + length + checksum + command + data

| Command | Command Value | Bytes in Packet | Description | Example |
| ------------- |:-------------:| :-----:|| -----:|-----:|
| PING | 0x30 | 5 | 带session的心跳连接 | `0x00 0x05 0x31 0x30 0x01` |
| VERSION | 0x31 | 4 | 获取BLE的固件版本 | `0x00 0x04 0x31 0x31`|
| UPDATE_STATION_ID | 0x32 |  5-20 | 更新station id | `0x00 0x07 0x35 0x32 0x01 0x01 0x01` |
| UPDATE_STATION_STATUS | 0x33 | 5 | 更新设备绑定状态 | `0x00 0x05 0x34 0x33 0x01`|
| UPDATE_BINDING_Progress | 0x34 | 5 | 更新首配状态 | `0x00 0x05 0x35 0x34 0x01` |

+ BLE到bled.js的输出的通用格式为 `0x00` + length + checksum + command + data
  1. 心跳包的返回(session为`0x01`): `0x00 0x05 0x31 0x30 0x01`
  2. 固件版本的请求结果 `0x00 0x6 0x32 0x31 0x01`
  3. 更新GATT的请求成功的返回 `0x00 0x05 0x72 0x32 0x40`

+ 启动过程
  1. bled.js与BLE模块分别启动
  2. BLE模块初始化状态并开始广播
  3. bled.js在Appifi启动后启动，获取设备的绑定状态、磁盘信息等 
  4. bled.js连接uart端口，并发起与BLE的通讯`0x55 0x55`
    a. 返回`0x00 0xCC`，进行刷写蓝牙固件的操作
    b. 返回`0x00 0xAA`: BLE处于application，下一步
    c. 返回其他内容或超时，重启蓝牙并重试过程4
  5. bled.js检测BLE固件版本
    a. 固件版本正常，下一步
    b. 固件版本较低或异常，重启蓝牙至SBL模式，刷写蓝牙固件
  6. bled.js将`Station Id`, `Station Status`更新至BLE的GATT

### 首配过程
  1. 客户端通过SPS向station发起绑定请求
  2. 根据绑定的进程，bled.js更新`Binding Progress`
  3. bled.js返回绑定结果
