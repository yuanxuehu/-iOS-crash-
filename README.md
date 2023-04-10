# iOS-crash
关于iOS一些常见crash问题的防护方案探讨

一、概述
二、防护方案
1、unrecognized selector crash 防护方案
2、NSTimer类型 crash 防护方案
3、KVO crash 防护方案
4、容器类crash 防护方案
5、野指针类型的crash 防护方案
三、方案实现
1、开启防护组件，可选择防护的问题类型
2、设置全局的未识别方法时的默认处理
3、判断指针是否为野指针
4、对NSTimer、KVO的防护


一、概述
针对iOS客户端日常开发中经常出现的一些crash问题，我们可以探讨其产生的原因，并利用Object-C的runtime对相关系统API的实现进行了替
换，来实现对这些问题的防护。
常见的一些crash问题如下：
1、unrecognized selector 未定义方法引发的异常。
2、因使用NSTimer不当引起的内存泄漏和crash问题。
3、因使用KVO不当导致的crash问题。
4、容器类（NSArray、NSDictionary等）边界越界、非法元素变更。
5、野指针。

二、防护方案
1、unrecognized selector crash 防护方案
利用Objective—C的消息转发机制，通过重写NSObject的四个方法来处理:
+ (BOOL)resolveClassMethod:(SEL)sel; //动态类方法决议机制,决议类方法
+ (BOOL)resolveInstanceMethod:(SEL)sel;//动态的对象方法决议，决议对象方法
- (id)forwardingTargetForSelector:(SEL)aSelector;//转发给其他的一个对象处理函数
- (void)forwardInvocation:(NSInvocation *)anInvocation;//灵活的将目标函数以其他形式执行

2、NSTimer类型 crash 防护方案
NSTimer所产生的问题的主要原因是因为其没有在一个合适的时机invalidate，同时还有NSTimer对target的强引用导致的内存泄漏问题，甚至
在定时任务触发时导致crash。
那么解决NSTimer的问题的关键点在于以下两点：
1：通过添加代理target，使得NSTimer对其target可以不强引用。
2：通过添加一个对象检查类来找到一个合适的时机，在确定NSTimer已经失效的情况下，让NSTimer自动invalidate。

3、KVO crash 防护方案
KVO使用不当导致的crash原因主要为：
1、KVO的被观察者dealloc时仍然注册着KVO导致的crash。
2、重复添加观察者或重复移除观察者。
可以让观察对象持有一个KVO的delegate,所有和KVO相关的操作均通过delegate来进行管理，delegate通过建立一张MAP表来维护KVO的整个关
系，这样做的好处有2个：
1：如果出现KVO重复添加观察或者移除观察者（KVO注册者不匹配的）情况，delegate，可以直接阻止这些非正常的操作。
2：被观察对象dealloc之前，可以通过delegate自动将与自己有关的KVO关系都注销掉，避免了KVO的被观察者dealloc时仍然注册着KVO导致的
crash。

4、容器类crash 防护方案
容器类的crash常见的有NSArray／NSMutableArray／NSDictionary／NSMutableDictionary／NSCache的crash。一些常见的越界，插入nil，等
错误操作均会导致此类crash发生。
可利用Objective-C的runtime将这些Container类型的类族的相关操作函数进行替换，在调用操作函数之前先进行必要的条件判断。
5、野指针类型的crash 防护方案
访问野指针导致的Crash占了很大一部分，野指针类型crash的表现为：
Exception Type:SIGSEGV，
Exception Codes: SEGV_ACCERR
1、利用Xcode提供的编译机制，让野指针的crash提前暴露出来：
开启 Malloc Scribble：
对已释放内存进行数据填充，从而保证野指针访问是必然崩溃的。
开启 Zombie Object：
将释放的对象标记为Zombie对象，再次给Zombie对象发送消息时，发生crash并且输出相关的调用信息。这套机制同时定位了发生cras
h的类对象以及有相对清晰的调用栈。
2、通过系统标准库中提供的 malloc_zone_t *malloc_zone_from_ptr(const void *ptr) 方法来判断指针指向的内存是否有效。

三、方案实现
1、开启防护组件，可选择防护的问题类型
// 开启常见crash保护
[ZCCrashProtector setupCrashProtectorWithStyle:ZCCrashProtectorStyleAll];
2、设置全局的未识别方法时的默认处理
[NSObject setUnrecognizedSelectorProtectorBlock:^(id _Nonnull self, SEL _Nonnull _cmd) {
         NSLog(@"未能识别的方法: %@", NSStringFromSelector(_cmd));
}];
3、判断指针是否为野指针
// mainVC是否为野指针
if (ZCValidPointer(self.mainVC)) {
}
4、对NSTimer、KVO的防护
//KVO
[self addObserver:self  forKeyPath:@"title"  options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld
 context:@"view title is change."];
//NSTimer
组件自动对调用NSTimer以下方法提供保护 （NSTimer对其target不强引用；在确定NSTimer已经失效的情况下，让NSTimer自动invalidate）：
scheduledTimerWithTimeInterval:target:selector:userInfo:repeats:
timerWithTimeInterval:target:selector:userInfo:repeats:
initWithFireDate:interval:target:selector:userInfo:repeats:
self.timer = [NSTimer scheduledTimerWithTimeInterval:5. target:self selector:@selector(timerFired:) userInfo:@"timer is
fired." repeats:YES];
调用NSTimer以下方法创建NSTimer对象如需要自动invalidate则需调用 autoInvalidateWithHolder: 方法：
scheduledTimerWithTimeInterval:repeats:block:
timerWithTimeInterval:repeats:block:
initWithFireDate:interval:repeats:block:
self.timerInBlock = [NSTimer scheduledTimerWithTimeInterval:4. repeats:YES block:^(NSTimer * _Nonnull timer) {
NSLog(@"timer is fired in blcok.");
}];
[self.timerInBlock autoInvalidateWithHolder:self];//如需自动invalidate一定要调用该方法
