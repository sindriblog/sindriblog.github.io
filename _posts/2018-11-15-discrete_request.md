---
title: 分析实现-离散请求
date: 2018-11-15 08:00
tags:
- 分析实现
---

网络层作为`App`架构中至关重要的中间件之一，承担着业务封装和核心层网络请求交互的职责。讨论请求中间件实现方案的意义在于中间件要如何设计以便减少对业务对接的影响；明晰请求流程中的职责以便写出更合理的代码等。因此在讲如何去设计请求中间件时，主要考虑三个问题：

- 业务以什么方式发起请求
- 请求数据如何交付业务层
- 如何实现通用的请求接口

## 以什么方式发起请求
根据暴露给业务层请求`API`的不同，可以分为`集约式请求`和`离散型请求`两类。`集约式请求`对外只提供一个类用于接收包括请求地址、请求参数在内的数据信息，以及回调处理（通常使用`block`）。而`离散型请求`对外提供通用的扩展接口完成请求

### 集约式请求
考虑到`AFNetworking`基本成为了`iOS`的请求标准，以传统的集约式请求代码为例：

    /// 请求地址和参数组装
    NSString *domain = [SLNetworkEnvironment currentDomain];
    NSString *url = [domain stringByAppendingPathComponent: @"getInterviewers"];
    NSDictionary *params = @{
        @"page": @1,
        @"pageCount": @20,
        @"filterRule": @"work-years >= 3"
    };
    
    /// 构建新的请求对象发起请求
    AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
    [manager POST: url parameters: params success: ^(NSURLSessionDataTask *task, id responseObject) {
        /// 请求成功处理
        if ([responseObject isKindOfClass: [NSArray class]]) {
            NSArray *result = [responseObject bk_map: ^id(id obj) {
                return [[SLResponse alloc] initWithJSON: obj];
            }];
            [self reloadDataWithResponses: result];
        } else {
            SLLog(@"Invalid response object: %@", responseObject);
        }
    } failure: ^(NSURLSessionDataTask *task, NSError *error) {
        /// 请求失败处理
        SLLog(@"Error: %@ in requesting %@", error, task.currentRequest.URL);
    }];

    /// 取消存在的请求
    [self.currentRequestManager invalidateSessionCancelingTasks: YES];
    self.currentRequestManager = manager;

这样的请求代码存在这些问题：

1. 请求环境配置、参数构建、请求任务控制等业务无关代码
2. 请求逻辑和回调逻辑在同一处违背了单一原则
3. `block`回调潜在的引用问题

在业务封装的层面上，应该只关心`何时发起请求`和`展示请求结果`。设计上，请求中间件应当只暴露必要的参数`property`，隐藏请求过程和返回数据的处理

### 离散型请求
和集约式请求不同，对于每一个请求`API`都会有一个`manager`来管理。在使用`manager`的时候只需要创建实例，执行一个类似`load`的方法，`manager`会自动控制请求的发起和处理：
    
    - (void)viewDidLoad {
        [super viewDidLoad];
        self.getInterviewerApiManager = [SLGetInterviewerApiManager new];
        [self.getInterviewerApiManager addDelegate: self];
        [self.getInterviewerApiManager refreshData];
    }
    
集约式请求和离散型请求最终的实现方案并不是互斥的，从底层请求的具体行为来看，最终都有统一执行的步骤：`域名拼凑`、`请求发起`、`结果处理`等。因此从设计上来说，使用基类来统一这些行为，再通过派生生成针对不同请求`API`的子类，以便获得具体请求的灵活性：
    
    @protocol SLBaseApiManagerDelegate
    
    - (void)managerWillLoadData: (SLBaseApiManager *)manager;
    - (void)managerDidLoadData: (SLBaseApiManager *)manager;

    @end
    
    @interface SLBaseApiManager : NSObject
    
    @property (nonatomic, readonly) NSArray<id<SLBaseApiManagerDelegate>) *delegates;
    
    - (void)loadWithParams: (NSDictionary *)params;
    - (void)addDelegate: (id<SLBaseApiManagerDelegate>)delegate;
    - (void)removeDelegate: (id<SLBaseApiManagerDelegate>)delegate;
    
    @end
    
    @interface SLBaseListApiManager : SLBaseApiManager 
    
    @property (nonatomic, readonly, assign) BOOL hasMore;
    @property (nonatomic, readonly, copy) NSArray *dataList;
    
    - (void)refreshData;
    - (void)loadMoreData;
    
    @end

离散型请求的一个特点是，将相同的请求逻辑抽离出来，统一行为接口。除了请求行为之外的行为，包括请求数据解析、重试控制、请求是否互斥等行为，每一个请求`API`都有单独的`manager`进行定制，灵活性更强。另外通过`delegate`统一回调行为，减少`debug`难度，避免了`block`方式潜在的引用问题等

## 请求数据如何交付
在一次完整的`fetch`数据过程中，数据可以分为四种形态：

- 服务端直接返回的二进制形态，称为`Data`
- 以`AFN`等工具拉取的数据，一般是`JSON`
- 被持久化或非短暂持有的形态，一般从`JSON`转换而来，称作`Entity`
- 展示在屏幕上的文本形态，大概率需要再加工，称作`Text`

这四种数据形态的流动结构如下：

        Server                  AFN                   controller                view
    -------------           -------------           -------------           -------------
    |           |           |           |           |           |  convert  |           |
    |   Data    |   --->    |   JSON    |   --->    |   Entity  |   --->    |    Text   |    
    |           |           |           |           |           |           |           |
    -------------           -------------           -------------           -------------
    
普通情况下，第三方请求库会以`JSON`的形态交付数据给业务方。考虑到客户端与服务端的命名规范、以及可能存在的变更，多数情况下客户端会对`JSON`数据加工成具体的`Entity`数据实体，然后使用容器类保存。从上图的四种数据形态来说，如果中间件必须选择其中一种形态交付给业务层，`Entity`应该是最合理的交付数据形态，原因有三：

1. 如果交付的是`JSON`，业务封装必须完成`JSON -> Entity`的转换，多数时候请求发起的业务在`C`层中，而这些逻辑总是造成`Fat Controller`的原因
2. 在`Entity -> Text`涉及到了具体的上层业务，请求中间件不应该向上干涉。在`JSON -> Entity`的转换过程中，`Entity`已经组装了业务封装最需要的数据内容

另一个有趣的问题是`Entity`描述的是数据流动的阶段状态，而非具体数据类型。打个比方，`Entity`不一定非得是类对象实例，只要`Entity`遵守业务封装的读取规范，可以是`instance`也可以是`collection`，比如一个面试者`Entity`只要能提供`姓名`和`工作年限`这两个关键数据即可：

    /// 抽象模型
    @interface SLInterviewer : NSObject
    
    @property (nonatomic, copy) NSString *name;
    @property (nonatomic, assign) CGFloat workYears; 
    
    @end
    
    SLInterviewer *interviewer = entity;
    NSLog(@"The interviewer name: %@ and work-years: %g", interviewer.name, interviewer.workYears);
    
    /// 键值约定
    extern NSString *SLInterviewerNameKey;
    extern NSString *SLInterviewerWorkYearsKey;
    
    NSDictionary *interviewer = entity;
    NSLog(@"The interviewer name: %@ and work-years: %@", interviewer[SLInterviewerNameKey], interviewer[SLInterviewerWorkYearsKey]);

如果让集约式请求的中间件交付`Entity`数据，`JSON -> Entity`的形态转换可能会导致请求中间件涉及到具体的业务逻辑中，因此在实现上需要提供一个`parser`来完成这一过程：

    @protocol EntityParser
    
    - (id)parseJSON: (id)JSON;
    
    @end
    
    @interface SLIntensiveRequest : NSObject
    
    @property (nonatomic, strong) id<EntityParser> parser;
    
    - (void)GET: (NSString *)url params: (id)params success: (SLSuccess)success failure: (SLFailure)failure;
    
    @end
    
而相较之下，离散型请求中`BaseManager`承担了统一的请求行为，派生的`manager`完全可以直接将转换的逻辑直接封装起来，无需额外的`Parser`，唯一需要考虑的是`Entity`的具体实体对象是否需要抽象模型来表达：

    @implementation SLGetInterviewerApiManager

    /// 抽象模型
    - (id)entityFromJSON: (id)json {
        if ([json isKindOfClass: [NSDictionary class]]) {
            return [SLInterviewer interviewerWithJSON: json];
        } else {
            return nil;
        }
    }
    
    - (void)didLoadData {
        self.dataList = self.response.safeMap(^id(id item) {
            return [self entityFromJSON: item];
        }).safeMap(^id(id interviewer) {
            return [SLInterviewerInfo infoWithInterviewer: interviewer];
        });
        
        if ([_delegate respondsToSelector: @selector(managerDidLoadData:)]) {
            [_delegate managerDidLoadData: self];
        }
    }
    
    /// 键值约定
    - (id)entityFromJSON: (id)json keyMap: (NSDictionary *)keyMap {
        if ([json isKindOfClass: [NSDictionary class]]) {
            NSDictionary *dict = json;
            NSMutableDictionary *entity = @{}.mutableCopy;
            for (NSString *key in keyMap) {
                NSString *entityKey = keyMap[key];
                entity[entityKey] = dict[key];
            }
            return entity.copy;
        } else {
            return nil;
        }
    }
    
    @end

甚至再进一步，`manager`可以同时交付`Text`和`Entity`这两种数据形态，使用`parser`可以对`C`层完成隐藏数据的转换过程：

    @protocol TextParser
    
    - (id)parseEntity: (id)entity;
    
    @end
    
    @interface SLInterviewerTextContent : NSObject
    
    @property (nonatomic, readonly) NSString *name;
    @property (nonatomic, readonly) NSString *workYear;
    @property (nonatomic, readonly) SLInterviewer *interviewer;
    
    - (instancetype)initWithInterviewer: (SLInterviewer *)interviewer;
    
    @end
    
    @implementation SLInterviewerTextParser
    
    - (id)parseEntity: (SLInterviewer *)entity {
        return [[SLInterviewerTextContent alloc] initWithInterviewer: entity];
    }
    
    @end
    

## 通用的请求接口
> 是否需要统一接口的请求封装层

在`App`中的请求分为三类：`GET`、`POST`和`UPLOAD`，在不考虑进行封装的情况下，核心层的请求接口至少需要三种不同的接口来对应这三种请求类型。此外还要考虑核心层的请求接口一旦发生变动（例如`AFN`在更新至`3.0`的时候修改了请求接口），因此对业务请求发起方来说，存在一个封装的请求中间层可以有效的抵御请求接口改动的风险，以及有效的减少代码量。上文可以看到对业务层暴露的中间件`manager`的作用是对请求的行为进行统一，但并不干预请求的细节，因此`manager`也能被当做是一个请求发起方，那么在其下层需要有暴露统一接口的请求封装层：

                -------------
    中间件       |  Manager  |
                -------------
                      ↓
                      ↓
                -------------
    请求层       |  Request  |
                -------------
                      ↓
                      ↓
                -------------
    核心请求     |  CoreNet  |
                -------------

封装请求层的问题在于如何只暴露一个接口来适应多种情况类型，一个方法是将请求内容抽象成一系列的接口协议，`Request`层根据接口返回参数调度具体的请求接口：

    /// 协议接口层
    enum {
        SLRequestMethodGet,
        SLRequestMethodPost,
        SLRequestMethodUpload  
    };

    @protocol RequestEntity
    
    - (int)requestMethod;           /// 请求类型
    - (NSString *)urlPath;          /// 提供域名中的path段，以便组装：xxxxx/urlPath
    - (NSDictionary *)parameters;   /// 参数
    
    @end
    
    extern NSString *SLRequestParamPageKey;
    extern NSString *SLRequestParamPageCountKey;
    @interface RequestListEntity : NSObject<RequestEntity>
    
    @property (nonatomic, assign) NSUInteger page;
    @property (nonatomic, assign) NSUInteger pageCount;
    
    @end
    
    /// 请求层
    typedef void(^SLRequestComplete)(id response, NSError *error);
    
    @interface SLRequestEngine
    
    + (instancetype)engine;
    - (void)sendRequest: (id<RequestEntity>)request complete: (SLRequestComplete)complete;
    
    @end
    
    @implementation SLRequestEngine
    
    - (void)sendRequest: (id<RequestEntity>)request complete: (SLRequestComplete)complete {
        if (!request || !complete) {
            return;
        }
    
        if (request.requestMethod == SLRequestMethodGet) {
            [self get: request complete: complete];
        } else if (request.requestMethod == SLRequestMethodPost) {
            [self post: request complete: complete];
        } else if (request.requestMethod == SLRequestMethodUpload) {
            [self upload: request complete: complete];
        }
    }
    
    @end

这样一来，当有新的请求`API`时，创建对应的`RequestEntity`和`Manager`类来处理请求。对于业务上层来说，整个请求过程更像是一个异步的`fetch`流程，一个单独的`manager`负责加载数据并在加载完成时回调。`Manager`也不用了解具体是什么请求，只需要简单的配置参数即可，`Manager`的设计如下：

    @interface WSBaseApiManager : NSObject
    
    @property (nonatomic, readonly, strong) id data;
    @property (nonatomic, readonly, strong) NSError *error;   /// 请求失败时不为空
    @property (nonatomic, weak) id<WSBaseApiManagerDelegate> delegate;
    
    @end
    
    @interface WSBaseListApiManager : NSObject
    
    @property (nonatomic, assign) BOOL hasMore;
    @property (nonatomic, readonly, copy) NSArray *dataList;
    
    @end

    @interface SLGetInterviewerRequest: RequestListEntity
    @end
    
    @interface SLGetInterviewerManager : WSBaseListApiManager
    @end
    
    @implementation SLGetInterviewerManager
    
    - (void)loadWithParams: (NSDictionary *)params {
        SLGetInterviewerRequest *request = [SLGetInterviewerRequest new];
        request.page = [params[SLRequestParamPageKey] unsignedIntegerValue];
        request.pageCount = [params[SLRequestParamPageCountKey] unsignedIntegerValue];
        [[SLRequestEngine engine] sendRequest: request complete: ^(id response, NSError *error){
            /// do something when request complete
        }];
    }
    
    @end
    
最终请求结构：

                                        -------------
    业务层                               |  Client  |
                                        -------------
                                              ↓
                                              ↓
                                        -------------
    中间件                               |  Manager  |
                                        -------------
                                              ↓
                                              ↓
                                        -------------
                                        |  Request  |
                                        -------------
                                              ↓
                                              ↓
    请求层                   -----------------------------------
                            ↓                 ↓               ↓
                            ↓                 ↓               ↓
                       -------------    -------------   -------------                
                       |    GET    |    |   POST    |   |   Upload  |                
                       -------------    -------------   -------------    
                            ↓                 ↓               ↓
                            ↓                 ↓               ↓
                       ---------------------------------------------
    核心请求            |                   CoreNet                 |
                       ---------------------------------------------



![关注我的公众号获取更新信息](https://upload-images.jianshu.io/upload_images/783864-5f15782c42a970c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

