## 微服务

### go-micro框架

+ 相关概念
  + 首先，我们要明确micro和go-micro的区别。micro是一个微服务的管理工具，就相当与我们平时用的工具包，负责管理微服务相关。go-micro是一个组件化的框架，提供了服务发现，负载均衡，同步传输，异步通信以及事件驱动的等机制。
  + consul是一个组件的注册中心，我们注册的相关微服务可以在上面看到。
  + goprotobuf相关工具。GRPC工具包

### gRPC技术

+ 不用proto生成代码

  + 这种方法是定义一个客户端和服务端，我们在服务端先生成我们的业务服务，然后调用rpc注册我们的服务，同时将服务注册到http中，之后监听对应的端口即可。

  + 定义好服务端后，我们可以在客户端进行调用，我们通过DailHttp进行连接暴露的端口，之后我们可以利用Call方法进行同步调用或者利用Go方法进行异步调用，但是异步调用的话需要注意的是可能会发生阻塞从而导致程序一直停留。

    ```go
    	//同步调用
    	//err = client.Call("MathUtil.CalculateCircleArea", req, &resp)
    	//if err != nil {
    	//	panic(err.Error())
    	//}
    	//fmt.Println(*resp)
    	//异步调用
    	syncCall := client.Go("MathUtil.CalculateCircleArea", req, &resp, nil)  //第一个参数是结构体名.函数接口名
    	call := <- syncCall.Done  //注意这里可能会发生阻塞
    	fmt.Println(call)
    	fmt.Println(*resp)
    ```

  + 如果有对多个参数的传递或者计算出多个值，可以先将变量封装在一个结构体里面，我们可以创建一个model包，将变量用结构体进行封装，然后传递结构体变量即可。
  
  + 最后先执行server端的程序，然后执行client的程序即可。
  
  + ```Go
    //server.go
    package main
    
    import (
    	"github.com/Peterliang233/demo/parm"
    	"net"
    	"net/http"
    	"net/rpc"
    )
    
    type MathUtil struct {
    }
    
    //调用方法接口
    func (mu *MathUtil) CalculateCircleArea(req parm.Result, resp *float64) error {
    	*resp = req.Num1 + req.Num2
    	return nil
    }
    
    func main() {
    	mathUtil := new(MathUtil)  //实例化
    	err := rpc.Register(mathUtil)  //注册一个远程调用函数
    	if err != nil {
    		panic(err.Error())
    	}
    	//把服务注册到http协议
    	rpc.HandleHTTP()
    
    	listen, err := net.Listen("tcp", ":9091")
    	if err != nil {
    		panic(err.Error())
    	}
    	err = http.Serve(listen, nil)
    	if err != nil {
    		panic(err.Error())
    	}
    }
    ```
  
  + ```Go
    //client.go
    package main
    
    import (
    "fmt"
    	"github.com/Peterliang233/demo/parm"
    	"net/rpc"
    )
    
    func main() {
    	client, err := rpc.DialHTTP("tcp", "localhost:9091")
    	if err != nil {
    		panic(err.Error())
    	}
    
    	req := &parm.Result{
    		13.93,
    		123.9,
    	}
    
    	var resp *float64
    	//同步调用
    	//err = client.Call("MathUtil.CalculateCircleArea", req, &resp)
    	//if err != nil {
    	//	panic(err.Error())
    	//}
    	//fmt.Println(*resp)
    	//异步调用
    	syncCall := client.Go("MathUtil.CalculateCircleArea", req, &resp, nil)
    	call := <- syncCall.Done  //注意这里可能会发生阻塞
    	fmt.Println(call)
    	fmt.Println(*resp)
    }
    
    ```



+ 用proto生成代码

  使用proto自动生成对应的.go文件，这里我们首先学习一写proto的基本语法：

  ```protobuf
  // 首先，我们在第一行申明我们使用的是哪种proto协议。
  syntax = "proto3";
  // 我们可以使用option定义我们的文件生成的路径
  option go_package = ".go文件生成的位置;proto文件名";
  // 定义包的名字，生成的.go文件会在同名的包下面
  package message;
  // message是proto的关键字，利用这个关键字，我们可以指定字段的类型，这里我们每个字段会分配一个数字，这个数字是唯一的，代表每个字段的编号,如果想定义多个字段的话，可以像定义结构体那样类似地定义我们的消息字段，在每个单独的消息字段，我们的字段编号必须是唯一的，另外就是要注意一下字段定义的方法，它是将字段类型放在前面，类似于C++那种定义方法
  message SearchRequest {
  	string query = 1;
  	int32 page_number = 2;
  	int32 result_per_page = 3;
  }
  ```

  我们编写好proto文件之后，执行`protoc proto_path --go_out=DST_DIR`就可以生成对应的.go文件了

  ```go
  // proto文件
  syntax = "proto3";
  option go_package="./;message";
  package message;
  
  //订单请求参数
  message OrderRequest {
      string orderId = 1;
      int64 timeStamp = 2;
  }
  
  //订单信息
  message OrderInfo {
      string OrderId = 1;
      string OrderName = 2;
      string OrderStatus = 3;
  }
  
  
  //server.go
  package main
  
  import (
  	"errors"
  	"github.com/Peterliang233/demo/message"
  	"net"
  	"net/http"
  	"net/rpc"
  	"time"
  )
  
  type OrderService struct {
  }
  
  func (os *OrderService) GetOrderInfo(request message.OrderRequest, response *message.OrderInfo) error {
  	orderMap := map[string]message.OrderInfo{
  		"201907300001": message.OrderInfo{OrderId: "201907300001", OrderName: "衣服", OrderStatus: "已付款"},
  		"201907310001": message.OrderInfo{OrderId: "201907310001", OrderName: "零食", OrderStatus: "已付款"},
  		"201907310002": message.OrderInfo{OrderId: "201907310002", OrderName: "食品", OrderStatus: "未付款"},
  	}
  
  	current := time.Now().Unix()
  
  	if request.TimeStamp > current {
  		response = &message.OrderInfo{
  			OrderId: "0",
  			OrderName: "",
  			OrderStatus: "订单信息异常",
  		}
  	} else {
  		result := orderMap[request.OrderId]
  		if result.OrderId != "" {
  			*response = result
  		} else {
  			return errors.New("server error")
  		}
  	}
  	return nil
  }
  
  func main() {
  	orderService := new(OrderService)
  	_ = rpc.Register(orderService)
  	rpc.HandleHTTP()
  	listen, err := net.Listen("tcp", ":9090")
  	if err != nil {
  		panic(err.Error())
  	}
  	err = http.Serve(listen, nil)
  	if err != nil {
  		panic(err.Error())
  	}
  }
  
  
  //client.go
  package main
  
  import (
  	"fmt"
  	"github.com/Peterliang233/demo/message"
  	"net/rpc"
  	"time"
  )
  
  func main() {
  	client, err := rpc.DialHTTP("tcp", ":9090")
  	if err != nil {
  		panic(err.Error())
  	}
  	timeStamp := time.Now().Unix()
  	request := message.OrderRequest{
  		OrderId:   "20190730001",
  		TimeStamp: timeStamp,
  	}
  	var response *message.OrderInfo
  	err = client.Call("OrderService.GetOrderInfo", request, &response)
  	if err != nil {
  		panic(err.Error())
  	}
  	fmt.Println(*response)
  }
  ```

+ 使用流模式

  首先执行` go get -u google.golang.org/grpc  `导入相关的库文件

  然后编写对应的.porto文件

  执行`protoc (.proto文件的路径) --go_out=plugins=grpc:(指定生成文件的路径)`就可以生成对应的pb.go文件(注意，中间不能有空格)，里面有流的相关函数

  

## Web应用开发

### 目录结构

+ api表示的是相关的版本的信息，一般下面的目录为v1，v2等等

+ configs存放的是相关的配置，我们把一些server配置和mysql的一些配置文件放在里面

+ databases存放的是数据库的连接和初始化的一些操作

+ 中间件目录存放的是一些登录验证，jwt验证等

+ 模型层存放的是一些结构体，我们将用户的属性用结构体进行打包存放

+ 路由目录是存储的是路由接口

+ utils存放的是一些处理的函数，如读取配置的函数等等

### gorm框架

+ gorm是一个开启数据库

  + ```Go
    databases.Db, err = gorm.Open(utils.Db,
    		fmt.Sprintf("%s:%s@tcp(%s:%s)/%s?charset=utf8&parseTime=True&loc=Local",
    			utils.DbUser,
    			utils.DbPassword,
    			utils.DbHost,
    			utils.DbPort,
    			utils.DbName,
    		))
    ```

+ gorm设置分页

  + ```GO
    //获取用户分页列表
    func GetUsers(PageSize, PageNum int) []User{
    	var users []User
    	err := databases.Db.Limit(PageSize).Offset((PageNum-1)*PageSize).Find(&users).Error
    	if err != nil && err != gorm.ErrRecordNotFound{
    		return nil
    	}
    	return users
    }
    ```

+ gorm的删除一般都是软删除，在数据库里面其实并没有进行数据的删除，但那是我们在进行查询的时候不会查询到对应被软删除的数据。

### 信息安全

#### JWT（JSON Web Tokens)中间件

##### jwt原理

+ 服务器认证以后，生成一个json对象，发回给用户，之后用户与服务端通信的时候，都要发回这个json对象，服务器就只要靠这个对象认定用户身份。

##### jwt数据结构

jwt解析之后，就是一个很长的字符串，这个字符串分为三个部分，每个部分之间用$.$进行分割。

+ jwt三部分分为Header(头部)，Payload(负载)，Signature(签名).

+ 写成一行的话就是Header.Payload.Signature

+ Header

  + Header是一个json对象，描述jwt的元数据

  ```go
  {
  	"alg": "HS256",
  	"typ": "JWT"
  }
  ```

  上面的代码中，alg属性表示签名的算法，默认的时HMAC SHA256,typ属性表示这个令牌的类型，jwt令牌统一写成jwt。

+ Payload

  + Payload部分也是一个json对象，用来存放实际需要传递的数据。jwt规定了7个官方的字段

    ```go
    + iss(issuer):签发人
    + exp (expiration time): 过期时间
    + sub (subject):主题
    + aud (audience): 受众
    + nbf (Not Before):生效时间
    + iat (Issued At):签发时间
    + jti (JWT ID):编号
    ```

  + 除了官方定义的字段，你可以自己定义私有字段。

  + 这个部分是公有的，大家都可以访问，所以注意不要把自己的重要的信息放到这个部分。

  + 这个json对象也要使用Base64URL 算法转化成字符串

+ 客户端接收到服务器返回的jwt，可以存储在Cookie里面，也可以储存在localStorage里面。之后每次客户端与服务器之间的通信，都会带上这个jwt。当然也可以放在cookie里面。但是这样的话就不能跨域，最好的办法就是放在http请求的头信息里面Authorization字段里面。另一种做法就是把jwt放到post请求的数据体里面。

```go
Authorization: Bearer <token>
```

+ jwt的几个特点
  + jwt默认不是加密的，但也可以是加密的，生成原始的token之后，可以再用密钥再加密一次。
  + jwt不加密的情况下，不能将秘密数据写入jwt。
  + jwt不仅可以用于认证，也可以用于信息交换。有效使用jwt，可以降低服务器查询数据库的次数。
  + jwt最大的缺点是，由于服务器不保存session状态，因此无法在使用过程之中废除某一个token，或者更改token的权限，一旦jwt签发下来，在设置的过期时间之前均有效。
  + 为了降低jwt被盗的风险，我们应该将jwt的有效时间设置得短一些。
  + 为了减少盗用，jwt不应该使用http协议明码进行传输，要使用https协议传输。

### 跨域

#### 什么是跨域？

当两个协议具有相同的端口，相同的host的时候，我们就可以认为这两个协议是具有相同的域。

#### 为什么要有跨域？

跨域就是指协议，域名和端口不一致的时候，处于安全的考虑，跨域的资源之间无法交互的。

#### CORS

+ cors是w3c的一个协议，全称是“跨域资源共享”
+ cors需要浏览器和服务器同时支持。
+ cors通信的关键是服务器，只要服务器实现了cors接口，那么就可以实现跨源通信。

#### 两种请求

浏览器将CORS的请求分为两类：简单请求和非简单请求。

+ 简单请求

  + 浏览器发现这次是简单请求，就会自动在头信息之中，增加一个Origin字段。
  + 只有在Origin指定的范围内的源，才可以被接受，否则会返回一个http回应。即对应的源需要被指定为：Access-Control-Allow-Origin
  + Access-Control-Allow-Origin:该字段是必需的，他指定了哪些源是被允许进行跨域的。这个一般都是指定为*
  + Access-Control-Allow-Credentials:该字段是可选的，它是一个bool值，表示是否允许发送Cookie。这个值一般设置为True
  + Access-Control-Expose-Headers：该字段也是可选的。
  + withCredentials属性：这个属性的设置主要是是否把cookie发送到服务器上面，这个要服务器同意，所以要在服务端设置为true。

+ 非简单请求

  预检请求（在正式的通信之前，增加一次http查询请求，称为“预检”请求。

  + 非简单请求是对服务器有特殊请求的请求，比如PUT和DELETE，或者Content-Type的字段的类型为application/json
  + Access-Control-Request-Method:这个字段是必须的，用来列出使用了哪些http方法。
  + Access-Control-Requese-Headers:这个是用逗号分割的字段，用来指定浏览器cors请求会额外发送的头信息字段。
  + Access-Control-Max-Age:这个字段用来指定的时本次预检请求的有效期，单位为秒。

#### 与JSONP的比较

+ CORS与JSONP的使用的目的相同，但是比JSONP更强大。
+ JSONP只支持GET请求，而CORS支持所有的http请求。JSONP的优点是支持老式的浏览器以及可以向不支持CORS的浏览器请求数据。

### 邮件发送

安装库

```
$ go get github.com/jordan-wright/email
```

配置：打开邮箱的设置，使用`SMTP/POP3/IMAP`等协议从邮件服务器上面拉去邮箱。带你级开启`IMAP/SMTP`服务，按照步骤开启即可。同时记住给出的密码，后面登录有用

```go
packge main
import(
    "log"
    "net/smtp"
    "github.com/jordan-wright/email"
)
func main(){
    e := email.NewEmail()  //实例化
    e.From = "Peterliang<ncuyanping666@126.com>"  //邮件的发送人
    e.CC = []string{"test1@qq.com"} //邮件的抽送人
    e.To = []string{"2107917115@qq.com"}   //邮件的接收人
    e.BCC = []string{"secret@qq.com"}  //秘密抄送
    e.Subject = "Testing"  //邮件主题
    e.Text = []byte("This is a test email")  //邮件的内容
    //内容以HTML的格式进行发送
    //e.HTML = []byte(
      // <h1> This is a head </h1>
   // )
    err := e.Send("smtp.126.com:25", smtp.PlainAuth("","邮箱","上面获取的密码","smtp.126.com"))  //发送邮件
    if err != nil {
        log.Fatal(err)
    }
}
```

### mqtt使用

+ 首先，下载对应的mqtt包，`yay -S mosquitto  ` 
+ 下载好之后需要启动相应的服务器 `systemctl start mosquitto`   和`systemctl enable mosquitto`。
+ 之后，我们可以创建一个订阅者`mosquitto_sub -t topic   ` ,同时创建一个发布者`mosquitto_pub -t topic -m "订阅消息"`





### 测试

##### 单元测试

+ Go语言的测试分为单元测试和基准测试。用于测试的文件的命名方法为`xxx_test.go`这个是一种默认的默认的命名方法，我么先写好一个用户测试的函数，然后就可以在这个测试文件里面测试这个函数的效果。

+ ```go
  package main
  
  import (
  	"fmt"
  	"testing"
  )
  
  func Add(a, b int) int {
  	return a + b
  }
  
  func TestAdd(t *testing.T) {
  	var a = 1
  	var b = 2
  
  	c := Add(a, b)
  
  	fmt.Println(c)
  }
  ```

+ 测试函数的命名规则就是Testxxx，其中Test为单元测试的固定开头，go test只会执行次开头的方法，



##### 基准测试

+ 基准测试是利用go的benchmark包里面的相关代码进行实现的。一般是用来测试一个函数的执行的时间。函数的命名规则一般为Benchmarkxxx, 基准测试是将这个函数执行很多次，然后计算每个函数执行的平均时间。以此来达到测试函数性能的目的。

+ ```go
  package test
  
  import "testing"
  
  func BenchmarkAppendSlice1(b *testing.B) {
  	for i := 0; i < b.N; i++ {
  		AppendSlice1()
  	}
  }
  
  
  func BenchmarkAppendSlice2(b *testing.B) {
  	for i := 0; i < b.N; i ++ {
  		AppendSlice2()
  	}
  }
  ```

  

##### 子测试

+ 子测试提供了一种在一个测试函数里面执行多个测试的能力，每个测试函数执行初始化需要做一些相同的初始化工作。利用子测试可以只需要执行一次初始化工作。

+ ```go
  func TestSub(t *testing.T){
      t.Run("A=1", Sub1)
      t.Run("B=2", Sub2)
      t.Run("C=3",sun3)
  }
  ```

##### 子测试并发

##### Main测试

+ main测试通过生命一个`func testMain(m *testing.m)`，它是名字比较特殊的测试，如果声明了一个函数，则当前测试程序将不是直接执行各项测试，而是将测试交给Testmain调度。

### Go插件

+ 验证插件
  + 验证插件用的是`"github.com/go-playground/validator/v10"`这个第三方库，用法就是在结构体上打上一个json的tag，然后在json里面设置这个属性的一些限制以及验证的信息。然后在函数里面注册一个validate的实例，最后就可以进行验证这个结构体是否满足了。但是这里有一个不太好的地方就是验证报错的信息是英文的，用户体验感不是很好，这里可以利用一个翻译的插件库`"github.com/goplayground/validator/v10/translations/zh"`，这个插件库就是先注册一个翻译器，然后就是将这个验证器和翻译器绑定在一起就行了。



### Go使用Https

+ 我们首先需要在认证平台获取.pfx格式的文件，然后通过这个文件生成.pem公钥，`openssl pkcs12 -in peterliang-top-iis-1104231106.pfx -nodes -out test.pem `
+ 获得这个公钥之后，我们需要利用这个公钥获取私钥，` openssl rsa -in test.pem -out test.key`
+ 这样，我们就得到了公钥和私钥匙，这样就可以放到项目的请求接口里面的中间件里面进行安全连接了。
