#iOS设置自定义Cookie
 当访问一个网页的时候，NSURLRequest会默认帮你主动记录当前站点的设置的Cookie,然后访问之后就会自动写入磁盘，iOS是把cookie信息放在的NSHTTPCookieStorage容器中管理，当下一次请求的时候，NSURLRequest会从内存中拿出上次保存下来的Cookie继续去请求。
##Cookie
Cookie有服务器端生成，响应后发送给客户端，每一个Cookie都是一个NSHTTPCookie类的实例。
##NSHTTPCookieStorage
NSHTTPCookieStorage实现了一个管理cookie的单例对象(只有一个实例)，每个Cookie都是NSHTTPCookie类的实例。同一个域下的Cookie在`应用内`是共享的。
##HTTPHeader
HTTPHeader中包含HTTP请求和响应的参数,header中定义了传输数据的各种特性。header以属性名开始，属性值之间用`;`间隔 。Cookie属于HTTPHeader的一部分。
##iOS中Cookie的读取和写入
Cookie一般是通过http的Response中传过来，作为Response的HTTP Header中，系统默认会将cookie写入磁盘保存。那么如何获取Cookie,并且写入自定义Cookie呢？

```
需要先手动获取当前NSHTTPCookieStorage中的所有cookie，然后将cookie放到NSURLRequest请求头中
```

###1.获取已有的Cookie，然后添加新的Cookie。
```
- (NSArray *)fetchNewCookies {
    NSMutableArray *cookies = [[[NSHTTPCookieStorage sharedHTTPCookieStorage] cookiesForURL:[NSURL URLWithString:@"https://backend.gmei.com"]] mutableCopy];
    NSHTTPCookie *accessCookie = [self fetchAccessTokenCookie];
    //添加新Cookie最好多加一步去重
    [cookies addObject:accessCookie];
    return [cookies copy];
}

- (NSHTTPCookie *)fetchAccessTokenCookie {
     NSMutableDictionary *properties = [NSMutableDictionary dictionary];
    [properties setObject:@"_gm_token" forKey:NSHTTPCookieName];
    [properties setObject:accessToken forKey:NSHTTPCookieValue];
    [properties setObject:@"backend.gmei.com" forKey:NSHTTPCookieDomain];
    [properties setObject:@"/" forKey:NSHTTPCookiePath];
    NSHTTPCookie *accessCookie = [[NSHTTPCookie alloc] initWithProperties:properties];
    return accessCookie;
}
```
iOS中NSHTTPCookie有一系列的键值对作为Cookie的属性
HTTPCookie属性中的key有:
```
NSString *NSHTTPCookieComment;
NSString *NSHTTPCookieCommentURL;
NSString *NSHTTPCookieDiscard;
NSString *NSHTTPCookieDomain;
NSString *NSHTTPCookieExpires;
NSString *NSHTTPCookieMaximumAge;
NSString *NSHTTPCookieName;
NSString *NSHTTPCookieOriginURL;
NSString *NSHTTPCookiePath;
NSString *NSHTTPCookiePort;
NSString *NSHTTPCookieSecure;
NSString *NSHTTPCookieValue;
NSString *NSHTTPCookieVersion;
```
下面简单解释下每个key的含义。

* NSHTTPCookieComment
包含cookie的注释的NSString对象,注释项,用户说明该 Cookie 有何用途,对Vesion1以及以后的版本有效，是可选属性
  
* NSHTTPCookieCommentURL
包含cookie的CommentURL的NSURL/NSString对象,对Vesion1以及以后的版本有效，是可选属性。

* NSHTTPCookieDiscard
一个NSString对象，用于标识在会话结束时是否应该丢弃改Cookie，是可选属性。String的值必须是`True`或者`false`。默认值是`false`，除非该cookie是version1或更高版本，否则不指定NSHTTPCookieMaximumAge的值，在这种情况下，它被假定为`TRUE`

* NSHTTPCookieDomain
包含cookie的域的NSString对象,如果缺少此cookie属性，则会根据NSHTTPCookieOriginURL的值推断该域。如果未指定NSHTTPCookieOriginURL的值，则必须为NSHTTPCookieDomain指定值。

* NSHTTPCookieExpires
指定了Cookie过期日期的NSDate对象或NSString对象,此cookie属性仅用于版本0的Cookie,是可选属性
  
* NSHTTPCookieMaximumAge
个包含一个整数值的NSString对象，表示cookie最大失效时间。仅适用于版本1 Cookie及更高版本。默认值为“0”。此cookie属性是可选的。

* NSHTTPCookieName
包含cookie名称的NSString对象。此cookie属性是必选的。

* NSHTTPCookieValue
包含cookie的值的NSString对象。此cookie属性是必选的。

* NSHTTPCookieOriginURL
包含设置此cookie的URL的NSURL或NSString对象。`如果您不提供NSHTTPCookieOriginURL的值，则必须为NSHTTPCookieDomain提供一个值。`

* NSHTTPCookiePath
该Cookie是在当前的哪个路径下生成的,`/`表示在当前目录,此cookie属性是必选的。

* NSHTTPCookiePort
NSString对象，包含逗号分隔的整数值，指定Cookie的端口。仅适用于version1或更高版本。默认值为空字符串,是可选的。

* NSHTTPCookieSecure
一个NSString对象，如果设置了这个属性，那么只会在 SSH 连接时才会回传该 Cookie。默认是`fasle`

* NSHTTPCookieVersion
Cookie有两个版本：Version 0 和 Version 1。通过它们有两种设置响应头的标识，分别是 “Set-Cookie”和“Set-Cookie2”。该属性的值必须是“0”或“1”。默认值为“0”,是可选的。

###2.Cookie的本地同步和动态绑定

`1`的操作首先是构造需要写入的自定义Cookie，构造成功后，因为Response中的Cookie不仅仅只有一个，所以我们通过url获取到当前域下的Cookie得到的是一个数组Cookies,然后将自定义的Cookie添加到原有的Cookies数组中得到新的Cookies。
```
NSMutableArray *cookies = [[[NSHTTPCookieStorage sharedHTTPCookieStorage] cookiesForURL:[NSURL URLWithString:@"https://backend.gmei.com"]] mutableCopy];
```
Okay,现在新Cookies我们已经构造好了。需要做的是，再新的http请求到来的时候，新的http请求的头部中的Cookies是我们构造好的Cookie。

* 场景1   自定义的Cookie的value是常量

如果我们仅仅写入的一个Cookie值是一个常量(当然很少见),我们可以直接将新的Cookies交由NSHTTPCookieStorage管理，执行完这段代码后，之后的请求Header中的Cookies就是新的Cookies。如果`fetchAccessTokenCookie方法中`设置一个例如24小时的过期时间的key，那么24小时内，新的Cookies在客户端都是有效的。

统提供setCookie方法,需要注意的是如果两个cookie的name名字相同,后面的会覆盖已经存在的cookie。
```
/*!
    @method setCookie:
    @abstract Set a cookie
    @discussion The cookie will override an existing cookie with the
    same name, domain and path, if any.
*/
[- (void)setCookie:(NSHTTPCookie *)cookie;]
```
执行就okay了
```
 NSHTTPCookie *accessCookie =  [self fetchAccessTokenCookie];
[[NSHTTPCookieStorage sharedHTTPCookieStorage] setCookie:accessCookie];
```

* 场景2   自定义的Cookie的value是变量且要求立即生效，不能遗漏任何一个http请求。

在这种要求生效的及时性和完整性的时候，上面的方案1就无法满足。这个时候我们需要将新的Cookies和接下来的请求url进行绑定。Cookie是HTTPHeader的一部分，所以现将得到的新Cookies，生成一个HTTPHeader，系统提供了将 cookie转成请求头的方法。

[requestHeaderFieldsWithCookies:](https://developer.apple.com/reference/foundation/nshttpcookie/1393021-requestheaderfieldswithcookies?language=objc)

####UIWebView
```
NSHTTPCookie *accessCookie =  [self fetchAccessTokenCookie];
NSDictionary *requestHeaders = [NSHTTPCookie requestHeaderFieldsWithCookies:accessCookie];
[request setValue:[requestHeaders objectForKey:@"Cookie"] forHTTPHeaderField:@"Cookie"];
[_webView loadRequest:request];
```
####AFNetworking
GET/PSOT/DELETE/PUT/PATCH
```
AFHTTPRequestOperationManager *manager = [AFHTTPRequestOperationManager manager];
manager.requestSerializer.HTTPShouldHandleCookies = YES;            
[manager.requestSerializer setValue:newCookies forHTTPHeaderField:@"Cookie"];
```
对于上传音视频的时候，AFNetworking用的是AFURLSessionManager而不是AFHTTPRequestOperationManager。
```
NSMutableURLRequest *request = [[AFHTTPRequestSerializer serializer] multipartFormRequestWithMethod:@"POST" URLString:URLString parameters:parameters constructingBodyWithBlock:^(id<AFMultipartFormData> formData) {
        //没有指定name的时候,name不能为空，默认填充 @"file" 即可
        [formData appendPartWithFileData:data name:name fileName:fileName mimeType:mineType];
    } error:nil];
    request.timeoutInterval = 30;
    NSHTTPCookie *accessCookie =  [self fetchAccessTokenCookie];
NSDictionary *requestHeaders = [NSHTTPCookie requestHeaderFieldsWithCookies:accessCookie];
    [request setValue:[httpHeader objectForKey:@"Cookie"] forHTTPHeaderField:@"Cookie"];
    AFURLSessionManager *manager = [[AFURLSessionManager alloc] initWithSessionConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration]];
    [UIApplication sharedApplication].networkActivityIndicatorVisible = YES;
    NSURLSessionUploadTask *uploadTask = [manager uploadTaskWithStreamedRequest:request progress:NULL completionHandler:^(NSURLResponse *response, id responseObject, NSError *error) {
        [UIApplication sharedApplication].networkActivityIndicatorVisible = NO;
        if (error) {
            [self failureWithTask:nil error:error failed:failed];
        } else {
            [self successWithTask:nil responseObject:responseObject success:success];
        }
    }];
    
    [uploadTask resume];

```
###Alamofire

```
var cookies = HTTPCookieStorage.shared.cookies(for: URL(string: @"http://backend.gmei.com")
if let cookie = fetchAccessTokenCookie() {
   HTTPCookieStorage.shared.setCookie(cookie)
   HTTPCookieStorage.shared.cookieAcceptPolicy = .always
   cookies?.append(cookie)
   let headers = HTTPCookie.requestHeaderFields(with: cookies!)
   let dataRequest = Alamofire.request(urlString, method: 
   method, parameters: parameters, encoding: 
   URLEncoding.default, headers: headers)
   dataRequest.validate(statusCode: 200..<400)
       .responseData { (response) in
       if let data = response.result.value, 
       response.result.isSuccess {
             
       } else {
            
       }
    }
} 
```

相关文章:
https://developer.apple.com/reference/foundation/nshttpcookie/http_cookie_attribute_keys?language=objc
https://my.oschina.net/kevinair/blog/192829
http://www.jianshu.com/p/d2c478bbcca5


