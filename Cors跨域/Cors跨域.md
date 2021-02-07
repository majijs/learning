# Cors跨域

## 方式一：网关自定义CorsFilter

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsFilter corsFilter() {
        
        String allowedOrigin = "*";
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true); // 允许cookies跨域
        config.addAllowedOrigin(allowedOrigin);// 允许向该服务器提交请求的URI，*表示全部允许。。这里尽量限制来源域，比如http://xxxx:8080 ,以降低安全风险。。
        config.addAllowedHeader("*");// 允许访问的头信息,*表示全部
        config.setMaxAge(18000L);// 预检请求的缓存时间（秒），即在这个时间段里，对于相同的跨域请求不会再预检了
        config.addAllowedMethod("*");// 允许提交请求的方法，*表示全部允许，也可以单独设置GET、PUT等
    /*    config.addAllowedMethod("HEAD");
        config.addAllowedMethod("GET");// 允许Get的请求方法
        config.addAllowedMethod("PUT");
        config.addAllowedMethod("POST");
        config.addAllowedMethod("DELETE");
        config.addAllowedMethod("PATCH");*/
        source.registerCorsConfiguration("/**", config);
        return new CorsFilter(source);
    }


    @Bean
    public RequestFilter filter() {
        return new RequestFilter();
    }
}
```

## 方式二：Nginx配置

标准配置

```shell
server {
    ... ...
 
        # #设置跨域配置 Start
        set $cors_origin "";
        if ($http_origin ~* "^http://api.xx.com$"){
                set $cors_origin $http_origin;
        }
 
        add_header Access-Control-Allow-Origin $cors_origin always; 
        add_header Access-Control-Allow-Methods GET,POST,PUT,DELETE,OPTIONS always;
        add_header Access-Control-Allow-Credentials true always;
        add_header Access-Control-Allow-Headers DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization,x-auth-token always;
        add_header Access-Control-Max-Age 1728000 always;
 
        # 预检请求处理
        if ($request_method = OPTIONS) {
                return 204;
        }
        # #设置跨域配置 End
 
    ... ...
}
```

设置多域名配置

```shell
   set $cors_origin "";
   if ($http_origin ~* "^http://api.xx.com$"){
   set $cors_origin $http_origin;
   }
   if ($http_origin ~* "^http://api2.xx.com$"){
   set $cors_origin $http_origin;
   }
```

另外还有其他方式，如使用 @CrossOrigin，具体可参见https://cloud.tencent.com/developer/article/1513473