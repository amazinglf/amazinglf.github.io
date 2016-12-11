---
layout: post
title: "OC运行时及消息转发"
date: 2015-07-29 14:20:52 +0800
comments: true
categories: 
---

##类与对象的关系结构图
前面两个图合起来即表示后面这个图，理解图中实例、类、元类以及isa指针、superclass指针的原理非常重要，后面的消息发送及转发过程就需要基于这些图来更好的理解
![实例isa指针与类之间的关系](../images/runtime/image1.png)
![实例、类与元类metaclass关系](../images/runtime/image2.png)
![isa指针、superclass指针、实例、类与元类metaclass关系](../images/runtime/image3.png)

##与OC运行时进行交互的方式
（1）通过NSObject类提供的方法，如isKindOfClass:  isMemberOfClass:   respondsToSelector: conformsToProtocol: methodForSelector:等

（2）通过Runtime函数，参考Objective-C Runtime Reference.
##消息发送过程
1.OC中所有的消息[receiver message]都会转化为一个C语言函数的调用，即
objc_msgSend(receiver, selector, arg1, arg2, ...)

2.在objc_msgSend函数执行过程中，会首先根据接收者（someObject）和选择子（measageName）的类型在接收者所属的类中搜寻它的方法列表，这个方法列表其实是类里面的一张表格，表格是一个存放健值对的列表，健用选择子的名字表示，值即选择子对应方法的函数指针，同时会对这个方法列表进行缓存，查找之前会优先从缓存查找

3.如果在缓存列表或者方法列表中找到与选择子名称相符的方法，就跳转到相应方法的实现代码，如果找不到，就沿着继承体系继续向上查找，若找到合适的方法就会再跳转到该方法实现代码，如果最终找到了相应的方法，会将结果缓存到快速映射表，这样后面执行相同的查找时，就可以直接从缓存中取到对应的值，如果还是没找到相符合的代码，执行消息转发。即后面将要介绍的消息转发过程。 

下面的几个函数，与objc_msgSend函数类似，它们主要用于处理一些边界情况，可以了解一下

objc_msgSend_stret 待发送的消息要返回结构体   objc_msgSend_fpret 待发送的消息要返回浮点数  objc_msgSendSuper待发送的消息要给超类发消息。
##消息转发过程
走到消息转发过程这一步，即表示调用的方法在当前类中没有实现，或者一些其他的原因当前类无法处理这个方法对应的选择子，于是启动消息转发。其中消息转发又分为动态方法解析、备援接受者、完整的消息转发三个步骤。
###动态方法解析
类的接受者，看是否能动态添加方法，以处理当前无法解读的选择子，首先要在所属类中重写下面的类方法，＋(BOOL) resolveInstanceMethod ( SEL )，返回类型为BOOL，即表示能否处理，前面是为了处理未知的实例方法，如果是处理未知的类方法，重写 ＋(BOOL) resolveClassMethod ( SEL )，selector 参数,即未知的选择子，在这里有机会新增处理此选择子的方法，若方法的实现代码已经写好，运行的时候就会动态插入到类里面，CoreData的Dynamic属性即通过上面的方式实现。下面是通过resolveInstanceMethod方法处理未知选择子的实例代码，其中class_addMethod中的最后一个参数"v@:"，分号前面表示返回值类型及方法名，分号后面是参数类型，其中v表示void类型，@表示对象类型，关于更多类型的表示方法参考

	#import "RuntimeTestClass.h"
	#import "objc/runtime.h"
		@interface RuntimeTest2Class : NSObject
		- (void)unImplementMehtod2;
		@end
		@implementation RuntimeTest2Class
		- (id)init
		{
		    if (self = [super init]) {
		        return self;
		    }
		    return nil;
		}
		- (void)unImplementMehtod2
		{
		    NSLog(@"消息转发第二步，由当前这个类的对象来处理");
		}
		@end
		 
		@interface RuntimeTestClass ()
		@property (nonatomic, strong) RuntimeTest2Class *test2;
		@end
		@implementation RuntimeTestClass
		- (id)init
		{
		    if (self = [super init]) {
		        _test2 = [[RuntimeTest2Class alloc] init];
		        return self;
		    }
		    return nil;
		}
		-(id)forwardingTargetForSelector:(SEL)aSelector
		{
		    if ([self.test2 respondsToSelector:aSelector]) {
		        return self.test2;
		    }
		    return [super forwardingTargetForSelector:aSelector];
		}
		@end 
   
###备援接受者
看当前类有没有其他对象能处理这条消息，这一步进行处理的相应方法为 － (id) forwardingTargetForSelector: (SEL) selector; 方法参数即代码未知选择子，返回值为id，即代表能处理这条消息的其他对象，这一步转发的消息，我们可以试着看当前类的其他对象是否能处理这个选择子，如果能则返回当前类的这个对象。 实例代码如下所示 
		
		实例一
		#import "RuntimeTestClass.h"
		#import "objc/runtime.h"
		@interface RuntimeTest2Class : NSObject
		- (void)unImplementMehtod2;
		@end
		@implementation RuntimeTest2Class
		- (id)init
		{
		 if (self = [super init]) {
		     return self;
		 }
		 return nil;
		}
		- (void)unImplementMehtod2
		{
		 NSLog(@"消息转发第二步，由当前这个类的对象来处理");
		}
		@end
		
		@interface RuntimeTestClass ()
		@property (nonatomic, strong) RuntimeTest2Class *test2;
		@end
		@implementation RuntimeTestClass
		- (id)init
		{
		 if (self = [super init]) {
		     _test2 = [[RuntimeTest2Class alloc] init];
		     return self;
		 }
		 return nil;
		}
		-(id)forwardingTargetForSelector:(SEL)aSelector
		{
		 if ([self.test2 respondsToSelector:aSelector]) {
		     return self.test2;
		 }
		 return [super forwardingTargetForSelector:aSelector];
		}
		@end 
		
		实例二
		#import "RuntimeTestClass.h"
		#import <objc/runtime.h>
		@protocol RuntimeTestProtocol <NSObject>
		@required
		- (void)protocolMethod;
		@end
		 
		@interface RuntimeTest2Class : NSObject <RuntimeTestProtocol>
		@end
		@implementation RuntimeTest2Class
		- (id)init
		{
		    if (self = [super init]) {
		        return self;
		    }
		    return nil;
		}
		- (void)protocolMethod;
		{
		    NSLog(@"消息转发第二步，由当前这个类的对象来处理");
		}
		@end
		 
		@interface RuntimeTestClass ()
		@property (nonatomic, weak) id<RuntimeTestProtocol> delegate;
		@end
		@implementation RuntimeTestClass
		- (id)init
		{
		    if (self = [super init]) {
		        _delegate = [[RuntimeTest2Class alloc] init];
		        return self;
		    }
		    return nil;
		}
		-(id)forwardingTargetForSelector:(SEL)aSelector
		{
		    struct objc_method_description methodDesc = protocol_getMethodDescription(@protocol(RuntimeTestProtocol), aSelector, YES, YES);
		    if (methodDesc.name) {
		        return self.delegate;
		    }
		    return [super forwardingTargetForSelector:aSelector];
		}
		@end
		
###完整消息转发
与前面备援接受者的消息转发不太一样的是，这一步还可以修改消息的内容，即创建NSInvocation对象，重新赋值消息的参数等信息，把未能处理的消息的选择子，目标及参数封装到这个对象中，然后触发NSInvocation对象，把消息指派给目标对象，通过重写下列方法来转发消息。－ （void) forwardInvocation:(NSInvocation *) invocation。此方法默认实现，即改变调用目标，使消息在新目标上可以调用，这样与备援接受者实现的方法等效，只是在发送给备援接受者之前可以修改消息内容。重写前面方法之前，还要注意重写- (nullable NSMethodSignature *)methodSignatureForSelector:(SEL)sel方法，获取seletor相关的方法的描述以及方法实现的函数地址。
	
	#import "RuntimeTestClass.h"
	#import <objc/runtime.h>
	@protocol RuntimeTestProtocol <NSObject>
	@required
	- (void)protocolMethod;
	- (void)newprotocolMethod:(NSString *)name;
	@end
	@interface RuntimeTest2Class : NSObject <RuntimeTestProtocol>
	@end
	@implementation RuntimeTest2Class
	- (id)init
	{
	    if (self = [super init]) {
	        return self;
	    }
	    return nil;
	}
	- (void)protocolMethod;
	{
	    NSLog(@"消息转发第二步，由当前这个类的对象来处理");
	}
	- (void)newprotocolMethod:(NSString *)name
	{
	    NSLog(@"消息转发第三步，重新包装后执行未能处理的方法");
	}
	@end 
	@interface RuntimeTestClass ()
	@property (nonatomic, strong) id<RuntimeTestProtocol> delegate;
	@end
	@implementation RuntimeTestClass
	- (id)init
	{
	    if (self = [super init]) {
	        _delegate = [[RuntimeTest2Class alloc] init];
	        return self;
	    }
	    return nil;
	}
	- (void)forwardInvocation:(NSInvocation *)anInvocation
	{
	    struct objc_method_description methodDesc = protocol_getMethodDescription(@protocol(newprotocolMethod:), anInvocation.selector, YES, YES);
	    if (methodDesc.name) {
	        NSString *arg = @"forwardSuccess";
	        [anInvocation setArgument:&arg atIndex:2];
	        anInvocation.selector = @selector(newprotocolMethod:);
	        [anInvocation invokeWithTarget:self.delegate];
	    }  
	}
	- (NSMethodSignature *)methodSignatureForSelector:(SEL)selector
	{
	    NSMethodSignature *methodSignature = [super methodSignatureForSelector:selector];
	    if (!methodSignature) {
	        if ([self.delegate respondsToSelector:@selector(newprotocolMethod:)]) {
	            return [(id)self.delegate methodSignatureForSelector:@selector(newprotocolMethod:)];
	        }
	    }
	    return methodSignature;
	}
	
##应用例子
1.method swizzing，即运用Runtime函数将method对应的IMP实现替换掉，在消息发送过程直接调用替换后的方法实现

2.僵尸对象(NSZombieObject)的实现，即要释放当前对象时不真正释放，而是把对象的isa指针指向僵尸类，当前要释放的对象就变成了僵尸对象，由于僵尸类可以处理任何类型的消息，即给僵尸对象发消息时，会调用到僵尸类的方法，在方法中输出方法的调用方、方法名等信息，这样就可以方便我们通过调试信息查找问题原因了。具体有关僵尸类的定义后续可以补上，整个过程是在dealloc方法中实现的，要启用这个功能需要在xcode系统环境中开启如下选项。
![开启僵尸对象检查](../images/runtime/image4.png)

##附录
参考文档：https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html
