---
layout: post
title: "Runloop学习调研"
date: 2016-12-11 16:32:17 +0800
comments: true
categories: 
---


###1、RunLoop与线程关系

![Alt text](../images/runloop3.png)   

 原理：从上面的表格可以看出，线程和RunLoop是息息相关的，即线程和RunLoop是一一对应的关系，从获取RunLoop的   源码(https://opensource.apple.com/source/CF/CF-1151.16/CFRunLoop.c.auto.html)可以看出，它是通过一个字典来实现线程和RunLoop之前一一对应关系的，<!--more-->  

 ![Alt text](../images/runloop1.png)   
 注意：CoreFoundation中方法是线程不安全的，而OC中方法是线程安全的，且OC方法是对CoreFoundation中方法的封装。使用CoreFoundation方法在保证线程安全的同时，要注意保证其不存在内存泄露问题  
 
####启动RunLoop  

   主线程的RunLoop是默认启动运行的，非主线程的RunLoop在获取后，需要调用RunLoop的相关run方法或者直接往runloop中添加输入源、定时器、观察者等来启动。
   
		//相关的run方法
		- (void)run;
		- (void)runUntilDate:(NSDate *)limitDate;
		- (BOOL)runMode:(NSRunLoopMode)mode beforeDate:(NSDate *)limitDate;
		//给runloop添加定时器、输入源的方法
		- (void)addTimer:(NSTimer *)timer forMode:(NSRunLoopMode)mode;
		- (void)addPort:(NSPort *)aPort forMode:(NSRunLoopMode)mode;
		//给runloop添加observer
		    NSRunLoop* myRunLoop = [NSRunLoop currentRunLoop];
		    CFRunLoopObserverContext  context = {0, self, NULL, NULL, NULL};
		    CFRunLoopObserverRef    observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
		            kCFRunLoopAllActivities, YES, 0, &myRunLoopObserver, &context);
		    if (observer)
		    {
		        CFRunLoopRef    cfLoop = [myRunLoop getCFRunLoop];
		        CFRunLoopAddObserver(cfLoop, observer, kCFRunLoopDefaultMode);
		    }
		//观察者的回调方法
		void MyRunLoopObserver(CFRunLoopObserverRef observer,
		                       CFRunLoopActivity activity,
		                       void* info)
		{
		    NSLog(@"INFO");
		    switch (activity) {
		        case kCFRunLoopEntry: {
		            NSLog(@"即将进入Loop");
		            break;
		        }
		        case kCFRunLoopBeforeTimers: {
		            NSLog(@"即将处理Timmer");
		            break;
		        }
		        case kCFRunLoopBeforeSources: {
		            NSLog(@"即将处理Source");
		            break;
		        }
		        case kCFRunLoopBeforeWaiting: {
		            NSLog(@"即将进入休眠");
		            break;
		        }
		        case kCFRunLoopAfterWaiting: {
		            NSLog(@"刚从休眠中唤醒");
		            break;
		        }
		        case kCFRunLoopExit: {
		            NSLog(@"即将退出Loop");
		            break;
		        }
		        case kCFRunLoopAllActivities: {
		            NSLog(@"所有的状态");
		            break;
		        }
		    }
		}
 
 直接通过run方法启动RunLoop时需要注意，在适当的时机给RunLoop添加事件源（source／timer／port）来触发RunLoop一直运行，保证其不会随着线程退出而退出。这个适当的时机是指在当前非主线程还没有退出之前，给RunLoop添加了事件源（source／timer／por／performselecter方法等t）即可，添加事件源与调用run方法之间没有顺序的限制。只要保证是在当前这个非主线程执行过程中。同时在非线程执行perfomrslector，必须保证其它线程有一个活跃的runloop也是同样的道理   
 不通过run相关方法，直接创建定时器和输入源，也需要通过上面列出的方法添加到当前runlooop上  
  通过下面的demo，在三种方式之间切换，可以验证上面的结论  
  
	@interface StartRunloopVC ()
	@property (nonatomic, strong) NSThread *thread;
	@end
	@implementation StartRunloopVC
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    self.view.backgroundColor = [UIColor lightGrayColor];
	    self.thread = [[NSThread alloc] initWithTarget:self selector:@selector(createRunloop) object:nil];
	    [self.thread setName:@"liangfang-thread"];
	    [self.thread start];
	    //方式二，在run之后，保证runloop不会退出
	//    [self performSelector:@selector(performsel) onThread:self.thread withObject:nil waitUntilDone:NO];
	}
	- (void)createRunloop
	{
	    NSRunLoop *runloop = [NSRunLoop currentRunLoop];
	    //方式一，在run之前，保证runloop不会退出
	//    [runloop addPort:[NSMachPort port] forMode:NSRunLoopCommonModes];
	    [runloop run];
	    NSLog(@"如果当前线程的runloop没有退出，则此处一直不会执行");
	}
	- (void)performsel
	{
	    NSLog(@"perform方法");
	}
	 
	- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
	{
	    NSLog(@"进入系统的下一次循环");
	    //方式三，下面是一个反例，下面的performsel方法永远不会执行，因为self.thread对应的runloop即线程已经退出了
	    [self performSelector:@selector(performsel) onThread:self.thread withObject:nil waitUntilDone:NO];
	}
	
原理：内部源码实现实际是通过do while() 循环来实现的

####退出RunLoop
	
主线程的RunLoop是一直不会退出的，除非应用结束，非主线程RunLoop的退出有两种方式  
（1）通过给RunLoop配置一个超时值，设置超时值的方式如下，此时就不能使用run了  
- (void)runUntilDate:(NSDate *)limitDate;  
- (BOOL)runMode:(NSRunLoopMode)mode beforeDate:(NSDate *)limitDate;  

	//在非主线程执行下面方法，doFireTimer方法执行10此之后，当前线程就退出了
	[NSTimer scheduledTimerWithTimeInterval:0.1 target:self selector:@selector(doFireTimer) userInfo:nil repeats:YES];
	[runloop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:1]];
 
 （2）通过调用 CFRunLoopStop 方法来退出，但是自己通过demo试了当前非主线程还是存在，好奇怪，官方文档说可以  
  ![Alt text](../images/runloop2.png)  
  
###RunLoop的不同模式及模式切换  
一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer，如下图示：http://cc.cocimg.com/api/uploads/20150528/1432798883604537.png  

系统默认注册了5个Mode，其中NSDefaultRunLoopMode（kCFRunLoopDefaultMode）、NSEventTrackingRunLoopMode、NSRunLoopCommonModes（kCFRunLoopCommonModes）是比较常用的，其它的Mode一般是系统在使用。  

DefaultRunLoop：默认的mode，主线程相关的事件响应等都在defaultmode下  
EventTrackingRunLoop：滑动界面，如scrollview的滑动监听在此mode下  
CommonMode：默认包含了所有的mode，譬如想让定时器在界面滑动的时候也生效，可以将定时器添加到commommode下，而不是defaultmode  
