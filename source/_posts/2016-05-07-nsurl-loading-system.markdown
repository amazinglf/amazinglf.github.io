---
layout: post
title: "NSURL Loading System"
date: 2016-05-07 15:45:29 +0800
comments: true
categories: 
---
###1、概述
NSURL Loading System主要用来描述使用标准internet协议与server端交互，以及与URLs进行交互的基础框架类。它是一组类和协议的集合，允许你的app去获取URL指定的内容，核心类即NSURL，通过它使我们的app可以操作URLs以及URLs指向的资源。

除了NSURL，这个基础框架还提供了包括加载URLs、上传数据到服务器、管理cookie存储、控制返回数据的缓存、处理证书存储和身份验证、编写自定义的协议扩展等一套丰富的功能集合。

NSURL Loading System主要支持通过以下协议来获取相应资源，包括文件传输协议、超文本传输协议、超文本加密传输协议、本地文件url传输协议、Data URLs传输协议（ftp://   http://  https://  file:///  data://），同时支持代理服务器和socks网关。

除了NSURL Loading System，iOS还支持其他API可以在其他应用如safari中打开网页等，即通过在UIApplication中使用openURL:打开相关URL，具体可参考UIApplication Class Reference.

NSURL Loading System包含了一系列helper类来协助URL loading类完成加载过程和相关行为，主要可以分为以下5大类，即协议支持、身份验证与证书、cookie存储、配置管理、缓存管理，如下图链接

https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/URLLoadingSystem/Art/nsobject_hierarchy_2x.png

###2、URL Loading

URL Loading是在NSURL Loading System中通过url获取相关内容使用最普遍的一个类，你可以通过很多种方式获取相关内容，这依赖于你app的需求、app系统版本，以及你是否希望以文件的形式获取数据，还是以内存的形式获取。

在iOS7及之后的版本，推荐使用NSURLSession API来执行URL请求，如果相应应用必须支持旧的版本，你可以通过NSURLConnectionx下载数据到内存，然后根据需要看是否将数据写入磁盘，或者使用NSURLDownload下载数据到磁盘文件，你选择的方法很大程度上依赖于你希望获取数据到内存还是磁盘文件。

###如何使用NSURLSession
概览：NSURLSession及其相关类主要提供了通过http实现下载功能的API。提供了包括支持请求认证、后台下载（app在没有运行状态、暂停状态下的一些）等功能。这些功能都通 过其代理方法获取。

###理解NSURLSession的基本概念

一个session中tasks的行为依赖于三件事情：session的类型（依赖于创建时配置对象的类型），task的类型，当task创建的时候app是否在前台。

####1.session的类型
Default sessions（默认）：与其它下载url的基础方法非常相似，使用了持久的基于磁盘的缓存，并且将认证信息存储到用户的keychain里面。

Ephemeral sessions（短暂）：它不存储任何数据到磁盘，所有的缓存、认证信息等都存储在RAM 随机存取存储器中与session进行关联，当app的session失效时，它们会自动清除。

Background sessions（后台）：与默认session非常相似，除了它需要使用一个单独的进程来处理所有的数据传输，并且它在某些方面存在着一些限制。

####2.task的类型

data tasks：用于发送和接受NSData类型的对象，适用于与服务端的短暂频繁交互的客户端请求。返回的数据可以通过一个comletion handler block一定时间内的部分或者所有的数据。

download tasks：以file文件的形式获取数据，并且支持app没有运行时的后台下载

upload tasks：以file文件的形式上传数据，并且也支持app没有运行时的后台上传

####3.后台数据传输的注意事项

对于Background sessions，由于实际的数据传输是i通过一个进程来执行的，并且重新启动app的过程相对代价也很大，有很多features功能也不可用，因此会存在如下一些限制：

（3-1）session必须提供一个event delivery事件交互的delegate，delegate的行为和当前进程数据传输过程保持一致

（3-2）仅仅支持http和https协议（不支持custom protocols）

（3-3）Redirects are always followed（重定义一直存在？）

（3-4）仅仅支持file文件格式的上传任务（上传data类型或stream类型的对象在程序退出时会失败）

（3-5）如果后台数据传输是在程序处于后台的时候初始化的，configuration配置对象的discretionary属性会设置为true。

注意：在iOS 8 and OS X 10.10之前，data tasks 不支持Background sessions。

对于iOS 和 OS X，在app重新启动时，处理方式也有细微差别。

在iOS中，当一个后台数据传输任务完成或者需要鉴权，如果你的app不在运行状态，iOS会在后台自动重启你的app，并且调用当前app的UIApplicationDelegate 对象的application:handleEventsForBackgroundURLSession:completionHandler: 方法，这个调用方法提供了引起当前app启动的session的 identifier 标识符信息，当前你的app应该存储completion handler信息，并且用同样的identifier创建一个background configuration object，并且用这个configuration object创建一个session，新的session就会自动与正在进行的后台任务建立关联。接下来，当session完成最后一个下载任务，它会给session代理发送一个URLSessionDidFinishEventsForBackgroundURLSession:消息，在这个代理方法中，切换到主线程上调用之前存储的completion handler，让你的操作系统知道 重新suspend 挂起你的app也是安全的。
同样的在iOS 和 OS X中，当用户重启你的app，对于上一次运行时的 outstanding tasks，你的app应该马上创建针对这个tasks的相关session，并且保证创建的background configuration object的identifier与相应session是一一对应的。这些新创建的session同样会自动与正在进行的后台任务建立关联，并且新下载文件的url会与此进行关联，

注意：你必须准备的创建每一个identifier对应的session，多个session共享同一个identifier的行为是undefined不支持的。

当你的app处于suspended挂起状态，有task 任务执行完成，则会调用task的URLSession:downloadTask:didFinishDownloadingToURL:代理方法。
同样的，如果task需要鉴权等认证过程，NSURLSession对象会调用它的URLSession:task:didReceiveChallenge:completionHandler: 代理方法，或者是URLSession:didReceiveChallenge:completionHandler:代理方法。
在出现网络错误的情况下，background sessions中的上传和下载任务，会由URL加载系统自动重试，不需要使用可达性api来确定何时重试失败的任务

对于使用NSURLSession的后台传输任务的例子，可以参考Simple Background Transfer.

###生命周期及与代理的交互

充分理解session的生命循环，对于处理NSURLSession相关的任务是非常有帮助，包括session怎样与它的代理交互，代理方法调用的顺序，如果server端返回一个重定向会发生什么，当你的app恢复一个错误的下载会发生什么，等等，关于NSURLSession生命周期的完整描述，参考Life Cycle of a URL Session

###NSCoping行为
session和task对象都遵循如下的NSCoping协议：
当你的app拷贝了一个session或者task对象，会返回一个同样的对象
当你的app拷贝了一个configuration对象，会返回一个你可以独立修改的新的对象