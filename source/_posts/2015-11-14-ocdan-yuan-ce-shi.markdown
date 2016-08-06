---
layout: post
title: "OC单元测试"
date: 2015-11-14 22:55:00 +0800
comments: true
categories: 
---

###1、XCode自带XCTest框架了解

       （1）创建一个工程，会自动为你创建一个单元测试的target，特点：文件名都是以tests结尾
       （2）分析一下X...XTests.m文件的内容，首先该类继承自XCTestsCase类，并且有三个方法，它们各自的功能如下：
         setUp方法用于在测试前设置好要测试的方法，
         tearDown则是在测试后将设置好的要测试的方法拆卸掉。
         testExample顾名思义就是一个示例。
        （3）运行单元测试，使用command＋u快捷键，我好像没有想到别的方法
         在这个文件中可以增加其他测试方法，注意方法名都以test开头，保证规范。可以在方法中编写系统提供的18个断言的测试用例。可以参考博客：http://blog.csdn.net/jymn_chen/article/details/21552941，
        （4）也可以新建其他Objective-C test case class的类来编写与上面类似方法的测试用例。
        
###2、Kiwi和Specta单元测试框架的对比

       首先知道这些框架在github上是可以找到的，与其他第三方库Masonry AFNetworking等框架的使用方式其实是差不多的。
       
      （1）Specta (BDD框架) 
       Specta是基于XCTest进行封装的，使用Specta，需要依赖别的第三方库，因此还要再引入OCmock/OCMockito以及Expecta/OCHamcrest一起配合使用，但是一般都会引入OCMock 和OCHamcrest 这两个一起使用。再加上OHHTTPStubs(http stub打桩)框架，
       
        目前主要使用Specta＋OCMock＋Expeata＋OHHTTPStubs结合的方式。其中各个框架的作用如下：
         
        OCMock Or OCMockito ：这两个都是用来mock对象，Stub方法的，区别在于使用OCMock的库比OCMockito的库多，而且文档和教程更加丰富。mock测试就是在测试过程中，对于某些不容易构造或者不容易获取的对象，用一个虚拟的对象来创建以便测试的测试方法。mock对象,这个虚拟的对象就是mock对象。mock对象就是真实对象在调试期间的代替品。在实际工程中的使用，我的理解是由于在面向对象编程中，需要测试的某个类，往往又依赖于其他的类，而我们不想去真正创建这个类，调用这个类的某些方法，而是模拟mock出一个这样的依赖类，stub它的方法，指定相应方法的返回值。
        
        Expecta Or OCHamcrest  ： 都是断言的扩展框架，Expecta不成熟，框架还有一些的问题。OCHamcrest更加成熟，而且可扩展性高，可以自定义自己的断言，更灵活。这是网上别人的观点，因为自己还没有尝试做很多实例代码，后面还有待完善。
 
        OHHTTPStubs：在实际工程使用中主要用于测试model层在进行网络数据请求时，对接口返回数据的处理（数据解析）是否正确。同时用这个工具来模拟从网络请求返回的json数据，这样可以在本地对这个数据进行任意修改来测试，而不是真正去网络请求数据。https://github.com/AliSoftware/OHHTTPStubs
        
     （2）Kiwi框架
     
        Kiwi包含了Specta和OCmock以及Expeata所有的功能。
        
       总结：对于常用的specta和wiki框架，我们可以根据需要进行选择，对于他们各自的优势，还要后面多测试和学习。注意这两个框架不能同时使用，因为网上有人测试，Kiwi与Specta是不能同时在项目中使用的，会Crash。

###3、框架的使用

    http://www.bubuko.com/infodetail-1030528.html
    http://www.cocoachina.com/ios/20150731/12859.html
    https://github.com/6david9/WWDC2015
    参照网上的一段
      BDD的理念: 不是写代码，而是讲故事。整个故事是由Given…When…Then组成。
      eg：BDD框架Kiwi的一段测试代码:
      describe(@"Team", ^{
      context(@"when newly created", ^{
        it(@"has a name", ^{
            id team = [Team team]; [[team.name should] equal:@"Black Hawks"];
         });
        it(@"has 11 players", ^{
            id team = [Team team]; [[[team should] have:11] players];
         });
    });
    });

      这个测试用例就是在说Given a Team,When newly created,it should have a name, and should have 11 players，基本上不需要注释就能知道在干嘛

  结合之前做的基于MVVM框架些的代码，做了一个打桩的测试，大致实现过程
  
     （1）创建ADModelHTTPStub类（.h .m文件），定义打桩的方法，截取符合相应字符串的网络请求的url，对这个请求的url返回本地的一个json文件来模拟返回的数据，前提是在创建好json文件，存放返回的数据
       + (void)stubFetchADInfoSucceed{
    [OHHTTPStubs stubRequestsPassingTest:^BOOL(NSURLRequest *request) {
        NSRegularExpression *re = [NSRegularExpression regularExpressionWithPattern:@"/test/v3/guanggao"
                                                                            options:0
                                                                              error:nil];
        return [re numberOfMatchesInString:[request.URL absoluteString] options:0 range:NSMakeRange(0, [[request.URL absoluteString] length])];
    } withStubResponse:^OHHTTPStubsResponse *(NSURLRequest *request) {
        NSData *data = [NSData dataWithContentsOfFile:[[NSBundle bundleForClass:[self class]] pathForResource:@"AdertisementInfo" ofType:@"json" inDirectory:nil]];
        return [OHHTTPStubsResponse responseWithData:data statusCode:200 headers:nil];
    }];
    }
    
    
     + (void)stubFetchADConfigSucceed{
    [OHHTTPStubs stubRequestsPassingTest:^BOOL(NSURLRequest *request) {
        NSRegularExpression *re = [NSRegularExpression regularExpressionWithPattern:@"/test/v3/perzhi"
                                                                            options:0
                                                                              error:nil];
        return [re numberOfMatchesInString:[request.URL absoluteString] options:0 range:NSMakeRange(0, [[request.URL absoluteString] length])];
    } withStubResponse:^OHHTTPStubsResponse *(NSURLRequest *request) {
        NSData *data = [NSData dataWithContentsOfFile:[[NSBundle bundleForClass:[self class]] pathForResource:@"ADConfig" ofType:@"json" inDirectory:nil]];
        NSDictionary *json = [NSJSONSerialization
                              JSONObjectWithData:data
                              options:kNilOptions
                              error:nil];
        NSLog(@"json:%@",json);
        return [OHHTTPStubsResponse responseWithData:data statusCode:200 headers:nil];
    }];
    }
    
（2）创建一个objective-c的.m文件，用spec语法进行创建

      SpecBegin(MyBannerModel)
      describe(@"MyBannerModel", ^{
    context(@"", ^{
        beforeAll(^{
            [ADModelHTTPStub stubFetchADInfoSucceed];
            [ADModelHTTPStub stubFetchADConfigSucceed];
        });        
        afterAll(^{
            [OHHTTPStubs removeAllStubs];
        });    
        it(@"", ^AsyncBlock{
            MyBannerModel *infoModel = [[MyBannerModel alloc] init];
            [[infoModel fetchSomeADInfoByID:@"2" withVersion:@"0.6.2"] subscribeNext:^(NSArray *infoArray) {
                expect(infoArray).notTo.beNil();
                expect([infoArray count]).to.equal(3);
                done(); 
            }];            
        });
        it(@"", ^AsyncBlock{            
            MyConfigModel *configModel = [[MyConfigModel alloc] init];
            [[configModel fetchWithApp:@"iphone"] subscribeNext:^(MTADConfig *config) {
                expect(config).to.beKindOf([MyADConfig class]);
                done();
            }]; 
        });    
    });
     });
    SpecEnd

###4、其他相关
    （1）logit test与application test区别
       logit test类似于白盒测试，用于测试工程中较细节的逻辑，application test类似于黑盒测试，或者接口测试，用于测试直接与用户交互的接口（服务器接口返回的数据在客户端展示是否正常）
      logit test与application test的区别还表现在setUp方法上， logit test只需要在setUp方法中初始化一些测试数据，application test需要在setUp方法中获取主应用的AppDelegate,供test方法调用。这样是网上有人这么说的， 但是自己不太明白真正的原因，猜想是应用测试需要调用接口？
      
       网上说要注意对于会侵入主应用的test bundle，在使用过程中要十分注意，不要让单元测试的资源覆盖主应用资源，否则会造成诡异的bug，我猜想这个可能就是要使用mock stub 等测试框架来模拟数据的原因，而不是直接使用主应用返回的数据。
       
     (2) xcode现在也支持与ui操作相关的测试，可以参照网上别人的探索
     http://www.cocoachina.com/ios/20150702/12253.html
     
    （3）RAC绑定相关测试的注意事项
      下面以一个登录页面文本输入框的绑定为例，摘录自网上实例。这里有一个关键点，emailTextField或passwordTextField必须调用sendActionsForControlEvents:UIControlEventEditingChanged方法，才能触发textField的text属性改变。
      
      技巧：要找到相应调用方法需要通过查看对应控件的RAC源码中将通知（相应方法）转化为信号的处理过程。
      SPEC_BEGIN(LoginViewControllerSpec)

     describe(@"LoginViewController", ^{
    __block LoginViewController *controller;

    beforeEach(^{
        controller = [UIViewController loadViewControllerWithIdentifierForMainStoryboard:@"LoginViewController"];
        [controller view];
    });

    afterEach(^{
        controller = nil;
    });

    describe(@"Email Text Field", ^{
        context(@"when touch text field", ^{
            it(@"should not be nil", ^{
                [[controller.emailTextField shouldNot] beNil];
            });
        });

        context(@"when text field's text is hello", ^{
            it(@"shoud euqal view model's email property", ^{
                controller.emailTextField.text = @"hello";
                [controller.emailTextField sendActionsForControlEvents:UIControlEventEditingChanged];
                [[controller.viewModel.email should] equal:@"hello"];
            });
        });
    });

    describe(@"Password Text Field", ^{
        context(@"when touch text field", ^{
            it(@"should not be nil", ^{
                [[controller.passwordTextField shouldNot] beNil];
            });
        });

        context(@"when text field' text is hello", ^{
            it(@"should equal view model's password property", ^{
                controller.passwordTextField.text = @"hello";
                [controller.passwordTextField sendActionsForControlEvents:UIControlEventEditingChanged];

                [[controller.viewModel.password should] equal:@"hello"];
            });
        });
    });
    });

     绑定的测试通过之后，可以进一步模拟绑定得到的数据，进行其他相关测试。
  
    （4）写与MVVM框架相关的单元测试的方法
        model层：主要通过打桩测试网络请求返回的数据，通过订阅相应方法的signal返回的信号进行测试
        viewmodel层：主要执行相应commad，然后订阅command中被赋值的属性数据，进行测试
        viewcontroller层：对一些绑定操作等的测试。

     （5）写单测过程中遇到的问题和解决办法
      在测试vm层相关command执行后返回的结果时，通常做法是通过RACObserver来观察返回值，使用
       [RACObserve(viewModel, sectionCollectionResults) subscribeNext:.... 或者
       [RACObserve(viewModel.sectionCollectionResults, objectsCount).....
       是需要取决于你要测试的代码，如果是在执行command返回时初始化sectionCollectionResults并添加元素，第一个可以订阅到信号。否则好似订阅不到的。需要通过下面的方式。
       
       使用[RACObserve(viewModel.sectionCollectionResults, objectsCount).订阅时，又由于commad在返回结果时会对sectionCollectionResults先进行clearall操作。
       
###(5) 各个框架的api文档
https://github.com/specta/specta
https://github.com/specta/expecta
http://ocmock.org/reference/
https://github.com/AliSoftware/OHHTTPStubs