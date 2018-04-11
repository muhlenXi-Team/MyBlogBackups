---
title: APP 启动广告页的实现和封装
date: 2016-06-08 11:26:27
categories: blog
tags: [Objective-C,XYJLaunchAdvertisingView]
---

 *版权声明：本文为 muhlenXi 原创文章，欢迎转载，转载请注明来源: [http://muhlenxi.com/2017/06/08/APP-Advertisement](http://muhlenxi.com/2017/06/08/APP-Advertisement)*

### 导语：

> 也许你也注意到了，现在很多 App 在启动页加载完毕后，还会出现一个 n 秒的广告页面，页面中有一个倒计时的按钮，我们可以通过点击跳过那个按钮来跳过，我们点击广告的时候，会进入广告的详情页面。如果我们不做任何操作的话，当倒计时为 0 秒是会自动进入主页面。

  接下来我们就研究研究这个是如何实现的。

<!-- more -->

### 解决方案

我们仔细想想：不妨也有两种思路来解决。

* 1、一种是 APP 初次运行时，将广告页面的图片 URL 和要点击广告要跳转的 URL 数据通过服务器下载下来，然后再异步下载图片数据，最后将图片的数据和跳转URL保存到本地沙盒中。第二次运行 APP 的时候会显示广告界面，这时候再通过服务器更新本地沙盒中的数据。第一次由于本地沙盒中没有数据则不会显示。*目前大部分 APP 采用此方式*

* 2、第二种是每次启动时就通过服务器异步下载图片数据和跳转 URL，然后将其显示出来。这样做的优点时，可以实时更新广告页面数据。缺点是当网络出现阻塞时或无网络时，会出现一个空白的广告界面。

  针对该缺点，有一个解决思路，就是当网络不好时，用一个固定的图片和跳转 URL 来替换或者只用一个固定的图片来替换，点击广告则不跳转。但是当跳转 URL 失效时，需要通过迭代版本，重新上架来更新。
  
  
### 第一种方案。

[Demo 演示视频 传送门](http://v.youku.com/v_show/id_XMTcxNzQyMzg4NA==.html?)

#### XYJAdvertisementView 的实现

在 `Xcode` 中通过 `File` -> `New` ->`File...` 新建一个继承与 `UIVIew` 的`XYJLaunchAdvertisingView`；

> 编写程序的核心在于要有一个完整的思路！其次为了高效率的编程，我们需要学会一些偷懒的方法，比如宏定义的使用、类型常量的使用、变量的高效命名...

【1】在 `XYJAdvertisementView.h` 文件中，我们需要几个宏定义和类型常量。代码如下：

```objc
#define kScreenWidth [UIScreen mainScreen].bounds.size.width
#define kScreenHeight [UIScreen mainScreen].bounds.size.height
#define kScreenBounds [UIScreen mainScreen].bounds
static NSString * const adImageName = @"adImageName";
static NSString * const adDownloadUrl = @"adDownloadUrl";
static NSInteger  const adTime = 3;
static NSString * const pushToADNotiName = @"pushToADNotiName";
static NSString * const pushToADUrl = @"pushToADUrl";
```

*我们还需声明一个图片路径和一个显示广告的 show 方法。*

```objc
@property (nonatomic,copy) NSString * filePath; //!<  图片路径 用于属性传值
- (void) showAD;  //显示广告页面方法
```

【2】在 `XYJAdvertisementView.m` 文件中,在匿名类中声明我们需要的控件。

*声明如下：*

```objc
@property (nonatomic,strong) UIImageView * adImageView;
@property (nonatomic,strong) UIButton * countBtn;  //倒计时
@property (nonatomic,strong) NSTimer  * countTimer;
@property (nonatomic,assign) NSInteger  count;  //记录当前的秒数
```

**接下来我们依次实现相应的方法，我们要记住`高内聚低耦合`的原则**

*通过重写 `initWithFrame` 方法来搭建我们的UI界面*

```objc
- (instancetype)initWithFrame:(CGRect)frame
{
    self = [super initWithFrame:frame];
    if (self) {
        //其他控件的初始化写在这里
        //1.广告图片
        _adImageView = [[UIImageView alloc] initWithFrame:frame];
        _adImageView.userInteractionEnabled = YES;
        _adImageView.backgroundColor =  [UIColor yellowColor];
        _adImageView.contentMode = UIViewContentModeScaleAspectFill;
        _adImageView.clipsToBounds = YES;
        UITapGestureRecognizer * tap = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(tapGesHandle:)];
        [_adImageView addGestureRecognizer:tap];
        //2.跳过按钮
        CGFloat btnW = 60.0f;
        CGFloat btnH = 30.0f;
        _countBtn = [[UIButton alloc] initWithFrame:CGRectMake(kScreenWidth-btnW-24, btnH, btnW, btnH)];
        [_countBtn addTarget:self action:@selector(dismissAD) forControlEvents:UIControlEventTouchUpInside];
        _countBtn.titleLabel.font = [UIFont systemFontOfSize:15];
        [_countBtn setTitleColor:[UIColor whiteColor] forState:UIControlStateNormal];
        [_countBtn setTitle:[NSString stringWithFormat:@"跳过%ld",adTime] forState:UIControlStateNormal];
        _countBtn.backgroundColor = [UIColor colorWithRed:38/255.0 green:38/255.0 blue:38/255.0 alpha:0.6];
        _countBtn.layer.cornerRadius = 4;
        [self addSubview:_adImageView];
        [self addSubview:_countBtn];
    }
    return self;
}
```

*通过 `懒加载` 的方式实现我们的定时器 countTimer *

    所谓的懒加载也就是延时加载，即当对象需要用到的时候再去加载，简单理解就是，重写对象的get方法。当我们重写get方法时，一定要注意，先要判断当前对象是否为空，为空的话再去实例化对象。
    
    懒加载的优点如下：
    1、对象的实例化在get方法中实现，可以降低耦合度。
    2、不需要在viewDidLoad中再实例化对象，可以简化代码，同时增强代码的可读性。
    3、有效减少内存的占用率。
    
代码如下：

```objc
- (NSTimer *)countTimer
{
    if (_countTimer == nil) {
       _countTimer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(countDownEventHandle) userInfo:nil repeats:YES];
    }
    return _countTimer;
}
```

*实现我们的事件响应方法*

```objc
//广告界面点击
- (void) tapGesHandle:(UITapGestureRecognizer *) tap
{
   [self dismissAD];
   [[NSNotificationCenter defaultCenter] postNotificationName:pushToADNotiName object:nil userInfo:nil];
}
//定时器响应
- (void) countDownEventHandle
{
    _count--;
    [_countBtn setTitle:[NSString stringWithFormat:@"跳过%ld",_count] forState:UIControlStateNormal];
    if (_count == 0) {
        [self dismissAD];
    }
}
//跳过按钮触发
- (void) dismissAD
{
    [self.countTimer invalidate];
    self.countTimer = nil;
    [UIView animateWithDuration:0.3 animations:^{
    self.alpha = 0;
    } completion:^(BOOL finished) {
       [self removeFromSuperview];
    }];
}
```

*实现我们前面声明的 Show 方法*

```objc
//启动定时器
- (void) startTimer
{
    _count = adTime;
    [[NSRunLoop mainRunLoop] addTimer:self.countTimer forMode:NSRunLoopCommonModes];
}
//显示广告页面
- (void)showAD
{
    [self startTimer];
    UIWindow * window = [UIApplication sharedApplication].keyWindow;
    [window addSubview:self];
}
//图片赋值
- (void)setFilePath:(NSString *)filePath
{
    _filePath = filePath;
    _adImageView.image = [UIImage imageWithContentsOfFile:filePath];
}
```

到这里，我们的View就定制完成，我们还需要一个数据管理类来管理沙盒中的广告数据！

#### 数据管理类 XYJADDataManager 的实现

通过 `File` -> `New` ->`File...` 新建一个继承与 `NSObject` 的 `XYJADDataManager`；

【1】在 `XYJADDataManager.h` 文件中，声明一个添加广告的方法声明。代码如下：

导入头文件：

```objc
#import "XYJAdvertisementView.h"
```
方法声明：

```objc
+ (void) addXYJAdvertisementView;
```

【2】在 `XYJADDataManager.m` 文件中，实现相应的方法。代码如下：

**在实现show方法之前，我们要先实现一些帮助方法**

*通过图片的名字获取该图片在沙盒中的绝对路径*

```objc
+ (NSString *)getFilePathWithImageName:(NSString *)imageName
{
    if (imageName) {
        NSArray * paths = NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES);
        //图片默认存储在Cache目录下
        return [paths[0] stringByAppendingPathComponent:imageName];
    }
    return nil;
}
```

*判断该路径下是否存在文件*

```objc
+ (BOOL)isFileExistWithFilePath:(NSString *) filePath
{
    NSFileManager * fileManager = [NSFileManager defaultManager];
    BOOL isDirectory = NO;
    return [fileManager fileExistsAtPath:filePath isDirectory:&isDirectory];   
}
```

*删除旧的图片*

```objc
+ (void) deleteOldImage
{
    NSString * imageName = [[NSUserDefaults standardUserDefaults] objectForKey:adImageName];
    if (imageName) {
        NSString * filePath = [self getFilePathWithImageName:imageName];
        NSFileManager * fileManager = [NSFileManager defaultManager];
        if ([self isFileExistWithFilePath:filePath]) {
            [fileManager removeItemAtPath:filePath error:nil];
        }
    }
}
```

*向服务器请求广告数据*

**该方法中要根据实际项目需求做相应调整**

```objc
+ (void) UpdateAdvertisementDataFromServer
{
    //TODO 在这里请求广告的数据，包含图片的图片路径和点击图片要跳转的URL
    //我们这里假设从服务器中获得的 图片的下载URl和跳转URl如下所示
    NSString * imageurl = @"http://pic.paopaoche.net/up/2012-2/20122220201612322865.png";
    NSString * pushtoURl = @"http://www.jianshu.com";
	//获取图片名
   NSString * imageName = [[imageurl componentsSeparatedByString:@"/"] lastObject];
   //将图片名与沙盒中的数据比较
    NSString * oldImageName =[[NSUserDefaults standardUserDefaults] objectForKey:adImageName];
    if ((oldImageName == nil) || (![oldImageName isEqualToString:imageName]) ) {
        //异步下载广告数据
        [self downloadADImageWithUrl:imageurl iamgeName:imageName];
        //保存跳转路径到沙盒中
        [[NSUserDefaults standardUserDefaults] setObject:pushtoURl forKey:pushToADUrl];
        [[NSUserDefaults standardUserDefaults] synchronize];
    }
 }
```

*异步下载图片数据*

```objc
+ (void) downloadADImageWithUrl:imageUrl iamgeName:(NSString *) imageName
{
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        //TODO 异步操作
        //1、下载数据
        NSData * data = [NSData dataWithContentsOfURL:[NSURL URLWithString:imageUrl]];
        UIImage * image = [UIImage imageWithData:data];
        //2、获取保存文件的路径
        NSString * filePath = [self getFilePathWithImageName:imageName];
        //3、写入文件到沙盒中
        BOOL ret = [UIImagePNGRepresentation(image) writeToFile:filePath atomically:YES];
        if (ret) {
            NSLog(@"广告图片保存成功");
            [self deleteOldImage];
            //保存图片名和下载路径
            [[NSUserDefaults standardUserDefaults] setObject:imageName forKey:adImageName];
            [[NSUserDefaults standardUserDefaults] setObject:imageUrl forKey:adDownloadUrl];
            [[NSUserDefaults standardUserDefaults] synchronize];
        } else {
            NSLog(@"广告图片保存失败");
        }
    });
}
```

**实现 `addXYJAdvertisementView` 方法**

```objc
+ (void) addXYJAdvertisementView;
{
    //1.判断沙盒中是否存在广告的图片名字和图片数据，如果有则显示
    NSString * imageName = [[NSUserDefaults standardUserDefaults] objectForKey:adImageName];
    if (imageName != nil)
    {
        NSString * filePath = [self getFilePathWithImageName:imageName];
        BOOL isExist = [self isFileExistWithFilePath:filePath];
        //本地存在图片
        if (isExist) {
            NSLog(@"本地存在图片");
            XYJAdvertisementView * adView = [[XYJAdvertisementView alloc] initWithFrame:kScreenBounds];
            adView.filePath = filePath;
            [adView showAD];
        }
}
    //更新本地广告数据
    [self UpdateAdvertisementDataFromServer];
}
```

#### 最后一步，在 AppDelegate 文件中添加广告

* 1、在 `AppDelegate.h` 文件中导入 `XYJADDataManager.h` 文件 。
* 2、在 `didFinishLaunchingWithOptions` 方法中加入如下代码即可。

```objc
//添加启动广告
[XYJADDataManager addXYJAdvertisementView];
```

* 3、如果你需要获取广告点击事件，则需要进行如下操作。

在主界面ViewController的 `viewDidLoad` 方法中添加：

```objc
//添加广告点击的监听
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(pushToADVC) name:pushToADNotiName object:nil];
```

同时实现事件响应方法：

```objc
- (void) pushToADVC
{
    //TODO 在这里处理广告事件响应
    NSLog(@"广告点击了");
    XYJADWebViewController * webVC = [[XYJADWebViewController alloc] init];
    webVC.url = [[NSUserDefaults standardUserDefaults] objectForKey:pushToADUrl];
    [self.navigationController pushViewController:webVC animated:YES];
}
```


编译运行后，会发现跟我们前面的效果展示是一样的。

[点这里下载完整 Demo](https://github.com/muhlenXi/XYJLaunchAdvertisingView)

*目前暂无实现第二种方案*

*感谢阅读，有什么意见可以给我留言*
