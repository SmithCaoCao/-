
《AFNetWorking是如何进行缓存的？之AFImageCache & NSURLCache 详解》
===
如果你是一个正在使用由Matt Thompson’s开发的网络库 `AFNetWorking`(如果你还没有使用，那你还在等什么？)的iOS开发者，也许你一直很好奇和困惑它的缓存机制，并且想要了解如何更好地充分利用它

1. ` AFImagecache` : 继承于NSCache，AFNetworking的图片内存缓存的类。
2. ` NSURLCache`   : NSURLConnection的默认缓存机制，用于存储NSURLResponse对象：一个默认缓存在内存，并且可以通过一些配置操作可以持久缓存到磁盘的类。

AFImageCache是如何工作的？
---

`AFImageCache`属于`UIImageView+AFNetworking`的一部分,继承于`NSCache`,以URL(从`NSURLRequest`对象中获取)字符串作为key值来存储`UIImage`对象。
`AFImageCache`的定义如下：(这里我们声明了一个2M内存、100M磁盘空间的`NSURLCache`对象。)

```objective-c
@interface AFImageCache : NSCache <AFImageCache>

// singleton instantiation :

+ (id <AFImageCache>)sharedImageCache {
    static AFImageCache *_af_defaultImageCache = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _af_defaultImageCache = [[AFImageCache alloc] init];

// clears out cache on memory warning :

    [[NSNotificationCenter defaultCenter] addObserverForName:UIApplicationDidReceiveMemoryWarningNotification object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification * __unused notification) {
        [_af_defaultImageCache removeAllObjects];
    }];
});

// key from [[NSURLRequest URL] absoluteString] :

static inline NSString * AFImageCacheKeyFromURLRequest(NSURLRequest *request) {
    return [[request URL] absoluteString];
}

@implementation AFImageCache

// write to cache if proper policy on NSURLRequest :

- (UIImage *)cachedImageForRequest:(NSURLRequest *)request {
    switch ([request cachePolicy]) {
        case NSURLRequestReloadIgnoringCacheData:
        case NSURLRequestReloadIgnoringLocalAndRemoteCacheData:
            return nil;
        default:
            break;
    }

    return [self objectForKey:AFImageCacheKeyFromURLRequest(request)];
}

// read from cache :

- (void)cacheImage:(UIImage *)image
        forRequest:(NSURLRequest *)request {
    if (image && request) {
        [self setObject:image forKey:AFImageCacheKeyFromURLRequest(request)];
    }
}
```
`AFImageCache`是`NSCache`的私有实现，它把所有可访问的`UIImage`对象存入`NSCache`中，并控制着`UIImage`对象应该在何时释放，如果`UIImage`对象释放的时候你希望去做一些监听操作，你可以实现`NSCacheDelegate`的 `cache:willEvictObject` 代理方法。
   Matt Thompson’s已经谦虚的告诉我在AFNetworking2.1版本中可通过`setSharedImageCache`方法来配置`AFImageCache`，这里是 AFN2.2.1中的`UIImageView+AFNetworking`文档。

NSURLCache
---
`AFNetworking`使用了`NSURLConnection`，它利用了iOS原生的缓存机制，并且`NSURLCache`缓存了服务器返回的`NSURLRespone`对象。`NSURLCache` 的`shareCache`方法是默认开启的，你可以利用它来获取每一个`NSURLConnection`对象的URL内容。
   让人很难过的是，它的默认配置是缓存到内存而且并没有写入到磁盘。为了tame the beast(驯服野兽？不太懂)，增加可持续性，你可以在AppDelegate中简单地声明一个共享的`NSURLCache`对象，像这样：
```objective-c
NSURLCache *sharedCache = [[NSURLCache alloc] initWithMemoryCapacity:2 * 1024 * 1024
                                              diskCapacity:100 * 1024 * 1024
                                              diskPath:nil];
[NSURLCache setSharedURLCache:sharedCache];
```

设置NSURLRequest对象的缓存策略
---
 NSURLCache 将对每一个NSURLRequest对象遵守缓存策略(NSURLRequestCachePolicy)，策略如下所示：

```objective-c
- NSURLRequestUseProtocolCachePolicy                                    默认的缓存策略，对特定的URL请求使用网络协议中实现的缓存逻辑
- NSURLRequestReloadIgnoringLocalCacheData                      忽略本地缓存，重新请请求
- NSURLRequestReloadIgnoringLocalAndRemoteCacheData  忽略本地和远程缓存，重新请求
- NSURLRequestReturnCacheDataElseLoad                                有缓存则从中加载，如果没有则去请求
- NSURLRequestReturnCacheDataDontLoad                              无网络状态下不去请求，一直加载本地缓存数据无论其是否存在
- NSURLRequestReloadRevalidatingCacheData                         默从原始地址确认缓存数据的合法性之后，缓存数据才可使用，否则请求原始地址
```
用NSURLCache缓存数据到磁盘
---
###Cache-Control HTTP Header
  Cache-Controlheader和Expires header两者中必须要有一个存在于服务器返回的 HTTP response header 中，来用于客户端的缓存工作(前者优先级要高于后者)，这里面有很多地方需要注意，Cache-Control可以拥有被定义为类似max-age的参数（在更新响应之前要缓存多长时间),public/private访问或者是non－cache(不缓存响应数据)，这里对HTTP cache headers进行了很好的介绍
