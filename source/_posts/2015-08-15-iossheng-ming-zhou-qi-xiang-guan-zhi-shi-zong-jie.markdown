---
layout: post
title: "iOS生命周期相关知识总结"
date: 2015-08-15 13:18:11 +0800
comments: true
categories: 
---
1、 iOS工程中的各文件和目录的作用

    Tests文件用于单元测试（commad＋u 执行单侧命令)
    Supporting files文件夹下Info.plist的文件，该文件对工程做一些运行期的配置，非常重要，不能删除
    Localiztion native development region(CFBundleDevelopmentRegion)-本地化相关
    Bundle display name(CFBundleDisplayName)-程序安装后显示  的名称,限制在10－12个字符，如果超出，将被显示缩写名称
    Icon file(CFBundleIconFile)-app图标名称,一般为Icon.png
    Bundle version(CFBundleVersion)-应用程序的版本号，每次往          
    App Store上发布一个新版本时，需要增加这个版本号
    Main storyboard file base name(NSMainStoryboardFile)-主storyboard文件的名称
    Bundle identifier(CFBundleIdentifier)-项目的唯一标识，部署到真机时用到
    .pch文件，是一个头文件(Xcode6没有找到)pch头文件的内容能被项目中的其他所有源文件共享和访问一般在pch文件中定义一些全局的宏
    
    
2、app的生命周期

  （1）执行main.m文件的main函数，实现下面两个操作
  
      创建UIApplication对象
      创建UIApplication的delegate对象
      main函数中执行了一个UIApplicationMain这个函数：
     int UIApplicationMain(int argc, char *argv[], NSString *principalClassName, NSString *delegateClassName); 各个参数含义如下：
      argc、argv：直接传递给UIApplicationMain进行相关处理即可
      principalClassName：指定应用程序类名（app的象征），该类必须是UIApplication(或子类)。如果为nil,则用UIApplication类作为默认值
     delegateClassName：指定应用程序的代理类，该类必须遵守  UIApplicationDelegate协议，UIApplicationMain函数会根据principalClassName创建UIApplication对象，根据delegateClassName创建一个delegate对象，并将该delegate对象赋值给UIApplication对象中的delegate属性。
     
     以创建的sigle view applicaiton为例，相应方法实现为：return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
     （2）接着会建立应用程序的Main Runloop（事件循环）  
     如果有storyboard，根据Info.plist获得最主要storyboard的文件名,加载最主要的storyboard
     创建UIWindow
     创建和设置UIWindow的rootViewController（即storyboard属性面板的class属性定义的视图控制器）显示窗口
    （再去执行delegate的相关方法）
    如果没有storyboard，delegate对象开始处理(监听)系统事件程序启动完毕的时候, 就会调用代理的application:didFinishLaunchingWithOptions:方法在application:didFinishLaunchingWithOptions:中创建UIWindow创建和设置UIWindow的rootViewController显示窗口

   有一点不明白的地方就是，如果有storyboard的时候，是显示窗口之后，再去调用appdelegete代理的方法，还是先调用appdelegete代理的方法，再显示storyboard的窗口（解决办法，在代   理的application:didFinishLaunchingWithOptions:方法中创建uiwindow或对它重新赋值，看是否会覆盖storyboard）结论是先从storyboard加载，再执行代理方法。  
 
 
 
   （3）UIWindow
   
 UIWindow是一种特殊的UIView，通常在一个app中只会有一个UIWindow
iOS程序启动完毕后，创建的第一个视图控件就是UIWindow，接着创建控制器的view，最后将控制器的view添加到UIWindow上，于是控制器的view就显示在屏幕上了

    一个iOS程序之所以能显示到屏幕上，完全是因为它有UIWindow,也就说，`没有UIWindow，就看不见任何UI界面
      /**
     *  程序启动完毕就会调用一次
     */
      - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
    {
    // 1.创建window
    self.window = [[UIWindow alloc] initWithFrame:[[UIScreen mainScreen] bounds]];
    // 2.设置window的背景色
    self.window.backgroundColor = [UIColor whiteColor];    
    MjOneViewController *one = [[MjOneViewController alloc] init];
    // 方法一 :加入到UIWindow  [self.window addSubview:one.view];
    //方法二 推荐
    self.window.rootViewController = one;
    // 3.显示window
    [self.window makeKeyAndVisible];
    return YES;
    }
    
     最后，程序正常退出时UIApplicationMain函数才返回。
     
     
   3、对象的生命周期
   
    任何对象都是继承自NSObject的，可以通过重写下列方法，设置对象的初始化。
    + (void)load;
    + (void)initialize;
    - (instancetype)init



   4、视图控制器的生命周期
   
     loadView     从nib载入视图 ，通常这一步不需要去干涉。
     loadView 此方法在控制器的view为nil的时候被调用。 此方法用于以编程的方式创建view的时候用到。 如：
    - ( void ) loadView {
    UIView *view = [ [ UIView alloc] initWithFrame:[ UIScreen mainScreen] .applicationFrame] ;
    [ view setBackgroundColor:_color] ;
    self.view = view;  }
    
     viewDidLoad方法：此方法为控制器已被实例化，在视图被加载到内存中的时候调用该方法，这个时候视图并未出现。在该方法中通常进行的是对所控制的视图进行初始化处理。
      viewDidLoad方法在应用运行的时候只调用一次，而其他4个方法viewWillAppear   viewDidAppear   viewWillDisapper    viewDidDisapper可以被反复调用多次，它们的使用很广泛但同时也具有很强的技巧性。
     iOS系统在低内存时情况下会调用didReceiveMemoryWarning:该方法的主要职能是释放内存，包括视图控制器中的一些成员变量和视图的释放
 
      ViewController的生命周期中各方法执行流程如下：
      init—>loadView—>viewDidLoad—>viewWillApper—>viewDidApper—>viewWillDisapper—>viewDidDisapper—>viewWillUnload->viewDidUnload—>dealloc
      
     修改应用初始启动时关联的storyboard文件，通过代码创建初始启动的视图控制器
     - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.
    //重新设置self.window
    self.window = [[UIWindow alloc] initWithFrame:[[UIScreen mainScreen] applicationFrame]];
    ViewController *vc = [[ViewController alloc] init];
    UINavigationController *nvc = [[UINavigationController alloc] initWithRootViewController:vc];
    self.window.backgroundColor = [UIColor whiteColor];
    //设置window上加载的视图控制器
    self.window.rootViewController = nvc;
    //使改动生效
    [self.window makeKeyAndVisible];
    NSLog(@"%s",__FUNCTION__);
    return YES;
    }


   
   通过创建storyboard加载视图控制器的方法：
创建一个storyboard，New File --> IOS --> User Interface --> Storyboard
设置storyboard右侧的class属性为相应视图控制器类
在代码中创建storyboard的同时，启动相应的视图控制器
UIStoryboard *two = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
[self.navigationController pushViewController:[two instantiateInitialViewController] animated:YES];
创建视图控制器时勾选创建xib，自动生成对应xib文件
New File --> IOS --> Source --> Cocoa Touch Class(勾选Also create Xib File)


   通过手动创建带xib的视图控制器
创建一个xib文件，New File --> IOS --> User Interface --> view
FiveViewController *fiveVC = [[FiveViewController alloc]initWithNibName:@"other" bundle:nil];
    [self.navigationController pushViewController:fiveVC animated:YES];
    
    
    
5、 视图控制器装载和卸载视图的过程

   ViewController的职责主要包括管理内部各个view的加载显示和卸载，同时负责与其他ViewController的通信和协调 
        ViewController可以分为两类：
        一类是显示内容的，比如UIViewController、UITableViewController等，同时还可以自定义继承自UIViewController的ViewController；另一类是ViewController容器，UINavigationViewController和UITabBarController等
         在控制器view加载过程中首先会调用loadView方法，在这个方法中主要完成一些关键view的初始化工作，比如UINavigationViewController和UITabBarController等容器类的ViewController；接下来就是加载view，加载成功后，会接着调用viewDidLoad方法，这里要记住的一点是，在loadView之前，是没有view的，也就是说，在这之前，view还没有被初始化。完成viewDidLoad方法后，ViewController里面就成功的加载view了，如下图右下角所示。
在Controller中创建view有两种方式，一种是通过代码创建、一种是通过Storyboard或Interface Builder来创建，后者可以比较直观的配置view的外观和属性
 关于loadView方法的重写，官方文档中有一个明显的注释，当通过代码方式去创建你自己的view时，在loadView方法中不应该调用super，如果调用[super loadView]会影响CPU性能。
  - (void)loadView  
{  
     CGRect applicationFrame = [[UIScreen mainScreen] applicationFrame];  
     UIView *contentView = [[UIView alloc] initWithFrame:applicationFrame];  
     contentView.backgroundColor = [UIColor blackColor];  
     self.view = contentView;   
} 
view加载过程.png 
       当系统发出内存警告时，会调用didReceiveMemoeryWarning方法，如果当前有能被释放的view，系统会调用viewWillUnload方法来释放view，完成后调用viewDidUnload方法，至此，view就被卸载了。此时原本指向view的变量要被置为nil，具体操作是在viewDidUnload方法中调用self.myButton = nil。
ViewController中view卸载过程.png
        loadView和viewDidLoad的区别就是，loadView时view还没有生成，viewDidLoad时，view已经生成了，loadView只会被调用一次，而viewDidLoad可能会被调用多次（View可能会被多次加载），当view被添加到其他view中之前，会调用viewWillAppear，之后会调用viewDidAppear。当view从其他view中移除之前，调用viewWillDisAppear，移除之后会调用viewDidDisappear。当view不再使用时，受到内存警告时，ViewController会将view释放并将其指向为nil。
        
        
        
6、视图UIView的生命周期

  常用方法如下：
- (void)removeFromSuperview;
- (void)insertSubview:(UIView *)view atIndex:(NSInteger)index;
- (void)exchangeSubviewAtIndex:(NSInteger)index1 withSubviewAtIndex:(NSInteger)index2;
 
- (void)addSubview:(UIView *)view;
- (void)insertSubview:(UIView *)view belowSubview:(UIView *)siblingSubview;
- (void)insertSubview:(UIView *)view aboveSubview:(UIView *)siblingSubview;
 
- (void)bringSubviewToFront:(UIView *)view;
- (void)sendSubviewToBack:(UIView *)view;
 
- (void)didAddSubview:(UIView *)subview;
- (void)willRemoveSubview:(UIView *)subview;
 
- (void)willMoveToSuperview:(nullable UIView *)newSuperview;
- (void)didMoveToSuperview;
- (void)willMoveToWindow:(nullable UIWindow *)newWindow;
- (void)didMoveToWindow;