# Runtime的几点用法总结
## 一、runtime简介
* RunTime简称运行时。OC就是***运行时机制***，也就是在运行时候的一些机制，其中最主要的是消息机制。
* 对于C语言，***函数的调用在编译的时候会决定调用哪个函数***。
* 对于OC的函数，属于***动态调用过程***，在编译的时候并不能决定真正调用哪个函数，只有在真正运行的时候才会根据函数的名称找到对应的函数来调用。
* 事实证明：
	* 在编译阶段，OC可以***调用任何函数***，即使这个函数并未实现，只要声明过就不会报错。
	* 在编译阶段，C语言调用***未实现的函数***就会报错。

## 给类别Category添加属性
比如说我们需要在类别中添加一个 NSString 类型的属性，直接在 .h 文件添加 **@property(nonatomic,copy) NSString*categoryProperty;**,这时候使用点语法进行调用的话，程序会出现crash错误 ：***Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[ViewController setCategoryProperty:]: unrecognized selector sent to instance 0x7ff661e43dd0'***。这种状况的原因其实很简单，只是没有实现setter和getter方法而已，所以我们的问题就转为实现setter 和 getter方法。
一言不合就要上代码了，主要记录两种类型数据的处理方式。例子为给UIImage添加了两个属性，没什么具体含义，主要记录用法：

	//.h
	#import <UIKit/UIKit.h>
	NS_ASSUME_NONNULL_BEGIN
	@interface UIImage (MNZAdd)
	
	@property(nonatomic,strong) NSString *name;
	@property(nonatomic,assign) CGFloat add;
	
	@end
	
	NS_ASSUME_NONNULL_END
	
	//.m
	#import "UIImage+ MNZAdd.h"
	#import <objc/runtime.h>
	
	@implementation UIImage (MNZAdd)
	#pragma mark - 添加属性
	
	- (void)setName:(NSString *)name
	{
	    [self willChangeValueForKey:NSStringFromSelector(@selector(name))];
	    objc_setAssociatedObject(self, _cmd, name, OBJC_ASSOCIATION_COPY);
	    [self didChangeValueForKey:NSStringFromSelector(@selector(name))];
	}
	
	- (NSString *)name
	{
	    return objc_getAssociatedObject(self, @selector(setName:));
	}
	
	- (void)setAdd:(CGFloat)add
	{
	    [self willChangeValueForKey:NSStringFromSelector(@selector(add))];
	
	    //区别在这里，区别在这里
	    NSValue *value = [NSValue value:&add withObjCType:@encode(CGFloat)];
	    objc_setAssociatedObject(self, _cmd, value, OBJC_ASSOCIATION_RETAIN);
	
	    [self didChangeValueForKey:NSStringFromSelector(@selector(add))];
	}
	
	- (CGFloat)add
	{
	    CGFloat cValue = {0};
	    NSValue *value = objc_getAssociatedObject(self, @selector(setAdd:));
	    [value getValue:&cValue];
	    return cValue;
	}
	
	@end
	
## 利用runtime来替换已有的系统方法
例子，初始化UIImage的时候，在不同的系统版本中添加不同的风格的切图，怎么就是和UIImage过不去了。

	//.h
	#import <UIKit/UIKit.h>
	
	@interface UIImage (MNZAdd)
	/*!
	 @brief 如果调用这个，其实调用的是原来系统的方法，因为他两交换了实现
	
	 @note 为防止误用，可以不声明该方法
	 */
	+ (nonnull UIImage *)mnz_imageNamed:(NSString *)name;
	@end

	//.m
	#import "UIImage+MNZAdd.h"
	#import <objc/runtime.h>
	
	@implementation UIImage (MNZAdd)
	
	+ (void)load {
	    static dispatch_once_t onceToken;
	    dispatch_once(&onceToken, ^{
	        Class class = [self class];
	        
	        SEL originalSelector = @selector(imageNamed:);
	        SEL mySelector = @selector(mnz_imageNamed:);
	        /*
	            //实例方法
	            Method originalMethod = class_getInstanceMethod(class, originalSelector);
	            Method myMethod = class_getInstanceMethod(class, mySelector);
	         */
	        Method originalMethod = class_getClassMethod(class, originalSelector);
	        Method myMethod = class_getClassMethod(class, mySelector);
	        
	        class_addMethod(class, originalSelector, class_getMethodImplementation(class, originalSelector), method_getTypeEncoding(originalMethod));
	        class_addMethod(class, mySelector, class_getMethodImplementation(class, mySelector), method_getTypeEncoding(myMethod));
	        //交换方法的实现
	        method_exchangeImplementations(originalMethod, myMethod);
	    });
	}
	
	#pragma mark - 交换系统方法
	+ (nonnull UIImage *)mnz_imageNamed:(NSString *)name {
	    /*!
	     在这里实现我们所需要做的操作
	     */
	    double systemVersion = [[[UIDevice currentDevice]systemVersion] floatValue];
	    if (systemVersion >= 9.0) {
	        name = [name stringByAppendingString:@"_os"];
	    }
	    //这个地方很关键，mnz_imageNamed:是调用系统的imageNamed:实现(两个方法交换了实现)
	    UIImage *image = [UIImage mnz_imageNamed:name];
	    return image;
	}

方法+ (void)load不是这里的重点，简单知道一下：一般情况下，类别中的方法会重写掉主类里面相同命名的方法，但+load:是个特例，当一个类被读到内存的时候，runtime会给这个类以及他的每一个类别都发送一个 +load:消息（知道这一点很重要）。。  
**Note：注意交换方法只能执行一次，不要总是执行，load的意义在这儿也有体现的。**

还有一点是，尝试添加原 selector 是为了做一层保护，因为如果这个类没有实现 originalSelector ，但是其父类实现了，那么 class_getInstanceMethod 返回的将是父类的方法。这样就导致了 method_exchangeImplementations 替换的是父类的方法。所以先尝试添加 originalSelector ，如果已经存在，再用 method_exchangeImplementations 把原来的方法的实现交换成新方法的实现。

## 实现自动归档和自动解档
其实归档的实现很简单，只不过就是实现协议<NSCoding>，需要说明一点的是： **实现了 ‘NSCoding’协议，就可以支持数据类和数据流间的编码和解码，而数据流可以持久化到硬盘。**

	//.h
	#import <Foundation/Foundation.h>
	
	@interface MNZEncodeModel : NSObject<NSCoding>
	
	@property(nonatomic,strong) NSString *name;
	@property(nonatomic,assign) int age;
	@property(nonatomic,assign) NSRange range;
	
	@end
	
	//.m
	#import "MNZEncodeModel.h"
	#import "NSObject+MNZEncode.h"
	
	NSString *const kEncodeName = @"name";
	NSString *const kEncodeAge = @"age";
	NSString *const kEncodeRange = @"range";
	
	@implementation MNZEncodeModel
	
	- (instancetype)initWithCoder:(NSCoder *)aDecoder
	{
	    self = [super init];
	    if (self) {
	        _name = [aDecoder decodeObjectForKey:kEncodeName];
	        _age = [aDecoder decodeIntForKey:kEncodeAge];
	        _range = [[aDecoder decodeObjectForKey:kEncodeRange] rangeValue];
	    }
	    return self;
	}
	
	- (void)encodeWithCoder:(NSCoder *)aCoder
	{
	    [aCoder encodeObject:self.name forKey:kEncodeName];
	    [aCoder encodeInt:self.age forKey:kEncodeAge];
	    [aCoder encodeObject:[NSValue valueWithRange:self.range] forKey:kEncodeRange];
	}
	
	@end
	
可问题在于，像这样只有三个属性需要我们，可以这样写，那如果有三十个属性呐，身为一个会偷懒的程序员自然要找偷懒的方法了

	//.h
	
	#import <Foundation/Foundation.h>
	
	@interface MNZEncodeModel : NSObject <NSCoding>
	
	@property(nonatomic,strong) NSString *name;
	@property(nonatomic,assign) int age;
	@property(nonatomic,assign) NSRange range;
	
	@end
	
	//.m
	#import "MNZEncodeModel.h"
	#import <objc/runtime.h>
	
	@implementation MNZEncodeModel

	- (instancetype)initWithCoder:(NSCoder *)coder
	{
	    self = [super init];
	    if (self) {
	        unsigned int count = 0;
	        Ivar *ivars = class_copyIvarList([self class], &count);
	        for (int i=0; i<count; i++) {
	            Ivar ivar = ivars[i];
	            const char *name = ivar_getName(ivar);
	            NSString *key = [NSString stringWithUTF8String:name];
	            id value = [coder decodeObjectForKey:key];
	            [self setValue:value forKey:key];
	        }
	        free(ivars);
	    }
	    return self;
	}
	
	- (void)encodeWithCoder:(NSCoder *)aCoder {
	    unsigned int count = 0;
	    Ivar *ivars = class_copyIvarList([self class], &count);
	    for (int i=0; i<count; i++) {
	        Ivar ivar = ivars[i];
	        const char *name = ivar_getName(ivar);
	        NSString *key = [NSString stringWithUTF8String:name];
	        id value = [self valueForKey:key];
	        [aCoder encodeObject:value forKey:key];
	    }
	    free(ivars);
	}
	
	@end

**简化一下使用方式**

而这样每一个模型类都要写着无聊的代码，而大部分类都是继承自NSObject，所以，我们可以实现一个NSObject类别来专门做这件事。

	//.h
	
	#import <Foundation/Foundation.h>
	
	@interface NSObject (MNZEncode)
	
	- (instancetype)mnz_initWithCoder:(NSCoder *)aDecoder;
	- (void)mnz_encodeWithCoder:(NSCoder *)aCoder;
	
	@end
	
	//.m
	
	#import "NSObject+MNZEncode.h"
	#import <objc/runtime.h>
	
	@implementation NSObject (MNZEncode)
	
	- (instancetype)mnz_initWithCoder:(NSCoder *)aDecoder
	{
	    if (!aDecoder) return self;
	    if (self == (id)kCFNull) return self;
	    unsigned int count = 0;
	    Ivar *ivars = class_copyIvarList([self class], &count);
	    for (int i = 0; i < count; i++) {
	        //取出i对应位置的成员变量
	        Ivar ivar = ivars[i];
	        //查看成员变量
	        const char *name = ivar_getName(ivar);
	        //归档
	        NSString *key = [NSString stringWithUTF8String:name];
	        id value = [aDecoder decodeObjectForKey:key];
	        //设置到成员变量身上
	        [self setValue:value forKey:key];
	    }
	    free(ivars);
	    return self;
	}
	
	- (void)mnz_encodeWithCoder:(NSCoder *)aCoder
	{
	    if (!aCoder) return;
	    if (self == (id)kCFNull) {
	        [((id<NSCoding>)self)encodeWithCoder:aCoder];
	        return;
	    }
	    unsigned int count = 0;
	    Ivar *ivars = class_copyIvarList([self class], &count);
	
	    for (int i = 0; i < count; i++) {
	        Ivar ivar = ivars[i];
	        const char *name = ivar_getName(ivar);
	        NSString *key = [NSString stringWithUTF8String:name];
	        id value = [self valueForKey:key];
	        [aCoder encodeObject:value forKey:key];
	    }
	    free(ivars);
	}
	
	@end
	
这样在使用的时候只需要简单的引用一下就可以了

## 消息转发
**objc_msgSend**方法的使用

	objc_msgSend(receiver,selector)

或者传入参数

	objc_msgSend(receiver,selector,arg1,arg2,...)
	
对于一个给定的函数调用，如

	[self SendImage:fileName];
	
可以通过如下方法来替换：

	void (*action)(id, SEL, NSString*) = (void (*)(id, SEL, NSString*))objc_msgSend;
	action(self, @selector(SendImage:), fileName);
	
注意objc_msgSend函数总是以一个id变量和一个selector作为它的前两个参数。objc_msgSend 被转换成函数指针后，就可以通过这个函数指针进行函数调用了。

当一个message被发送给object，会根据object的isa 指针找到类结构里的方法，如果不能找到，一直顺着父类寻找该方法的实现，直到NSObject类。

为加快速度，runtime system 会缓存使用过的selector和方法地址。 

* 通过object的isa指针找到他的class
* 在class的method_list中找到方法
* 如果class中没有找到方法，继续往superclass中查找
* 一旦找到这个函数，执行对应的方法实现 （IMP）
* 找不到 Dynamic Method Resolution(动态方法决议) 如果是实例方法，调用**+ (BOOL)resolveInstanceMethod:(SEL)sel,**如果是类方法，调用**+ (BOOL)resolveClassMethod:(SEL)sel，**这样可以让程序在运行时动态的为一个selector提供实现。如果返回YES，运行时系统会重启一次消息的发送过程，调动动态添加方法。


		+(BOOL)resolveInstanceMethod:(SEL)sel
		{
		  if (sel == @selector(foo)) {
		      class_addMethod([self class], sel, (IMP)dynamicMethodIMP, "V@:");
		  }
		  return [super resolveInstanceMethod:sel];
		}
		void dynamicMethodIMP(id self,SEL _cmd){
		  NSLog(@"%s",__PRETTY_FUNCTION__);
		}
		+(BOOL)resolveClassMethod:(SEL)sel
		{
		  return [super resolveClassMethod:sel];
		}]

**Note：Objective-C的方法本质上是一个至少包含了两个参数（id self,SEL _cmd)的C函数。**

## Message Forwarding（消息转发）
分两步：  
1、首先运行时系统会调用- (id)forwardingTargetForSelector:(SEL)aSelector方法，如果这个方法中返回的不是nil或者self，运行时系统将把消息发送给返回的那个对象  
2、如果- (id)forwardingTargetForSelector:(SEL)aSelector返回的是nil或者self，运行时系统首先会调用- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector方法来获得方法签名，方法签名记录了方法的参数和返回值的信息，如果－methodSignatureForSelector 返回的是nil, 运行时系统会抛出unrecognized selector exception，程序到这里就结束了

	- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
	{
	  NSMethodSignature *signature =[super methodSignatureForSelector:aSelector];
	  if (!signature) {
	      //获取指定对象的方法签名
	      signature = [target methodSignatureForSelector:aSelector];
	  }
	
	  return signature;
	}
	
	- (void)forwardInvocation:(NSInvocation *)anInvocation
	{
	  //检测target是否实现来该方法
	  if ([target respondsToSelector:anInvocation.selector]) {
	      //如果实现了，在这儿将方法分发到对象中去 。可利用这个实现多重代理
	      [anInvocation invokeWithTarget:target];
	  }
	}
	
	- (id)forwardingTargetForSelector:(SEL)aSelector
	{
	  return nil;
	}

或者

	// 第一步：我们不动态添加方法，返回NO，进入第二步；
	+ (BOOL)resolveInstanceMethod:(SEL)sel
	{
	    return NO;
	}
	
	// 第二部：我们不指定备选对象响应aSelector，进入第三步；
	- (id)forwardingTargetForSelector:(SEL)aSelector
	{
	    return nil;
	}
	
	// 第三步：返回方法选择器，然后进入第四部；
	- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector 
	{
	    if ([NSStringFromSelector(aSelector) isEqualToString:@"sing"]) {
	        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
	    }
	    return [super methodSignatureForSelector:aSelector];
	}
	
	// 第四部：这步我们修改调用对象
	- (void)forwardInvocation:(NSInvocation *)anInvocation 
	{
	    // 我们改变调用对象为People
	    NewClass *newTarget = [[NewClass alloc] init];
	    [anInvocation invokeWithTarget:newTarget];
	}

流程图：

![流程图](http://upload-images.jianshu.io/upload_images/412094-06902bf5c47bb62e.png?imageMogr2/auto-orient/strip%7CimageView2/2)

### 几个特殊方法
**objc_msgSend_stret**  
如果待发送的消息要返回结构体，那么可交由此函数处理。只有当CPU的寄存器能够容纳得下消息返回类型时，这个函数才能处理此消息。若是返回值无法容纳于CPU寄存器中（比如说返回的结构体太大了），那么就由另一个函数执行派发。此时，那个函数会通过分配在栈上的某个变量来处理消息所返回的结构体。

**objc_msgSend_fpret**  
如果消息返回的是浮点数，那么可交由此函数处理。在某些架构的CPU中调用函数时，需要对“浮点数寄存器”（floating-point register）做特殊处理，也就是说，通常所用的objc_msgSend在这种情况下并不合适。这个函数是为了处理x86等架构CPU中某些令人稍觉惊讶的奇怪状况。

**objc_msgSendSuper**  
如果要给超类发消息，例如[super message:parameter]，那么就交由此函数处理。也有另外两个与objc_msgSend_stret和objc_msgSend_fpret等效的函数，用于处理发给super的相应消息。

上文说过，当找到相应的方法时，会跳转过去。之所以可以这样实现，是因为每一个Objective-C函数都可以看作是一个简单的C函数，原型如下：

	<return_type> Class_selector(id self,SEL _cmd,...)
	
以上Class及selector的命名是为了方便理解。每个类中都有一张类似于字典的方法表，而selector就相当于查找方法的key，objc_msgSend函数就是通过查这张表来实现跳转的。之所以以上原型和objc_msgSend方法长的非常相像，是为了更好使用tail-call技术来时方法的跳转更加优化。

如果某函数的最后一项操作是调用另外一个函数，那么就可以运用“tail-call”技术。
此时编译器会生成调转至另一函数所需的指令码，而不会向调用堆栈中推入新的“栈帧”。tail-call使用的条件比较苛刻，除了要求函数的最后一项操作是调用另外一个函数外，，并且要求另外一个函数不是有返回值的函数类型。tail-call对objc_msgSend非常关键，如果不这么做的话，那么每次调用Objective-C方法之前，都需要为调用objc_msgSend函数准备“栈帧”，若是不优化，还会过早地发生“栈溢出”（stack overflow）现象。

在写OC中，我们其实并不需要了解那么多底层的东西，但是我们需要知道调用一个方法之后，OC底层都发生了什么。

## 总结
* 1 一个消息包含接受者，选择子和参数。调用一个方法相当于像对象发送一条消息。
* 2 当发送消息是，动态绑定机制会帮助我们查找方法的实现并进行运行。


