---
layout: post
title: "MVVM框架学习实践"
date: 2015-09-20 17:54:08 +0800
comments: true
categories: 
---

##1、mvvm基本概念

       在MVC框架的基础之上，将view和view controller联系在一起作为一个整体，同时新建一个视图模型层view model，负责view和view controller这个整体与model层之间的通信。同时将原来ViewController中放置用户输入验证逻辑、视图显示逻辑、发起网络请求和其他各种各样的耗时操作放到 view model中。在MVVM框架的基础之上，再结合RAC响应式编程，通过在控制器层对模型层数据的绑定，来实现界面与模型层之间的数据交换。
##2、mvvm层次结构

         MVVM主要分为一下几层，
         model层：数据模型层
         view(view和view controller)层：显示自定义视图和和管理视图的控制器层
         view model层：连接model层和view层之间的桥梁，主要负责一些业务逻辑操作
##3、mvvm使用场景

    （1）结合RAC简化自定义view与model层之间数据交互在控制器中使用RAC、RACObserver宏对view中要展示数据与view model层获取的model层数据进行绑定。
      使用RACCommad来响应界面的交互，具体步骤如下：
      第一步：在自定义view中定义一个command，在子view点击操作中执行这个command，excute：可以看看官网文档上它的用法
      
      第二步：在controlller中，把它与自定义view中的相应command进行绑定。在绑定的时候就实现这个commad的具体操作。
      
      第三步：也可以在controlller中定义这个commad属性。在属性的set方法中返回它的具体实现，最后不能忘记还要对这个commad与自定义view中的相应command进行绑定。
       
      第四步：commad的执行一般分以下几种状态：正在执行、执行失败、执行完（可以返回一个数据，但是一般通过修改某个与view进行绑定的数据之后，就可以自动通知view界面进行修改了，所有不返回数据也是可以的）
      
      （2）将UITableView上的数据源（实现<UITableViewDataSource>协议及其方法的类）分离出来，放到vm层。这样使得代码逻辑更清晰，且更有利于做单元测试。如果设置M层，还可以对代码做进一步的改进。
                   实例代码如下所示：
 VM层


    //声明一个block类型
    typedef void (^TableViewCellConfigureBlock)(id cell, id item);
     //声明TableDataSource 类，实现UITableViewDataSource协议
    @interface TableDataSource : NSObject <UITableViewDataSource>
     //定义一个实例化方法,设置数组、重用标识符，并调用相应block
    - (instancetype)initWithItems:(NSArray *)anItems
               CellIdentifier:(NSString *)aCellIdentifier
           ConfigureCellBlock:(TableViewCellConfigureBlock)aConfigureCellBlock;
     //根据indexPath返回item数组中的相应值
    - (id)itemAtIndexPath:(NSIndexPath *)indexPath; 
    @end



    ＃import “TableDataSource.h”
    @interface TableDataSource ()
     //存放数据的数组
    @property (nonatomic, strong) NSArray  *items;
     //重用标识符
    @property (nonatomic, copy) NSString *cellIdentifier;
     //block对象
     @property (nonatomic, copy) TableViewCellConfigureBlock configureCellBlock;
     @end
 
     @implementation TableDataSource
     //自动生成get和set方法
     @synthesize items = _items;
     @synthesize cellIdentifier = _cellIdentifier;
     @synthesize configureCellBlock = _configureCellBlock;
 


     - (id)init {
    return nil;
     }
    - (instancetype)initWithItems:(NSArray *)anItems
               CellIdentifier:(NSString *)aCellIdentifier
           ConfigureCellBlock:(TableViewCellConfigureBlock)aConfigureCellBlock
     {
    self = [super init];
    if (self) {
        self.items              = anItems;
        self.cellIdentifier     = aCellIdentifier;
        self.configureCellBlock = [aConfigureCellBlock copy];
    }
    return self;
     }
 
 
    - (id)itemAtIndexPath:(NSIndexPath *)indexPath {
    return _items[(NSUInteger)indexPath.row];
    }
 
   pragma mark - UITableViewDataSource
   
    - (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
    return 1;
     }
 
     - (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return [_items count];
     }
 
     - (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:_cellIdentifier];
    id item = [self itemAtIndexPath:indexPath];
     if (!cell) {
         cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:_cellIdentifier];
     }
    //回调block。这个block是传进来的，所有实际是执行外面的代码
    self.configureCellBlock(cell, item);
    return cell;
    }
 
    @end


V层

     #import "TableViewController.h"
     #import "TableDataSource.h"
     static NSString * const kCellIdentifier = @"Cell";
     @interface TableViewController ()//定义一个实现UITableViewDataSource协议的类的属性。用于给tableview设置数据源
     @property (strong, nonatomic) TableDataSource *dataSource;
     @end

     @implementation TableViewController
     //自动生成get和set方法
     @synthesize dataSource = _dataSource;
    - (void)viewDidLoad
    {
    [super viewDidLoad];
    TableViewCellConfigureBlock cellConfigureBlock = ^(UITableViewCell *cell, NSString *item) {
        cell.textLabel.text = item;
    };
    NSArray *stringsArray = @[@"1", @"2", @"3", @"1", @"2", @"3", @"1", @"2", @"3", @"1", @"2", @"3", @"1", @"2", @"3", @"1", @"2", @"3", @"1", @"2", @"3", @"1", @"2", @"3"];
    self.dataSource = [[TableDataSource alloc] initWithItems:stringsArray
                                              CellIdentifier:kCellIdentifier
                                          ConfigureCellBlock:cellConfigureBlock];
    self.tableView.dataSource = _dataSource;
     }
 
    @end
         
   （3）下面的代码将用户登录的验证代码分离出来，但是应该不是完全的mvvm的模式，因为还是通过block进行的回调？还要多些代码体会理解
              类似VM层
              
    ＃import <Foundation/Foundation.h>
    //定义返回值参数的block的类型
    typedef void (^RWSignInResponse)(BOOL);
    @interface RWDummySignInService : NSObject
    //在该方法的实现中，对传入的用户名密码进行处理，然后给block传入参数，并调用block
    //定义一个包含输入参数及返回值block参数的函数。函数的返回值通过block传递出去，调用这个函数，执行block，就可以对这个返回值进行处理了。
    - (void)signInWithUsername:(NSString *)username password:(NSString *)password complete:(RWSignInResponse)completeBlock;
    @end


     ＃import "RWDummySignInService.h"
      @implementation RWDummySignInService
     //具体实现，传入用户名和密码，调用block（设置block的参数，然后回调传入的blobk的实现代码，
     //block作为方法的参数，调用方法的时候一定要实现这个block，方法的实现是调用这个block
     
     - (void)signInWithUsername:(NSString *)username password:(NSString *)password complete:(RWSignInResponse)completeBlock {
    double delayInSeconds = 2.0;
    //设置执行线程之前的等待时间,将前面的double类型进行一个转换
    dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC));
    //执行线程，参数popTime：延迟时间。参数dispatch_get_main_queue()：获取运行在主线程的Main queue
    dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
        BOOL success = [username isEqualToString:@"user"] && [password isEqualToString:@"password"];
        completeBlock(success);
    });
     }
    @end
 
V层

     $import <UIKit/UIKit.h>
     @interface RWViewController : UIViewController
     @end
 
 
      #import "RWViewController.h"
      #import "RWDummySignInService.h"
      #import "ReactiveCocoa/ReactiveCocoa.h"
      @interface RWViewController ()
      @property (weak, nonatomic) IBOutlet UITextField *usernameTextField;
      @property (weak, nonatomic) IBOutlet UITextField *passwordTextField;
      @property (weak, nonatomic) IBOutlet UIButton *signInButton;
     @property (weak, nonatomic) IBOutlet UILabel *signInFailureText;
     @property (strong, nonatomic) RWDummySignInService *signInService; //处理用户点击事件的接口（信号）
 
     @end
 
 
     @implementation RWViewController
 
     - (void)viewDidLoad {
      [super viewDidLoad];
     //下面是创建（有效状态信号），理解其含义的代码
    //我们可以使用map操作转换我们想要的数据，只需要它是一个对象，
    RACSignal *validUsernameSignal = [self.usernameTextField.rac_textSignal map:^id(NSString *text) {
        return @(text.length > 3);
      ]);
    }];
    RACSignal *validPasswordSignal = [self.passwordTextField.rac_textSignal map:^id(NSString *text) {
        return @(text.length > 3);
    }];
 


 //下面ReactiveCocoa定义的一些宏，感觉这个宏是把函数要返回的对象的属性都声明了，直接return就行了
 
    RAC(self.passwordTextField, backgroundColor) = [validPasswordSignal map:^id(NSNumber *passwordValid){
        return [passwordValid boolValue] ? [UIColor clearColor] : [UIColor yellowColor];
    }];
    
    RAC(self.usernameTextField, backgroundColor) = [validUsernameSignal map:^id(NSNumber *usernameValid) {
        return [usernameValid boolValue] ? [UIColor clearColor] : [UIColor yellowColor];
    }];

  //一开始将下面的combinelatest的参数直接用text，一致报找不到方法的错误，然后马上跟断点进行调试。发现应该要转换成信号。才可以
  
    RAC(self.signInButton, enabled) = [RACSignal combineLatest:@[self.usernameTextField.rac_textSignal, self.passwordTextField.rac_textSignal] reduce:^id(NSString *name, NSString *password){
        return @(name.length > 3 && password.length>3);
    }];

//正确的，将按钮点击事件映射到一个登录信号，但同时通过将事件从内部信号发送到外部信号，使这个过程变得扁平化。

    [[[self.signInButton rac_signalForControlEvents:UIControlEventTouchUpInside] flattenMap:^RACStream *(id value) {
        return [self signInSignal];
    }] subscribeNext:^(NSNumber *signedIn) {
        BOOL success = [signedIn boolValue];
        self.signInFailureText.hidden = YES;
        if (success) {
            [self performSegueWithIdentifier:@"signInSuccess" sender:self];
        }
        NSLog(@"sign in result :%@",signedIn);
        
    }];


     //创建处理点击事件的接口类对象
      self.signInService = [RWDummySignInService new];
     // 初始化提示框为隐藏
      self.signInFailureText.hidden = YES;
     }

    - (RACSignal *)signInSignal{
    return [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [self.signInService signInWithUsername:self.usernameTextField.text password:self.passwordTextField.text complete:^(BOOL success) {
            [subscriber sendNext:@(success)];
            [subscriber sendCompleted];
        }];
        return nil;
    }];
     }
     @end