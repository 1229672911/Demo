# api_nexus_login

## 代码解读

**功能包含用户登录和修改密码**



### 路由注册

```go
func RegisterNexusLoginAPIRoutes(router *gin.Engine, container *nexus_wire.AppContainer) {
    api := router.Group("/nexus-api")
    // 使用闭包来引入configUpdater
    api.POST("/login", func(c *gin.Context) {
       login(c, container.GetDB())
    })
    api.Use(web_auth.AuthMiddleware)
    {
       api.PUT("/user/password", func(c *gin.Context) {
          changePassword(c, container.GetDB())
       })
    }
}
```

- **`router.Group("/nexus-api")`**：创建一个路由组，前缀为 `/nexus-api`。
- **`api.POST("/login", ...)`**：注册一个 $POST$ 路由 `/nexus-api/login`，处理用户登录请求。
- **`api.Use(web_auth.AuthMiddleware)`**：为 `/nexus-api` 下的所有路由添加 $JWT$ 认证中间件。
- **`api.PUT("/user/password", ...)`**：注册一个 $PUT$ 路由 `/nexus-api/user/password`，处理用户修改密码请求。



**路由组是什么，这里$router.Group()$的具体用法**

- 路由组是 $Gin$ 框架中用于组织和管理一组相关路由的功能。它允许为多个路由设置共同的前缀、中间件或其他配置，从而提高代码的可读性和可维护性。

- $router.Group()$创建一个路由组，该组路由都会以$/nexus-api$为前缀

**注册一个$POST$路由，调用$login()$函数来处理用户的登录请求**

- $container.GetDB()$ 用于获取数据库连接（$gorm.DB$ 对象），并将其传递给 $login()$ 函数

**中间件是什么，$api.Use()$更详细的解释**

- 中间件是$Gin$框架的一种机制，用于在请求到达路由处理函数之前或之后执行一些公共逻辑。如身份验证，日志记录，或者错误处理等。

**？？中间件是否可以理解为在调用函数之前先进行一些操作，来判断是否可以调用该函数**---可以

- $api.Use()$为路由组中所有路由添加一个中间件
- $web_auth.AuthMiddleware$是一个自定义的中间件函数，用来验证用户的身份
  - 如过验证成果，则继续执行后续的处理函数
  - 否则直接返回错误相应，而不会执行后续路由处理函数

**注册一个$PUT$路由，并调用$changePassword()$来进行密码修改。其中的$container.GetDB()$更详细解释**

- $container.GetDB()$ 是从$container$对象中获取数据库链接的方法，通常,$container$是一个依赖注入容器，用于管理应用程序中的各种资源（如数据库连接、配置等）。

- **在代码中的作用**：将数据库连接传递给 $login$ 和 $changePassword$函数，以便它们可以操作数据库。



### 登录逻辑实现

用一个$struct$存用户名和密码信息。

之后以用户名为关键信息在数据库中查找，如果查找失败则返回失败结果。

若查找成功则和密码库里的哈希值比较而非明文密码。

如果验证密码成功之后就生成一个包含用户名和$Token$的过期时间的$Token$，随后返回$Token$。

```go
func login(c *gin.Context, db *gorm.DB) {
    
	var dto web_model.NexusUserDTO
	if err := c.ShouldBindJSON(&dto); err != nil {
		web_model.ResponseError(c, http.StatusInternalServerError, "Failed to parse user json :"+err.Error())
		return
	}
```

- **`c.ShouldBindJSON(&dto)`**：将请求中的 $JSON$ 数据绑定到 $dto$ 变量（$NexusUserDTO$ 结构体）。
- **`web_model.ResponseError`**：返回错误响应，包含状态码和错误信息。



在$web_model.NexusUserDTO$中
```go
type NexusUserDTO struct {
	LoginName string `json:"loginName" example:"192.168.1.1"`
	Password  string `json:"password" example:"192.168.1.1"`
}
```

(\`...`)这样在结构体字段后的字符串是**结构体标签**

格式为 

```go
`key1:"value1" key2:"value2" ...`
```

每个键值对用空格分隔



```go
// 在数据库中查找用户	
var dbUser model.User
	if err := db.Where("login_name = ?", dto.LoginName).First(&dbUser).Error; err != nil {
		web_model.ResponseError(c, http.StatusPaymentRequired, "账号不存在")
		return
	}

```

- **`db.Where("login_name = ?", dto.LoginName).First(&dbUser)`**：在数据库中找到 `login_name` 匹配的用户记录。
- **`web_model.ResponseError`**：如果用户不存在，返回错误响应。



```go
// 验证密码	
if err := bcrypt.CompareHashAndPassword([]byte(dbUser.Password), []byte(dto.Password)); err != nil {
		web_model.ResponseError(c, http.StatusPaymentRequired, "密码错误")
		return
	}
```

- **`bcrypt.CompareHashAndPassword`**：对比用户输入的密码和数据库中的哈希密码。如果验证失败，返回错误。



```go
// 验证通过，生成 JWT Token	
expirationTime := time.Now().Add(24 * time.Hour)// Token 有效期为 24 小时
	claims := &web_auth.Claims{
		LoginName: dbUser.LoginName,
		StandardClaims: jwt.StandardClaims{
			ExpiresAt: expirationTime.Unix(),
		},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	tokenString, err := token.SignedString(web_auth.JwtKey)//生成Token
	if err != nil {
		web_model.ResponseError(c, http.StatusInternalServerError, "Failed to generate token")
		return
	}

```

- 生成 $JWT\ \ Token：$
  - **`claims`**：包含用户的登录名和 $Token$ 的过期时间。
  - **`jwt.NewWithClaims`**：创建一个新的 $JWT\ \ Token$。
  - **`token.SignedString`**：使用密钥对 $Token$ 进行签名。
- **`web_model.ResponseError`**：如果生成 $Token$ 失败，返回错误。



**？？签名是什么意思**

$JWT Token$ 由三部分组成，用 `.` 分隔：

- **Header**（头部）：描述 $Token$ 的元信息，如签名算法（如 $HS256$）。

- **Payload**（载荷）：包含用户信息和其他数据（如登录名、过期时间）。

- **Signature**（签名）：对 $Header$ 和 $Payload$ 的加密签名。

首先将头部信息和载荷分别编码，之后用`.`来连接。

随后使用密钥和加密算法对以上字符串进行加密，此时生成的字符串就是签名结果。

最后将这三个部分用`.`来链接就生成最终$Token$了。

**密钥不会被传送**。在 $HMAC-SHA256$ 签名算法中，**密钥是保密的**，只存在于 **服务器端**（或生成 $Token$ 的一方）。客户端只会收到 $JWT Token$（包含 $Header$、$Payload$ 和 $Signature$），但不会知道密钥。



```go
// 返回成功响应	
web_model.ResponseSuccess(c, gin.H{"token": tokenString})

}
```

- **`web_model.ResponseSuccess`**：返回成功响应，包含生成的 $Token$。



### 修改密码逻辑实现

首先解析请求中的$JSON$数据，用一个结果提来存修改后的用户密码和用户名

之后以用户名为关键字在数据库中查找用户，如果查找成功则验证旧密码。

验证旧密码成功之后生成新密码的哈希值，并更新新密码的哈希值。

```go
func changePassword(c *gin.Context, db *gorm.DB) {
	var changePasswordRequest web_model.NexusPwdUpdateDTO//修改后的用户信息
	if err := c.ShouldBindJSON(&changePasswordRequest); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
```

- **`c.ShouldBindJSON(&changePasswordRequest)`**：将请求中的 $JSON$ 数据绑定到 $changePasswordRequest$ 变量。



```go
	var dbUser model.User
	if err := db.Where("login_name = ?", c.GetString("username")).First(&dbUser).Error; err != nil {
		web_model.ResponseError(c, http.StatusNotFound, "用户不存在")
		return
	}
```

- **`db.Where("login_name = ?", c.GetString("username")).First(&dbUser)`**：在数据库中找到当前登录用户的记录。



```go
	if err := bcrypt.CompareHashAndPassword([]byte(dbUser.Password), []byte(changePasswordRequest.Password)); err != nil {
		web_model.ResponseError(c, http.StatusForbidden, "旧密码验证错误")
		return
	}
```

- **`bcrypt.CompareHashAndPassword`**：验证用户输入的旧密码。



```go
	newPasswordHash, err := model.HashPassword(changePasswordRequest.NewPassword)
	if err != nil {
		web_model.ResponseError(c, http.StatusInternalServerError, "密码生成失败")
		return
	}
```

- **`model.HashPassword`**：生成新密码的哈希值。



```go
	if err := db.Model(&dbUser).Update("password", newPasswordHash).Error; err != nil {
		web_model.ResponseError(c, http.StatusInternalServerError, "密码更新失败")
		return
	}
```

- **`db.Model(&dbUser).Update("password", newPasswordHash)`**：在数据库中更新用户密码。



```go
	web_model.ResponseSuccess(c, gin.H{})
}
```

- **`web_model.ResponseSuccess`**：返回成功响应。





## HTTP相关

- $HTTP$ 是一种用作获取诸如 $HTML$ 文档这类资源的协议。它是 $Web$ 上进行任何数据交换的基础，同时，也是一种客户端—服务器$（client-server）$协议，也就是说，请求是由接受方——通常是 $Web$ 浏览器——发起的。完整网页文档通常由文本、布局描述、图片、视频、脚本等资源构成。

- 客户端向服务器请求所需要的资源，每一个资源都是独立的，如图片，视频，音频等。

- 是一种应用层协议，通过$TCP$或$TLS$（一种加密的$TCP$）来传输。

### HTTP请求方法

- $GET:$ 

  - 从服务器**获取资源**
  - 请求的参数通常附加在$URL$中
  - 不修改服务器资源

- $POST:$ 

  - 向服务器提交数据，通常用来**创建新资源**
  - 请求数据通常包含在请求体中
  - 可能会修改服务器资源

- $PUT:$ 

  - 更新服务器上的资源，通常用于**替换整个资源**
  - 请求数据通常包含在请求体中
  - 可能会修改服务器资源

- $DELETE:$ 

  - **删除服务器上的资源**
  - 会修改服务器资源

- $PATCH:$ 

  - **对资源进行部分更新**
  - 请求数据通常包含在请求体中，但只需包含需要更新的字段
  - 会修改服务器资源

  

## Git用法

分布式版本控制系统。

- 是存储文件的仓库，集中存储在服务器

- 并不是单纯的存储，有版本管理的功能。可以记录每一次修改历史，用户可翻看历史版本或还原文件。

提交文件到暂   存区，是指选定那些被修改文件为此次上传内容

```git
//本地仓库操作
--------------------------------------------------
cd [目标仓库副本目录]			//切换当前的工作目录
git status					//查看被修改的文件列表  
git add [文件1] [文件2]		//提交文件到暂存区
git add *					//提交全部文件到缓存区
git diff [文件]				//查看具体文件修改内容
git rm <name>				//取消文件跟踪
git diff					//查看文件哪里被修改
git log						//查看历史提交
---------------------------------------------------
//远程仓库操作
---------------------------------------------------
git remote add [远程仓库名] [远程仓库地址]//链接远程仓库
git push [远程仓库名字]//将本地代码推送到远程仓库
---------------------------------------------------
```



## 依赖注入

### **是什么**

- 是一种软件设计模式**$DI$ **,就是代码的一种写法，这样写可以使得代码更加容易维护。

- 实例 $A$ 的创建，依赖于实例 $B$ 的创建，且在实例 $A$ 的生命周期内，持有对实例 $B$ 的访问权限。
- **将对象的创建和依赖关系交给外部容器来管理**，而不是在代码中硬编码。
- **提高代码的可测试性和可维护性**，因为依赖关系可以被动态替换或配置。

**依赖**：一个对象需要的其他对象或服务。例如，一个服务类可能需要一个数据库连接对象来执行操作。

**注入**：将依赖对象从外部传递到需要它的对象中，而不是由需要它的对象自己创建。

### **为什么要用**

**不使用依赖注入风险：**

- 全局变量十分不安全，存在覆写的可能

- 资源散落在各处，可能重复创建，浪费内存，后续维护能力极差

- 提高循环依赖的风险

- 全局变量的引入提高单元测试的成本

**依赖注入的主要目的是解决代码的紧耦合问题，它带来的好处包括：**

- 解耦：通过将依赖关系从内部实现中移出，代码更加模块化，易于维护和扩展。

- 可测试性：依赖注入使得单元测试更加容易。可以在测试中使用模拟对象$（Mock）$替代真实的依赖对象，从而隔离被测代码。

- 灵活性：可以轻松替换依赖的实现。例如，可以将数据库从 $MySQL$ 切换到 $PostgreSQL$，而无需修改 $Service$ 的代码。

- 可重用性：通过抽象依赖关系，代码可以在不同的场景中重用。例如，$Service$可以被多个模块使用，而无需为每个模块单独实现依赖。



