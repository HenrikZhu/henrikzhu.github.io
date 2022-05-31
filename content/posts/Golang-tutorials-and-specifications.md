---
title: "Golang tutorials and specifications"
date: 2020-04-09T08:24:45+08:00
draft: true
---



# Golang教程与规范

# 目的

1. 了解我们常用的技术与框架，快速手golang开发相关的工作。
2. 养成良好的代码习惯。
3. 避免一些常见的误区。

# golang教程

1. （必做）系统看一本go语言教程，搭建go语言开发环境，IDE推荐goland。 可以找yixu借一本，或者读这本在线的：https://docs.hacknode.org/gopl-zh/

下面几个可以工作中用到的时候再慢慢学些，如果有时间建议过一遍。

1. build-web-application-with-golang https://astaxie.gitbooks.io/build-web-application-with-golang/content/en/03.1.html
2. gorilla/mux(URL路由调度) 我们用的rest框架，支持正则表达式等多种路由方法 http://www.010lm.com/roll/2016/0613/2242035.html

How to UT gorilla handler: https://github.com/quii/testing-gorillas

1. Go db 连接文档 http://go-database-sql.org/modifying.html
2. About context https://dave.cheney.net/2017/08/20/context-isnt-for-cancellation
3. Structs/Json 定义 https://yourbasic.org/golang/json-example/

# 编程规范检查的tool，要放到CI里面

http://blog.ralch.com/tutorial/golang-tools-inspection/ go tool vet -composites=false ./

# 规范／Tips（真正的做开发之前，要读一遍）

## 1. 异常处理 ： https://www.cnblogs.com/zhangboyu/p/7911190.html

Error(错误)/Panic(异常)

### Error：

- 不要采用这种方式

```
if err != nil {
    // error handling
} else {
    // normal code
}
```

- 而要采用下面的方式

```
    if err != nil {
        // error handling
        return // or continue, etc.
    }
    // normal code
```

- 如果方法需要返回error，那么error要在方法声明的返回值列表的最后一个, e.g.

```
  func funcB() (string,error) 
```

### Panic

- 很多场景，我们需要在panic的时候，期望程序不会退出，可以继续执行。 如果在funcB（）里面做了recover的事情(if p := recover(); p != nil)，那么panic会被截住，main里面可以不受影响。

```
func funcB() error {
	defer func() error {
		if p := recover(); p != nil {
			fmt.Println("funcB recover! p: %v", p)
			//debug.PrintStack()
		}
		fmt.Println("funcB defer func")
		return errors.New("success")
	}()
	// simulation
	panic("test")
	return errors.New("success")
}

func main() {
	fmt.Println("from main", funcB())
	fmt.Println("do more")
}
```

- 在程序入口处，要写recover的code保证异常退出的时候捕获并打印相关的信息。 e.g.

```
func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Printf("panic recover! error: ", err)
			debug.PrintStack()
		}
	}()
```

## 2. 事务管理

1. 对于数据库的操作的方法的封装要把 *TransactionManager 声明成 receiver：
2. 对于具体数据库的操作，要掉用receiver的具体方法，比如：d.Insert(&rgtype)

```
func (d *TransactionManager) CreateResourceGroup(userId int, name string, desc string) (int, error) {
	var rg ResourceGroup = ResourceGroup{0, userId, name, desc}
	var rgtype ResourceGroupType = ResourceGroupType{rg}
	rgId, error := d.Insert(&rgtype)
	if error != nil {
		lavalog.Warning.Println(error)
	}
	return rgId, error
}
```

1. 在request handler方法里面，如下取出事务管理器的instance，所有的本request生命周期内的所有的数据库的操作都需要用这个instance来操作。

```
var transactionManager *database.TransactionManager
transactionManager = r.Context().Value("TransactionManager").(*database.TransactionManager)
```

1. 在任何一个步骤出现error，把error传到request的全局事务管理对象里面，该事务会被rollback。

```
transactionManager.Error = error
```

1. 特殊的场景，需要新建一个独立的事务来处理数据库操作，该事务不依赖于全局的事务。

```
        tempRequestHandler, err := database.CreateRequestHandler()
	if err != nil {
		lavalog.Error.Println(requestHandler.RequestId, "Can not create transaction", err)
		return cloudResource, err
	}
	if tempRequestHandler != nil {
		defer tempRequestHandler.Commit()
	}
	_, err = tempRequestHandler.Insert(&cloudResource)
	if err != nil {
		lavalog.Error.Println("Create Cloud Resource Error", err)
		return cloudResource, err
	}
```

1. 自己创建的事务，**一定一定一定**要显示的commit或者rollback。

## 3. 关于日志：

1. 方式：

-- fmt.Print* -- 控制台，不推荐

```
Oushu Cloud 1.0 starting...
```

-- log.Print* -- 控制台，推荐,可以打印文件名与行号

```
2018/07/04 11:23:36 main.go:48: Oushu Cloud 1.0 starting...
```

-- lavalog.*Level*.Print* -- 指定的log，默认为lava.log

```
INFO:    2018/07/04 09:39:40 azure.go:147: vmName: test-eason-22222-1
```

-- lavalog.*Level*.Fetal* -- 指定的log，并退出程序

1. 业务相关的日志，不要用fmt.Print* 或者log.Print*（少量的不重复的启动日志可以用）, 要用lavalog.*level*.Println(). 日志的level包括：Entry/Exit/Info/Error
2. 根据具体的场景打印不同level的日志 -- Level:

```
const entryLevel = 0
const exitLevel = 0
const infoLevel = 1
const warningLevel = 2
const errorLevel = 3
```

1. 日志里面，要打印requestid, method, 需要的参数。参见cloud/user/session.go Logon(w http.ResponseWriter, r *http.Request) 方法。

```
Exit:    2018/07/04 09:39:40 session.go:289: 1e8fddee: checkSession {-1000 oushu true  false false  oushu false public}
```

## 4. 关于锁：

### 数据库锁

postgres数据库的锁机制： https://blog.csdn.net/pg_hgdb/article/details/79403651 在update/insert/delete的时候，会有行的排他锁，而且这个锁，一直到他所属的事务结束之后才会释放。所以，如果一个事务时间太长，可能会导致并发效率很低的问题， 甚至导致死锁的现象。 最佳实践是：

1. 对于update/insert/delete这些操作，放到事务也就是request的最后面操作，这样hold锁的时间会变短。
2. 对于时间长的事务，尽量的避免，如果可以，拆分成多个更小粒度的事务。
3. 可以通过新建一个事务的方式来更新持久化的数据，操作完数据库，减少锁的时间。 比如，在创建集群的时候，我们要对operation插一条记录，而这条记录，不应该因为创建集群的事务的失败而回滚。 那么，我们会new一个新的事务来插这条记录，插完就commit。

### Golang锁

```
多个goroutine同时访问共有变量时，会出现数据不一致的问题，要用适当的锁来保证数据的一致性。读写锁效率会跟高一些。
对于map如果多协程共同读写，会出类似“concurrent map iteration and map write”错误。解决方案是：
1. 加读写锁。2. 用sync.Map代替map。建议自己加锁，sync.Map不成熟，有很多限制，而且代码不清晰。
```

## 5. 模块化：

- 松耦合：
- 面向接口编程： 一种业务逻辑的多种实现，最好用接口去做。比如支付模块要支持支付宝与微信支付，每一种支付方式都是一个实现。再比如集群管理的操作。
- 扩展性：前期具体的实现可以不完美，但是架构尽量可扩展。

## 6. 关于hard-code：

```
1. 尽量避免hard-code一些常量。比如，actionname， 有start，stop，dallocate等，这些在好多地方用到了，需要定义全局的常量来存储，大家都引用这些常量，减少后期维护的难度。
2. 所有的账号、密码（不限这些）等私密的东西，都不要hard-code在源码里面。
```

## 7. 文档／注释

1. 包 每个程序包都应该有一个包注释，一个位于package子句之前的块注释或行注释。 包如果有多个go文件，只需要出现在一个go文件中即可。

```
//Package regexp implements a simple library 
//for regular expressions.
package regexp 
```

1. 结构体

```
// Use Billing response.
type GetUserBillResponse struct {
	Userid          	int     `json:"userid"` //User Identifier
	Username        	string  `json:"username"` //User Name
	Amount          	float32 `json:"amount"` //Total amount ever used
	AmountThisMonth 	float32 `json:"amount_this_month"`// Total amount used this calendar month.
	AmountLastMonth 	float32 `json:"amount_last_month"`// Total amount used last calendar month.
	AmountFuture    	float32 `json:"amount_future"` // Amount fore case for next 24 hours.
	AIAmountThisMonth 	float32 `json:"ai_amount_this_month"` // Amount for AI this month.
	                                                          // This is already counted into the AmountThisMonth
	                                                          // but we need show AI specific amount in UI.
	AIAmount 			float32 `json:"ai_amount"` // Total amount ever used for AI which is already counted into Amount.
}
```

1. 方法 程序中每一个被导出的（大写的）名字，都应该有一个文档注释。需要

- 这个方法干什么
- （如果逻辑复杂）：分几步，每一步干什么？
- 各个参数的描述与返回值的描述。

```
// Logon the user.
//
// 1. Check the input Json, verify the username/password according to the Users table.
//    There will be encrypt/decrypt logic here. Because password should be encrypted.
//
// 2. If passed the verification at step 1, create cookie to the response, set the signature to the cookie for
//    later session verification.
//
// 3. Update the users.lastsession with the time used to build the signature.
//
//w: ResponseWriter
//
//r: *http.Request
func Logon(w http.ResponseWriter, r *http.Request) {
```

第一条语句应该为一条概括语句，并且使用被声明的名字作为开头

```
// Compile parses a regular expression and returns, if successful, a Regexp
// object that can be used to match against text.
func Compile(str string) (regexp *Regexp, err error) {
```

1. 程序内部. 复杂或者不太易懂的逻辑要加注释。e.g.

```
func getHostFromRequestURL(r *http.Request) string {
	requestHandler := r.Context().Value("TransactionManager").(*database.TransactionManager)
	requestId := requestHandler.RequestId + ": " + "getHostFromRequestURL"
	host := r.Host
	var index = strings.Index(host, ":")
	if index > 0 { 
                // The host is with port e.g. localhost:4200
		// Parse the real host without the port number.
		host = host[0:index]
	}
	lavalog.Exit.Println(requestId, host)
	return host
}
```

## 8. 格式

大部分的格式问题可以通过gofmt解决，gofmt自动格式化代码，保证所有的go代码一致的格式。签入代码前*必须*使用`gofmt`工具进行自动格式化。

## 9. goroutine/线程

1. 如何起？
2. 注意事项：传参数与直接用参数，事务的坑。

## 10. 命名

### 总的原则

[驼峰式命名](https://baike.baidu.com/item/骆驼命名法?fr=aladdin)

```
MixedCaps 大驼峰法，大写开头，可导出
mixedCaps 小驼峰法，小写开头，不可导出
```

### 包名

包名应该为小写单词，不要使用下划线或者混合大小写。

### 接口名

接口名以"er"作为后缀，如Reader,Writer；接口的实现则不要“er”

```
type Reader interface {
        Read(p []byte) (n int, err error)
}
```

两个函数的接口名综合两个函数名，并加"er"

```
type WriteFlusher interface {
    Write([]byte) (int, error)
    Flush() error
}
```

三个以上函数的接口名，类似于结构体名

```
type Car interface {
    Start([]byte) 
    Stop() error
    Recover()
}
```

## 11. Rest API 设计原则

https://blog.csdn.net/houjixin/article/details/54315835

- 所有的Rest API都是小驼峰写法，比如 /users /userGroup
- API尽量不要用动词， 比如/getusers, 应该是 GET(http action) /users
- HTTP Actions 优先使用HTTP的Action：POST（create），GET（get），PUT（update），DELETE 如果上面四种action不满足我们的需求，可以自己定义action，放到api最后：PUT /user/{userId}/password/reset, reset是我们自定义的action。
- Json对象里面的参数，也要用驼峰命名。
- Response Json里面一定要有error string这个字段,正常返回的时候，error字段是空，否则会是相应的error message。

Response Code： -- 200 OK -- 401 authenication fail -- 404 object notfound

## 12. 数据库操作

- 数据库的存取 [https://github.com/oushu-io/lava/wiki/Cloud-%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B8%8E%E6%95%B0%E6%8D%AE%E5%BA%93%E5%AF%B9%E5%BA%94%E7%9B%B8%E5%85%B3](https://github.com/oushu-io/lava/wiki/Cloud-数据结构与数据库对应相关)
- 我们封装了数据库的insert／select方法。其中Insert是安全的，SelectSafe也是安全的，不要用Select.
- 数据库的读写，一定要用 $符号的占位符，避免sql注入

```
sql := "update operation set status=$1, updatetime=$2 where userid=$3 and operation= $4 and acttime = updatetime"
d.Transaction.Exec(sql, status, updateTime, userId, operation);
```

不要这样字符串拼接：

```
sql := "update operation set status="+status+" where userid="+userid
d.Transaction.Exec(sql);
```

## 13. 显式用短http连接

```
        req.Close = true  // 1. 显示的表示要用短连接。
	resp, err := client.Do(req)
	if err != nil {
		return "", err
	}
	defer resp.Body.Close() // 2.显示的关掉 response body 
```

## 14. 其他Tips

- Go语言对数组或者map的操作 用下标操作能改变原来的数组或者map里面的值，否则不可以。
  for _, node := range mymap { node.Value = "1111" //修改不了原来的值。传值 } for i, _ := range mymap { mymap[i].Value = "1111" //能修改原来的值。等同于指针操作 }
- := 声明变量与赋值的坑

如果变量不在同一个作用域，那么，会建立一个新的变量。比如：

```
var requestBytes []byte
if xyz {
   requestBytes, err := getRequestBytes()//此处，会创建一个新的requestBytes，切记切记
}
```

正确做法

```
var requestBytes []byte
if xyz {
   var err error
   requestBytes, err = getRequestBytes()//此处，if之外声明的那个
}
```

- 一些最佳实践： https://zhuanlan.zhihu.com/p/27518650

# IDE tips

- goland如果三个月试用期到了，删掉重装
- Refactor ![default](https://user-images.githubusercontent.com/32856755/39029571-1229d716-448f-11e8-9d05-f0ae49ced0a4.png)
- 批量搜索

![default](https://user-images.githubusercontent.com/32856755/39029559-02ea5654-448f-11e8-816f-42d2d9d1cddc.png)

- 数据库插件 ![default](https://user-images.githubusercontent.com/32856755/39029540-e819a4c4-448e-11e8-8fe3-e79e76975b38.png)
- 定位当前执行的问天在哪个包内

![default](https://user-images.githubusercontent.com/32856755/39110743-d7c21930-4704-11e8-8372-181401a91a0e.png)

- Git相关
- 命令行 下面有Terminal命令行工具，方便在ide里面执行command，不用切换出去用系统的Terminal。 可以通过command + T打开多个Terminal。
- 常用快捷键（注意，必须切换到英文输入，快捷键才有用） command + [ ]， 回退／前进到前一个视图 command + O， 打开一个文件 option + 回车，自动引入光标所在的错误处的包依赖（import） command + N ，打开最近浏览过的文件 command + ／用//注释选中的行（可以多行） command + shift + /， 用／****／注释选中的行（可以多行） command+R，替换文本。 command+F，查找文本。 command+SHIFT+F，进行全局查找。 command+G，快速定位到某行 command+B，快速打开光标处的结构体或方法（跳转到定义处）

附：后端环境搭建tips go环境检查 go env检查安装环境 go env GOPATH可看安装默认 echo $GOPATH 查看系统GOPATH export GOPATH= 设置系统GOPATH