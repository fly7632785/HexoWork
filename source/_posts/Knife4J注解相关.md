# Controller层添加注解
### @Api:用于类；表示标识这个类是swagger的资源
| 属性名称       | 数据类型        | 默认值 | 说明                                                      |
| -------------- | --------------- | ------ | --------------------------------------------------------- |
| value          | String          | ""     | 字段说明                                                  |
| tags           | String[]        | ""     | 标签说明                                                  |
| description    | String          | ""     | 详情描述                                                  |
| basePath       | String          | ""     | 基本路径可以不配置                                        |
| position       | int             | ""     | 如果配置多个Api 想改变显示的顺序位置                      |
| produces       | String          | ""     | 提供者 (For example, "application/json, application/xml") |
| consumes       | String          | ""     | 消费者(For example, "application/json, application/xml")  |
| protocols      | String          | ""     | 协议(Possible values: http, https, ws, wss.)              |
| authorizations | Authorization[] | ""     | 高级特性认证时配置                                        |
| hidden         | boolean         | ""     | 配置为true 将在文档中隐藏                                 |

注解加载在controller类上

常用模型:
```
@Api(tags = "HELLO CONTROLLER 测试功能接口")
 @RestController
 public class HelloController {
 
 }
```

### @ApiResponses:在 Rest 接口上使用，用作返回值的描述
| 属性名称 | 数据类型      | 默认值 | 说明     |
| -------- | ------------- | ------ | -------- |
| value    | ApiResponse[] | ""     | 访问对象 |

#### ApiResponse参数
| 属性名称          | 数据类型         | 默认值 | 说明                                                         |
| ----------------- | ---------------- | ------ | ------------------------------------------------------------ |
| code              | String           | ""     | 响应的HTTP状态码                                             |
| message           | String           | ""     | 响应的信息内容                                               |
| response          | Class<?>         | ""     | 用于描述消息有效负载的可选响应类，对应于响应消息对象的 schema 字段 |
| reference         | String           | ""     | 指定对响应类型的引用，指定的应用可以使本地引用，也可以是远程引用，将按原样使用，并将覆盖任何指定的response()类 |
| responseHeaders   | ResponseHeader[] | ""     | 声明包装响应的容器，有效值为List或Set，任何其他值都将被覆盖  |
| responseContainer | String           | ""     | 声明响应的容器，有效值为List,Set,Map，任何其他值都将被忽略   |
| examples          | Example          | ""     | 例子                                                         |

注解加载在方法上

常用模型:
```
@ApiResponses(value = {
            @ApiResponse(code = 200, message = "接口返回成功状态"),
            @ApiResponse(code = 500, message = "接口返回未知错误，请联系开发人员调试")
    })
    @PostMapping("hello")
    public Results<UserVO> hello(@RequestBody UserVO userVO){

        Results<UserVO> results = new Results<>(200,"SUCCESS", userVO);
        return results;
    }
```

### @ApiOperation:用在方法上，说明方法的作用，每一个url资源的定义
| 属性名称          | 数据类型         | 默认值 | 说明                                                         |
| ----------------- | ---------------- | ------ | ------------------------------------------------------------ |
| value             | String           | ""     | url的路径值                                                  |
| notes             | String           | ""     | 文本说明                                                     |
| tags              | String[]         | ""     | 如果设置这个值、value的值会被覆盖                            |
| response          | Class<?>         | ""     | 返回的对象                                                   |
| responseContainer | String           | ""     | 声明响应的容器，有效值为List,Set,Map，任何其他值都将被忽略   |
| responseReference | String           | ""     | 声明包装响应的容器，有效值为List或Set，任何其他值都将被覆盖  |
| httpMethod        | String           | ""     | "GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS" and "PATCH" |
| position          | int              | ""     | 如果配置多个Api 想改变显示的顺序位置                         |
| nickname          | String           | ""     | 昵称                                                         |
| produces          | String           | ""     | 提供者 (For example, "application/json, application/xml")    |
| consumes          | String           | ""     | 消费者(For example, "application/json, application/xml")     |
| protocols         | String           | ""     | 协议(Possible values: http, https, ws, wss.)                 |
| authorizations    | Authorization[]  | ""     | 高级特性认证时配置                                           |
| hidden            | boolean          | ""     | 隐藏                                                         |
| responseHeaders   | ResponseHeader[] | ""     | 声明包装响应的容器，有效值为List或Set，任何其他值都将被覆盖  |
| code              | String           | ""     | http的状态码 默认 200                                        |
| extensions        | Extension[]      | ""     | 扩展属性                                                     |
| ignoreJsonView    | boolean          | ""     | 是否忽略显示                                                 |

注解加载在方法上

常用模型:
```
@ApiOperation(value = "Hello 测试接口", notes = "访问此接口，返回hello语句，测试接口")
    @PostMapping("hello")
    public Results<UserVO> hello(@RequestBody UserVO userVO){

        Results<UserVO> results = new Results<>(200,"SUCCESS", userVO);
        return results;
    }
```

### @PathVariable:是获取get方式，url后面参数，进行参数绑定(单个参数或两个以内参数使用)
参数使用@PathVariable注解，会自动解析，不需要额外的注解

### @RequestBody :在当前对象获取整个http请求的body里面的所有数据(两个以上参数封装成对象使用)
参数使用@RequestBody 注解，会自动解析，不需要额外的注解