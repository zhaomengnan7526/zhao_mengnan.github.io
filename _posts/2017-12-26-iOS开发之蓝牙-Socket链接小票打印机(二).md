---
layout:     post
title:      iOS开发之蓝牙/Socket链接小票打印机（二）
subtitle:   介绍iOS蓝牙相关的知识，包括搜索设备、连接设备、发送数据等
date:       2017-12-26 15:30:00
author:     赵梦楠
header-img: img/kevin-bhagat-343433.jpg
catalog: true
tags:
    - iOS
    - 蓝牙
    - 小票打印机
    - Socket
    - ESC/POS
--- 


## 前言

[上一篇](https://zhaomengnan.top/2017/12/19/iOS开发之蓝牙小票打印机(一)/)主要介绍了部分ESC/POS指令集，包括一些常用的排版指令，打印位图指令等。另外，还介绍了将图片转换成点阵图的方法。在这篇文章中，将主要介绍通过蓝牙连接打印机，发送打印指令相关知识。

## 蓝牙链接小票打印机

### 简介

蓝牙是一种支持设备间短距离通讯的无线电技术。iOS系统中，有四个框架支持蓝牙链接：

  - `GameKit.framework`： 只能用于iOS设备之间的连接，多用于蓝牙对战的游戏，iOS7开始已过期；
  - `MultipeerConnectivity.framework`：只能用于iOS设备之间的连接，从iOS7开始引入，主要用于替代`GameKit`；
  - `ExternalAccessory.framework `：可用于第三方蓝牙设备交互，但是蓝牙设备必须经过苹果MFi认证；
  - `CoreBluetooth.framework `：目前最iOS平台最流行的框架，并且设备不需要MFi认证，手机至少4S以上，第三方设备必须支持蓝牙4.0；这里介绍的链接打印机就是使用此框架，因此开始前要确保打印机是支持蓝牙4.0的；

**CoreBluetooth框架有两个核心概念，central（中心）和 peripheral（外设），它们分别有自己对应的API；这里显然是手机作为central，蓝牙打印机作为peripheral；**

### 步骤

#### 1.初始化中心设备管理

```
self.centralManager = [[CBCentralManager alloc] initWithDelegate:self queue:nil];
```

#### 2. 确认蓝牙状态

设置代理后，会回调此方法，确认蓝牙状态，当状态为`CBCentralManagerStatePoweredOn`才能去扫描设备，蓝牙状态变化时，也会回调此方法

```
- (void)centralManagerDidUpdateState:(CBCentralManager *)central
{
    NSString * state = nil;
    
    switch ([central state])
    {
        case CBCentralManagerStateUnsupported:
            state = @"The platform/hardware doesn&#39;t support Bluetooth Low Energy.";
            break;
        case CBCentralManagerStateUnauthorized:
            state = @"The app is not authorized to use Bluetooth Low Energy.";
            break;
        case CBCentralManagerStatePoweredOff:
            state = @"Bluetooth is currently powered off.";
            break;
        case CBCentralManagerStatePoweredOn:
            state = @"work";
            break;
        case CBCentralManagerStateUnknown:
        default:
            ;
    }
    
    NSLog(@"Central manager state: %@", state);
}
```

#### 3. 扫描外设
调用此方法开始扫描外设

注意：第一个参数指定一个`CBUUID`对象数组，每个对象表示外围设备正在通告的服务的通用唯一标识符（UUID）。此时，仅返回公布这些服务的外设。当参数为`nil`，则返回所有已发现的外设，而不管其支持的服务是什么。

```
[self.centralManager scanForPeripheralsWithServices:nil options:nil];
```
当扫描到4.0外设后会回调此方法，这里包含设备的相关信息，如名称、UUID、信号强度等；

```
/*
 扫描，发现设备后会调用
 */
- (void)centralManager:(CBCentralManager *)central didDiscoverPeripheral:(CBPeripheral *)peripheral advertisementData:(NSDictionary *)advertisementData RSSI:(NSNumber *)RSSI
{
    NSString *str = [NSString stringWithFormat:@"----------------发现蓝牙外设: peripheral: %@ rssi: %@, UUID:  advertisementData: %@ ", peripheral, RSSI,  advertisementData];
    NSLog(@"%@",str);
    if (![self.peripherals containsObject:peripheral]) {
        [self.peripherals addObject:peripheral];
    }
}
```
#### 4. 选择外设进行连接
调用此方法连接外设
`[self.centralManager connectPeripheral:peripheral  options:nil];`

注意：第一个参数是要连接的外设。第二个参数`options`是可选的`NSDictionary `,系统定义了一下三个键，它们的值都是NSNumber (Boolean)；默认为NO。当设置为YES，则应用进入后台或者被挂起后，系统会用Alert通知蓝牙外设的状态变化，效果是这样![锁屏](http://upload-images.jianshu.io/upload_images/1612722-587c48ef366ff9de.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![未锁屏](http://upload-images.jianshu.io/upload_images/1612722-65f7bc0dfd025fbf.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
CBConnectPeripheralOptionNotifyOnConnectionKey;连接时Alert显示
CBConnectPeripheralOptionNotifyOnDisconnectionKey;断开时Alert显示
CBConnectPeripheralOptionNotifyOnNotificationKey;接收到外设通知时Alert显示
```

```
    [self.centralManager connectPeripheral:peripheral  options:@{
                                                                 CBConnectPeripheralOptionNotifyOnConnectionKey : @YES,
                                                                 CBConnectPeripheralOptionNotifyOnDisconnectionKey : @YES,
                                                                 CBConnectPeripheralOptionNotifyOnNotificationKey : @YES
                                                                 }];
```
连接成功或失败，都有对应的回调方法

```
/*
 连接失败后回调
 */
- (void)centralManager:(CBCentralManager *)central didFailToConnectPeripheral:(CBPeripheral *)peripheral error:(NSError *)error
{
    NSLog(@"%@",error);
}
/*
 连接成功后回调
 */
- (void)centralManager:(CBCentralManager *)central didConnectPeripheral:(CBPeripheral *)peripheral
{
    peripheral.delegate = self;//设置代理
    [central stopScan];//停止扫描外设
    [peripheral discoverServices:nil];//寻找外设内所包含的服务
}
```

#### 5. 扫描外设中的服务和特征
连接成功后设置代理`peripheral.delegate = self`,调用`[peripheral discoverServices:nil];`寻找外设内的服务。这里的参数是一个存放`CBUUID`对象的数组，用于发现特定的服务。当传nil时，表示发现外设内所有的服务。发现服务后系统会回调下面的方法:

```
/*
 扫描到服务后回调
 */
- (void)peripheral:(CBPeripheral *)peripheral didDiscoverServices:(NSError *)error
{
    if (error)
    {
        NSLog(@"Discovered services for %@ with error: %@", peripheral.name, [error localizedDescription]);
        return;
    }
    for (CBService* service in  peripheral.services) {
        NSLog(@"扫描到的serviceUUID:%@",service.UUID);
        //扫描特征
        [peripheral discoverCharacteristics:nil forService:service];
    }
}
```
发现服务后，调用`[peripheral discoverCharacteristics:nil forService:service];`去发现服务中包含的特征。和上面几个方法一样，第一个参数用于发现指定的特征。为nil时，表示发现服务的所有特征。

```
/*
 扫描到特性后回调
 */
- (void)peripheral:(CBPeripheral *)peripheral didDiscoverCharacteristicsForService:(CBService *)service error:(NSError *)error
{

    if (error)
    {
        NSLog(@"Discovered characteristics for %@ with error: %@", service.UUID, [error localizedDescription]);
        return;
    }
    
    for (CBCharacteristic * cha in service.characteristics)
    {
        CBCharacteristicProperties p = cha.properties;
        if (p & CBCharacteristicPropertyBroadcast) {
            
        }
        if (p & CBCharacteristicPropertyRead) {
            self.characteristicRead = cha;
        }
        if (p & CBCharacteristicPropertyWriteWithoutResponse) {
            self.peripheral = peripheral;
            self.characteristicInfo = cha;
        }
        if (p & CBCharacteristicPropertyWrite) {
            self.peripheral = peripheral;
            self.characteristicInfo = cha;
        }
        if (p & CBCharacteristicPropertyNotify) {              
                self.characteristicNotify = cha;
                [self.peripheral setNotifyValue:YES forCharacteristic:self.characteristicNotify];
            NSLog(@"characteristic uuid:%@  value:%@",cha.UUID,cha.value);
            
        }
    }
    
}

```




## Socket链接小票打印机


## 参考
[Core Bluetooth Programming Guide](https://developer.apple.com/library/content/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/AboutCoreBluetooth/Introduction.html#//apple_ref/doc/uid/TP40013257-CH1-SW1)

[Getting the pixel data from a CGImage object](https://developer.apple.com/library/content/qa/qa1509/_index.html)

[Core Bluetooth Programming Guide](https://developer.apple.com/library/content/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/AboutCoreBluetooth/Introduction.html#//apple_ref/doc/uid/TP40013257?language=objc)