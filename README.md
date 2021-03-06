# 解决接口参数爆炸问题

本篇文章聊一下传参的问题，这是基本每一个快速发展的项目都会遇到的问题，这里聊一下我自己在开发过程中想到的解决方案以及踩得坑。

如果可以前置把问题解决，会静默提升团队开发的效率。

## 一、问题描述

### （一）解决参数爆炸问题

首先先描述下我们要解决的问题：随着接口参数的不断增加，一个业务加一个参数，会导致接口的传参爆炸，且新调用接口的同学不清楚旧参数的含义。比如：

```
- (instancetype)initWithParam1:(int)param1
                        param2:(int)param2
                        param3:(int)param3
                        param4:(int)param4
                        param5:(int)param5
                        param6:(int)param6 {
    if (self = [super init]) {
        
    }
    return self;
}
```

A业务 可能只需要传 param1、param2、param4 ，而B业务可能只需要传 param1、param2、param5。

所以第一个优化的想法出来了：`参数业务隔离`，按业务拆分接口，收敛基类接口，优化的代码如下：

```
// 方法A
- (instancetype)initWithParam1:(int)param1
                        param2:(int)param2
                        param4:(int)param4{
    if (self = [super init]) {
        
    }
    return self;
}

// 方法B
- (instancetype)initWithParam1:(int)param1
                        param2:(int)param2
                        param5:(int)param5{
    if (self = [super init]) {
        
    }
    return self;
}

// 方法C
- (instancetype)initWithParam1:(int)param1
                        param2:(int)param2
                        param5:(int)param5
                        param6:(int)param6{
    if (self = [super init]) {
        
    }
    return self;
}

```

这种接口按业务隔离的方式我们实行了一段时间，刚开始还是比较顺利的，但当接入的业务越来越多，我们发现接口已经开始相互套娃了，比如 方法B调用了方法C ，当我们需要新增一个参数时，需要改动N个业务的初始化方法。

且因为自动补全的缘故，当参数越来越多时，很可能有同学写 init 方法就变成了 方法B调用方法C，然后方法C又调用了方法B，出现死循环（真实发生过）。

所以`业务隔离`并不是解决参数爆炸的好方法，还会引入套娃问题，那能不能只暴露一个接口，让业务层自己决定传什么呢？

### （二）解决套娃问题

最简单的方式是接口只暴露一个参数：NSDictionary，外界要传什么元素，就塞到 NSDictionary 就可以了。

```
- (instancetype)initWithDic:(NSMutableDictionary *)dic {
    if (self = [super init]) {
        
    }
    return self;
}
```

这种方式肯定是不行的，传参一定不要使用 NSDictionary ，这种传参的方式就是在偷懒，如果不在init方法里打断点，开发者是根本不知道接口要通过 NSDictionary 传过来什么参数，相当于把接收参数的风险转嫁到了接收方，且传参的业务方也不清楚应该按什么样的规格传参，所以这种方式一定要pass掉，至少应该用Model来进行传参。

### (三)解决隐藏参数问题

我们为传参新建一个model，这里起名`BNParamsModel`，

然后我们将要传的参数都塞到这个 Model 里：

```
@interface BNParamsModel : NSObject

@property (nonatomic, assign) int param1;
@property (nonatomic, assign) int param2;
@property (nonatomic, assign) int param3;
@property (nonatomic, assign) int param4;
@property (nonatomic, assign) int param5;
@property (nonatomic, assign) int param6;

@end
```

然后在初始化方法中传入Model参数:

```
- (instancetype)initWithParamModel:(BNParamsModel *)model {
    if (self = [super init]) {
        
    }
    return self;
}
```

这样之后添加的参数都放到Model中，就不会造成接口参数爆炸的问题，同时也让传参可读。

但这样还是会出现一个问题：不同业务添加自己业务的字段到 Model 中，但 A业务 可能只需要传 param1、param2、param4 ，而B业务可能只需要传 param1、param2、param5。

新接手这个接口的同学不知道哪些字段是必填（required）的，哪些字段是和业务相关可选（optional）的，这时就要求我们对 ParamsModel 进行一个改造。

### （四）解决漏传参数问题

我们将参数标识为 required 和 optional ,提供 _init 方法，通过代码强制约束 init 方法必须传入 required 的参数，`BNParamsModel`代码如下：

```
BNParamsModel.h

@interface BNParamsModel : NSObject

// required
@property (nonatomic, assign, readonly) int param1;
@property (nonatomic, assign, readonly) int param2;

// optional
@property (nonatomic, assign, readonly) int param3;
@property (nonatomic, assign, readonly) int param4;
@property (nonatomic, assign, readonly) int param5;
@property (nonatomic, assign, readonly) int param6;

// 业务自己构建初始化方法，不要使用 init\new 进行构建
+ (instancetype)new DISPATCH_UNAVAILABLE_MSG("Use initWith BUSINESS");
- (instancetype)init DISPATCH_UNAVAILABLE_MSG("Use initWith BUSINESS");

// 业务A
- (instancetype)initWithParam1:(int)param1
                        param2:(int)param2
                        param4:(int)param4;

// 业务B
- (instancetype)initWithParam1:(int)param1
                        param2:(int)param2
                        param5:(int)param5;

@end

```

```
BNParamsModel.m

@interface BNParamsModel ()

// required
@property (nonatomic, assign) int param1;
@property (nonatomic, assign) int param2;

// optional
@property (nonatomic, assign) int param3;
@property (nonatomic, assign) int param4;
@property (nonatomic, assign) int param5;
@property (nonatomic, assign) int param6;

@end

@implementation BNParamsModel

- (instancetype)_initWithParam1:(int)param1
                         param2:(int)param2 {
    if (self = [super init]) {
        _param1 = param1;
        _param2 = param2;
    }
    return self;
}

// 业务A
- (instancetype)initWithParam1:(int)param1
                         param2:(int)param2
                         param4:(int)param4 {
    self = [self _initWithParam1:param1 param2:param2];
    if (self) {
        _param4 = param4;
    }
    return self;
}

// 业务B
- (instancetype)initWithParam1:(int)param1
                         param2:(int)param2
                         param5:(int)param5 {
    self = [self _initWithParam1:param1 param2:param2];
    if (self) {
        _param5 = param5;
    }
    return self;
}

@end
```

## 二、解决方案

### （一）优点

这种处理方式不仅解决了参数爆炸问题，而且还通过init方式隔离业务，明确了调用业务的场景。

当有新业务接入时，就写自己的init方法，让我们知道当前有多少业务接入。

### （二）缺点

唯一的缺点就是工作量的问题，你不必为每个传参接口都构建对应的 ParamsModel ，只有当你评估该接口未来会有各种业务接入，会新增各种参数时，才建议你构建接口的 ParamsModel。
