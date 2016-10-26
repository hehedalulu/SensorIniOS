# SensorIniOS

---
title: iOS中各种传感器使用的介绍与教程
### all by BadCodeLu 转载请注明出处
最近有做到调用系统底层的一些SDK，在iOS10中，我们已知的传感器有 
环境光传感器 （调节屏幕的亮度）
距离传感器(打电话自动黑屏) 
磁力计传感器（感应周边的磁场）
内部温度传感器（提醒用户降温 防止损伤设备）
湿度传感器
（上面的都是不常用的 也就是说并没有什么卵用的 下面的比较有用
陀螺仪（感应设备的的平衡 或者说是持握方式）（赛车类游戏 重力弹球）
加速器 (感受设备的运动 摇一摇 记步器)

介绍几个吧 
### 1.距离传感器
距离传感器是通过添加观察者 监听通知来实现的
通知的名称是 UIDeviceProximityStateDidChangeNotification
监听的状态是 [UIDevice currentDevice].proximityState
使用的代码是
```
- (void)viewDidLoad
{
    [super viewDidLoad];
    // [UIApplication sharedApplication].proximitySensingEnabled = YES;
    [UIDevice currentDevice].proximityMonitoringEnabled = YES;
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(proximityStateDidChange) name:UIDeviceProximityStateDidChangeNotification object:nil];
}

- (void)proximityStateDidChange
{
    if ([UIDevice currentDevice].proximityState) {
        NSLog(@"手机靠近");
    } else {
        NSLog(@"手机离开");
    }
}
```



#### 加速度传感器和陀螺仪

要做加速度传感器和陀螺仪相关
我们需要了解CoreMotion.Framework这个框架
这个框架不仅仅提供给你获得实时的加速度值和旋转速度值，更重要的是，苹果在其中集成了很多算法，可以直接给你输出把重力加速度分量剥离的加速度，省去你的高通滤波操作，以及提供给你一个专门的设备的三维位置信息。
CoreMotion框架的使用
## 基本数据
加速度值      CMAccelerometerData
陀螺仪值      CMGyroData
设备motion值  CMDeviceMotion
  设备的motion值是通过加速度和旋转速度进行变换算出来的
  CMDeviceMotion的属性
      attitude：手机在当前空间的位置和姿势
      gravity：重力信息，重力加速度矢量在当前设备的参考坐标系中的表达
      userAcceleration：加速度信息
      rotationRate：即时的旋转速率，是陀螺仪的输出
## 使用的步骤
1. 初始化CMMotionManager管理对象
2. 调用管理对象的对象方法获取数据，有2种方式
3. 处理数据
4. 当你不需要使用的时候，停止获取数据
```
-(void)stopAccelerometerUpdates;//停止获取加速度计数据
-(void)stopGyroUpdates;//停止获取陀螺仪数据
-(void)stopDeviceMotionUpdates;//停止获取设备motion数据
```
## 获取数据的两种方式
Push方式：
提供一个线程管理器NSOperationQueue和一个回调Block，CoreMotion自动在每一个采样数据到来的时候回调这个Block，进行处理。在这种情况下，Block中的操作会在你自己的主线程内执行。
Pull方式：
你必须主动去向CMMotionManager要数据，这个数据就是最近一次的采样数据。你不去要，CMMotionManager就不会给你。


### 2.加速度传感器
加速器传感器的读取其实还是蛮酷的 监听的方式是通过设置代理实现的 它能够监测设备在xyz轴上不同的加速度情况
具体的实现逻辑是
1. 获取加速器的单例对象
` UIAccelerometer *accelerometer = [UIAccelerometer sharedAccelerometer]; `
2. 获取加速器的代理对象
` accelerometer.delegate = self; `
3. 设置采样间隔
` accelerometer.updateInterval = 0.3; `
4. 实现代理相关的方法 监听数据(分为pull和push两种不同的方法)
` - (void)accelerometer:(UIAccelerometer *)accelerometer didAccelerate:(UIAcceleration *)acceleration; `

使用起来的代码是这样的
1. 加速度使用pull方法获取的代码
```
- (void)useAccelerometerPull{
    //初始化全局管理对象
    CMMotionManager *manager = [[CMMotionManager alloc] init];
    self.motionManager = manager;
    //判断加速度计可不可用，判断加速度计是否开启
    if ([manager isAccelerometerAvailable] && [manager isAccelerometerActive]){
        //告诉manager，更新频率是100Hz
        manager.accelerometerUpdateInterval = 0.01;
        //开始更新，后台线程开始运行。这是Pull方式。
        [manager startAccelerometerUpdates];
    }
    //获取并处理加速度计数据
    CMAccelerometerData *newestAccel = self.motionManager.accelerometerData;
    NSLog(@"X = %.04f",newestAccel.acceleration.x);
    NSLog(@"Y = %.04f",newestAccel.acceleration.y);
    NSLog(@"Z = %.04f",newestAccel.acceleration.z);
}
```
2. 加速度使用push方法获取的代码
```
- (void)useAccelerometerPush{
    //初始化全局管理对象
    CMMotionManager *manager = [[CMMotionManager alloc] init];
    self.motionManager = manager;
    //判断加速度计可不可用，判断加速度计是否开启
    if ([manager isAccelerometerAvailable] && [manager isAccelerometerActive]){
        //告诉manager，更新频率是100Hz
        manager.accelerometerUpdateInterval = 0.01;
        NSOperationQueue *queue = [[NSOperationQueue alloc] init];
        //Push方式获取和处理数据
        [manager startAccelerometerUpdatesToQueue:queue
                 withHandler:^(CMAccelerometerData *accelerometerData, NSError *error)
         {
             NSLog(@"X = %.04f",accelerometerData.acceleration.x);
             NSLog(@"Y = %.04f",accelerometerData.acceleration.y);
             NSLog(@"Z = %.04f",accelerometerData.acceleration.z);
         }];
    }
}
```
### 2.陀螺仪传感器
陀螺仪传感器使用起来与加速度传感器使用起来是相似的
具体代码实现如下
1. 陀螺仪实现用pull方法获取的代码
```
//pull的方法实现陀螺仪
- (void)useGyroPull{
    
    // 1.判断加速计是否可用
    if (!self.Gyromanager.isAccelerometerAvailable) {
        NSLog(@"加速计不可用，请跟换手机");
        return;
    }
    
    // 2.开始采集数据
    [self.Gyromanager startAccelerometerUpdates];
    
    CMRotationRate rotationRate = self.Gyromanager.gyroData.rotationRate;
    NSLog(@"x:%f-y:%f-z:%f",rotationRate.x, rotationRate.y, rotationRate.z);

}
```
2. 陀螺仪的实现用push方法获取的代码
```
- (void)useGyroPush{
    //初始化全局管理对象
    CMMotionManager *manager = [[CMMotionManager alloc] init];
    self.Gyromanager = manager;
    //判断陀螺仪可不可以，判断陀螺仪是不是开启
    if ([manager isGyroAvailable] && [manager isGyroActive]){
        
        NSOperationQueue *queue = [[NSOperationQueue alloc] init];
        //告诉manager，更新频率是100Hz
        manager.gyroUpdateInterval = 0.01;
        //Push方式获取和处理数据
        [manager startGyroUpdatesToQueue:queue
                             withHandler:^(CMGyroData *gyroData, NSError *error)
         {
             NSLog(@"Gyro Rotation x = %.04f", gyroData.rotationRate.x);
             NSLog(@"Gyro Rotation y = %.04f", gyroData.rotationRate.y);
             NSLog(@"Gyro Rotation z = %.04f", gyroData.rotationRate.z);
         }];
    }
}

```
### tips
push的使用就是利用一个block 不断的回调获取的值 通过isAccelerometerAvailable这个属性来判断能否获取值 但是在官方文档中 isAccelerometerAvailable这个方法只有getter的方法 是没有setter的方法的 所以如果我们想要自己停止获取传感器的值 就要另加一个bool类型的参数自己传进去 当然也可以利用多线程进行时间的的限制 repeats设置成1次就好了 但是还是有这样那样的问题 如果大家有什么好的可以控制传感器进程的方法请给我评论哦 我会认真的看的
