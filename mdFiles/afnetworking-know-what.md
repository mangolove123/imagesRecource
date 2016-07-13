---
title: AFNetWorking 深度理解
---


先看一下`afnetworking`的目录结构：

![图1.0](https://github.com/wmf00032/imagesRecource/blob/master/652024-cd53ec5e18c218a4.png?raw=true)


大家都看见了网络请求其实有两种方式。一种是用`AFHTTPRequestOperationManager` ，另一种是用`AFHTTPSessionManager`。那么这两种有什么区别尼？ 

估计又的人有所不知，`AFNetworking`最初的版本使用的就是`AFHTTPRequestOperationManager` 以就是自己自定义`NSOperation`的封装实现的。`AFHTTPSessionManager`是在`2.0`以后才引入进来的。以就是说在`2.0`之前，都是使用`AFHTTPRequestOperationManager`。

`AFHTTPSessionManager` 继承自 `AFURLSessionManager`，而`AFURLSessionManager`主要是使用系统提供的 `NSURLSession`和`NSURLSessionTask`进行网络操作的。我们看一下官方文档对这两个类的描述：

```
NS_CLASS_AVAILABLE(NSURLSESSION_AVAILABLE, 7_0)
@interface NSURLSession : NSObject

/*
 * NSURLSessionTask - a cancelable object that refers to the lifetime
 * of processing a given request.
 */
NS_CLASS_AVAILABLE(NSURLSESSION_AVAILABLE, 7_0)
@interface NSURLSessionTask : NSObject <NSCopying>

```
是的，你没有看错，`NSURLSession`和`NSURLSessionTask`是`iOS7`以后才出现的，所以如果你想要适配到`iOS6`，那么请你乖乖的使用前者 进行网络请求。这就解释了为什么作者会把两种请求方式都放在这儿。






下面分两部分进行网络`AFNetworking`网络请求的分析：
`AFHTTPRequestOperationManager` 部分的网络请求原理：

`AFHTTPRequestOperationManager`的`init`方法是这样的：

```
- (instancetype)initWithBaseURL:(NSURL *)url {
    self = [super init];
    if (!self) {
        return nil;
    }

    // Ensure terminal slash for baseURL path, so that NSURL +URLWithString:relativeToURL: works as expected
    if ([[url path] length] > 0 && ![[url absoluteString] hasSuffix:@"/"]) {
        url = [url URLByAppendingPathComponent:@""];
    }

    self.baseURL = url;

    self.requestSerializer = [AFHTTPRequestSerializer serializer];
    self.responseSerializer = [AFJSONResponseSerializer serializer];

    self.securityPolicy = [AFSecurityPolicy defaultPolicy];

    self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];

    self.operationQueue = [[NSOperationQueue alloc] init];

    self.shouldUseCredentialStorage = YES;

    return self;
}
```

其中默认的请求方式和解析方式被设置了默认值：

```    
self.requestSerializer = [AFHTTPRequestSerializer serializer];
self.responseSerializer = [AFJSONResponseSerializer serializer];
```
**当然用户可以修改这两个参数，指定自己的请求方式和解析方式。**


下面以`GET`请求为例来说。
当用户发起一个`GET`请求，下面的方法会被掉用：

```
- (AFHTTPRequestOperation *)GET:(NSString *)URLString
                     parameters:(id)parameters
                        success:(void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                        failure:(void (^)(AFHTTPRequestOperation *operation, NSError *error))failure
{
    AFHTTPRequestOperation *operation = [self HTTPRequestOperationWithHTTPMethod:@"GET" URLString:URLString parameters:parameters success:success failure:failure];

    [self.operationQueue addOperation:operation];

    return operation;
}
```

其中`[self.operationQueue addOperation:operation];`就是将当前的任务放进操作队列。
关键我们看看`AFHTTPRequestOperation` 里面都做了什么。

`AFHTTPRequestOperation` 是继承自`AFURLConnectionOperation`，`AFURLConnectionOperation`实现了各种代理：

`@interface AFURLConnectionOperation : NSOperation <NSURLConnectionDelegate, NSURLConnectionDataDelegate, NSSecureCoding, NSCopying>`


我们知道当一个`operation`任务被启动的时候`start`方法就会被调用：

```
- (void)start {
    [self.lock lock];
    if ([self isCancelled]) {
        [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    } else if ([self isReady]) {
        self.state = AFOperationExecutingState;

        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    }
    [self.lock unlock];
}

- (void)operationDidStart {
    [self.lock lock];
    if (![self isCancelled]) {
        self.connection = [[NSURLConnection alloc] initWithRequest:self.request delegate:self startImmediately:NO];

        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        for (NSString *runLoopMode in self.runLoopModes) {
            [self.connection scheduleInRunLoop:runLoop forMode:runLoopMode];
            [self.outputStream scheduleInRunLoop:runLoop forMode:runLoopMode];
        }

        [self.outputStream open];
        [self.connection start];
    }
    [self.lock unlock];

    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingOperationDidStartNotification object:self];
    });
}
```

我们分析一下一上代码块，当`AFURLConnectionOperation`任务被正常启动的时候，下面的方法会被调用：

```
[self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];

operationDidStart方法会在[[self class] networkRequestThread]返回的线程中被调用。我们看看这是一个什么样的线程：
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];

        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}

+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });

    return _networkRequestThread;
}
```

是的这个新的小线程命名为**"AFNetworking"**, 它在床见得时候就启动了一个人`runloop`事件循环，并添加了一个`NSMachPort` 空端口，改`NSMachPort` 仅仅只是一个空的端口其目的是用来维护`runloop`的执行不被退出。

`operationDidStart` 方法中的：

```
        self.connection = [[NSURLConnection alloc] initWithRequest:self.request delegate:self startImmediately:NO];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        for (NSString *runLoopMode in self.runLoopModes) {
            [self.connection scheduleInRunLoop:runLoop forMode:runLoopMode];
            [self.outputStream scheduleInRunLoop:runLoop forMode:runLoopMode];
        }

        [self.outputStream open];
        [self.connection start];
```
        
说明`self.connection`和`self.outputStream`周期性任务被绑定在当期的`runloop`的`self.runLoopModes` 模式中。（`self.runLoopModes`在初始化的时候被赋值为`[NSSet setWithObject:NSRunLoopCommonModes]`）


行了，那么当网络请求数据到达的时候，数据是如何被接收到的尼，我想这一点才是大家最关心的。
网络请求是一个异步的过程，当网络请求数据流到达的时候，`runloop`监听到改事件源，`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`方法会被调用。之后`CFNetwork`的`_NSURLConnectionDidReceiveData(_CFURLConnection*, __CFData const*, long, void const*)`放会被调用获取到网络数据。与此同时`NSURLConnectionInternal`的`_withActiveConnectionAndDelegate`方法会被调用，本地代理被激活。`AFURLConnectionOperation`中的代理方法：

```
- (void)connection:(NSURLConnection __unused *)connection
    didReceiveData:(NSData *)data
{
    NSUInteger length = [data length];
    while (YES) {
        NSInteger totalNumberOfBytesWritten = 0;
        if ([self.outputStream hasSpaceAvailable]) {
            const uint8_t *dataBuffer = (uint8_t *)[data bytes];

            NSInteger numberOfBytesWritten = 0;
            while (totalNumberOfBytesWritten < (NSInteger)length) {
                numberOfBytesWritten = [self.outputStream write:&dataBuffer[(NSUInteger)totalNumberOfBytesWritten] maxLength:(length - (NSUInteger)totalNumberOfBytesWritten)];
                if (numberOfBytesWritten == -1) {
                    break;
                }

                totalNumberOfBytesWritten += numberOfBytesWritten;
            }

            break;
        } else {
            [self.connection cancel];
            if (self.outputStream.streamError) {
                [self performSelector:@selector(connection:didFailWithError:) withObject:self.connection withObject:self.outputStream.streamError];
            }
            return;
        }
    }

    dispatch_async(dispatch_get_main_queue(), ^{
        self.totalBytesRead += (long long)length;

        if (self.downloadProgress) {
            self.downloadProgress(length, self.totalBytesRead, self.response.expectedContentLength);
        }
    });
}
```

被调用，通过以上代码中的：

```
           NSInteger numberOfBytesWritten = 0;
            while (totalNumberOfBytesWritten < (NSInteger)length) {
                numberOfBytesWritten = [self.outputStream write:&dataBuffer[(NSUInteger)totalNumberOfBytesWritten] maxLength:(length - (NSUInteger)totalNumberOfBytesWritten)];
                if (numberOfBytesWritten == -1) {
                    break;
                }

                totalNumberOfBytesWritten += numberOfBytesWritten;
            }
            
```
            
网络数据流被写入缓存。数据被写入缓存完成后，代理方法：

```
- (void)connectionDidFinishLoading:(NSURLConnection __unused *)connection {
    self.responseData = [self.outputStream propertyForKey:NSStreamDataWrittenToMemoryStreamKey];

    [self.outputStream close];
    if (self.responseData) {
       self.outputStream = nil;
    }

    self.connection = nil;

    [self finish];
}
```

会被调用后。我们追踪`[self finish];` 看看它里面的实现：

```
- (void)finish {
    [self.lock lock];
    self.state = AFOperationFinishedState;
    [self.lock unlock];

    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingOperationDidFinishNotification object:self];
    });
}
```
`self.state = AFOperationFinishedState `这句代码是重点，它标示了该请求任务已经结束。而这句赋值还做了一个KVO的操作，如下代码：

```
- (void)setState:(AFOperationState)state {
    if (!AFStateTransitionIsValid(self.state, state, [self isCancelled])) {
        return;
    }

    [self.lock lock];
    NSString *oldStateKey = AFKeyPathFromOperationState(self.state);
    NSString *newStateKey = AFKeyPathFromOperationState(state);

    [self willChangeValueForKey:newStateKey];
    [self willChangeValueForKey:oldStateKey];
    _state = state;
    [self didChangeValueForKey:oldStateKey];
    [self didChangeValueForKey:newStateKey];
    [self.lock unlock];
}
```

`NSOperationInternal`中的`_observeValueForKeyPath:ofObject:changeKind:oldValue:newValue:indexes:context:`方法会监听`state`状态的改变。然后回归到主线程，AFURLConnectionOperation中的`setCompletionBlock`方法被回调：

```
- (void)setCompletionBlock:(void (^)(void))block {
    [self.lock lock];
    if (!block) {
        [super setCompletionBlock:nil];
    } else {
        __weak __typeof(self)weakSelf = self;
        [super setCompletionBlock:^ {
            __strong __typeof(weakSelf)strongSelf = weakSelf;

#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"
            dispatch_group_t group = strongSelf.completionGroup ?: url_request_operation_completion_group();
            dispatch_queue_t queue = strongSelf.completionQueue ?: dispatch_get_main_queue();
#pragma clang diagnostic pop

            dispatch_group_async(group, queue, ^{
                block();
            });

            dispatch_group_notify(group, url_request_operation_completion_queue(), ^{
                [strongSelf setCompletionBlock:nil];
            });
        }];
    }
    [self.lock unlock];
}
```

关键看这句代码：

```
dispatch_group_async(group, queue, ^{
                block();
            });

```
`block()`一调用就调用到了`AFHTTPRequestOperation`的方法：

```
#pragma mark - AFHTTPRequestOperation
- (void)setCompletionBlockWithSuccess:(void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                              failure:(void (^)(AFHTTPRequestOperation *operation, NSError *error))failure
{
    // completionBlock is manually nilled out in AFURLConnectionOperation to break the retain cycle.
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-retain-cycles"
#pragma clang diagnostic ignored "-Wgnu"
    self.completionBlock = ^{
        if (self.completionGroup) {
            dispatch_group_enter(self.completionGroup);
        }

        dispatch_async(http_request_operation_processing_queue(), ^{
            if (self.error) {
                if (failure) {
                    dispatch_group_async(self.completionGroup ?: http_request_operation_completion_group(), self.completionQueue ?: dispatch_get_main_queue(), ^{
                        failure(self, self.error);
                    });
                }
            } else {
                id responseObject = self.responseObject;
                if (self.error) {
                    if (failure) {
                        dispatch_group_async(self.completionGroup ?: http_request_operation_completion_group(), self.completionQueue ?: dispatch_get_main_queue(), ^{
                            failure(self, self.error);
                        });
                    }
                } else {
                    if (success) {
                        dispatch_group_async(self.completionGroup ?: http_request_operation_completion_group(), self.completionQueue ?: dispatch_get_main_queue(), ^{
                            success(self, responseObject);
                        });
                    }
                }
            }

            if (self.completionGroup) {
                dispatch_group_leave(self.completionGroup);
            }
        });
    };
#pragma clang diagnostic pop
}

```

`self.completionBlock`中的内容被执行，这里面最关键的一句id `responseObject = self.responseObject` 获得了解析数据，我们看看`self.responseObject`的实现：

```
- (id)responseObject {
    [self.lock lock];
    if (!_responseObject && [self isFinished] && !self.error) {
        NSError *error = nil;
        self.responseObject = [self.responseSerializer responseObjectForResponse:self.response data:self.responseData error:&error];
        if (error) {
            self.responseSerializationError = error;
        }
    }
    [self.lock unlock];

    return _responseObject;
}
```

**我靠！春天来了**，这句代码：

```
self.responseObject = [self.responseSerializer responseObjectForResponse:self.response data:self.responseData error:&error];
```

将`stream`中的数据解析成了我们想要数据比如`json、xml、plist`等等。然后`AFHTTPRequestOperation`方法`setCompletionBlockWithSuccess`中代码`success(self, responseObject)`一回调。好了，现在就不用说了，我们通常就是在这个`success` `block`做我们自己的处理的。




下面我们看一下`AFHTTPSessionManager` 网络请求是怎么样进行的
苹果已经对`NSURLSessionDataTask`做了高度的封装，类似于`AFURLConnectionOperation`这样的复杂封装已经看不见了。但是原理跟`AFURLConnectionOperation`的差不多这里就不在赘述了。

到一个网络数据流到达的时候，`NSURLSession的URLSession:dataTask:didReceiveData:`方法就会被激活。`AFURLSessionManager`实现了`NSURLSession`的代理，于是乎`AFURLSessionManager`中的代理方法就会被调用：

```
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:dataTask];
    [delegate URLSession:session dataTask:dataTask didReceiveData:data];

    if (self.dataTaskDidReceiveData) {
        self.dataTaskDidReceiveData(session, dataTask, data);
    }
}
```

`[delegate URLSession:session dataTask:dataTask didReceiveData:data]`里面的实现四这样的：

```
- (void)URLSession:(__unused NSURLSession *)session
          dataTask:(__unused NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
    [self.mutableData appendData:data];
}
```

这样网络数据就被写入了`self.mutableData`。
当数据获取完成之后`URLSession:task:didCompleteWithError:`方法就会被调用：

```
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];

    // delegate may be nil when completing a task in the background
    if (delegate) {
        [delegate URLSession:session task:task didCompleteWithError:error];

        [self removeDelegateForTask:task];
    }

    if (self.taskDidComplete) {
        self.taskDidComplete(session, task, error);
    }

}
```

我们看看 `[delegate URLSession:session task:task didCompleteWithError:error]` 方法中的实现：

```
- (void)URLSession:(__unused NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"
    __strong AFURLSessionManager *manager = self.manager;

    __block id responseObject = nil;

    __block NSMutableDictionary *userInfo = [NSMutableDictionary dictionary];
    userInfo[AFNetworkingTaskDidCompleteResponseSerializerKey] = manager.responseSerializer;

    //Performance Improvement from #2672
    NSData *data = nil;
    if (self.mutableData) {
        data = [self.mutableData copy];
        //We no longer need the reference, so nil it out to gain back some memory.
        self.mutableData = nil;
    }

    if (self.downloadFileURL) {
        userInfo[AFNetworkingTaskDidCompleteAssetPathKey] = self.downloadFileURL;
    } else if (data) {
        userInfo[AFNetworkingTaskDidCompleteResponseDataKey] = data;
    }

    if (error) {
        userInfo[AFNetworkingTaskDidCompleteErrorKey] = error;

        dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
            if (self.completionHandler) {
                self.completionHandler(task.response, responseObject, error);
            }

            dispatch_async(dispatch_get_main_queue(), ^{
                [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
            });
        });
    } else {
        dispatch_async(url_session_manager_processing_queue(), ^{
            NSError *serializationError = nil;
            responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError];

            if (self.downloadFileURL) {
                responseObject = self.downloadFileURL;
            }

            if (responseObject) {
                userInfo[AFNetworkingTaskDidCompleteSerializedResponseKey] = responseObject;
            }

            if (serializationError) {
                userInfo[AFNetworkingTaskDidCompleteErrorKey] = serializationError;
            }

            dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
                if (self.completionHandler) {
                    self.completionHandler(task.response, responseObject, serializationError);
                }

                dispatch_async(dispatch_get_main_queue(), ^{
                    [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
                });
            });
        });
    }
#pragma clang diagnostic pop
}

```

以上代码中 `data = [self.mutableData copy]` 可知网络数据流被`copy`到了 `data`之中，关键代码：

` responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError] `
网络数据被解析成了我们最终获取到的数据。`self.completionHandler `一执行如下代码块：

```
if (self.completionHandler) {
                    self.completionHandler(task.response, responseObject, serializationError);
                }
                
```

就回到了`AFHTTPSessionManager`的方法：

```
- (NSURLSessionDataTask *)dataTaskWithHTTPMethod:(NSString *)method
                                       URLString:(NSString *)URLString
                                      parameters:(id)parameters
                                         success:(void (^)(NSURLSessionDataTask *, id))success
                                         failure:(void (^)(NSURLSessionDataTask *, NSError *))failure
{
    NSError *serializationError = nil;
    NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:&serializationError];
    if (serializationError) {
        if (failure) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
#pragma clang diagnostic pop
        }

        return nil;
    }

    __block NSURLSessionDataTask *dataTask = nil;
    dataTask = [self dataTaskWithRequest:request completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(dataTask, error);
            }
        } else {
            if (success) {
                success(dataTask, responseObject);
            }
        }
    }];

    return dataTask;
}
```

这`block`就会被调用：

```
    __block NSURLSessionDataTask *dataTask = nil;
    dataTask = [self dataTaskWithRequest:request completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(dataTask, error);
            }
        } else {
            if (success) {
                success(dataTask, responseObject);
            }
        }
    }];
```

这里的`success`和 `failure` 就是我们经常看见的回调方法。

***欢迎交流，你对afnetworking的理解！***