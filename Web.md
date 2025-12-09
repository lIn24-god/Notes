# Web

## HTTP

1. http.StatusOK
2. http.StatusBadRequest
3. http.StatusInternalServerError
4. http.StatusMethodNotAllowed
5. http.StatusUnauthorized
6. http.StatusForbidden
7. http.ResponseWriter
8. http.Request
9. http.MethodGet
10. http.NewServeMux
11. http.Server
12. http.ListenAndServe

```Go
package main

import (
	"fmt"
	"net/http"
)

// PingHandler 是一个独立封装的处理函数
func PingHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		http.Error(w, "Method Not Allowed", http.StatusMethodNotAllowed)
		return
	} //判断请求方法是否是GET
	w.Write([]byte("pong"))
}

// HelloHandler 返回 "Hello, World!"
func HelloHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello, World!"))
}

func main() {
	// 创建自定义的多路复用器
	mux := http.NewServeMux()

	// 注册路由，使用封装的处理函数
	mux.HandleFunc("/ping", PingHandler)
	mux.HandleFunc("/hello", HelloHandler)

	// 创建自定义的 HTTP 服务器
	server := &http.Server{
		Addr:    ":8080", // 监听地址和端口
		Handler: mux,     // 使用自定义的多路复用器
	}

	// 启动服务器
	fmt.Println("Server is running at http://localhost:8080")
	if err := server.ListenAndServe(); err != nil {
		fmt.Println("Error starting server:", err)
	}
}
```

---

## Gin

1. gin.Default()
2. gin.HandlerFunc {...}
3. gin.H{"a" : "b"......}
4. c *gin.context

    - c.GetHeader
    - c.AbortWithStatusJSON
    - c.Set
    - c.Next
    - c.Request

      1. c.Request.Method
      2. c.Request.URL.Path
      3. c.Request.Header
    - c.JSON
    - c.Get
    - c.Param
    - c.ShouldBindJSON
5. r := gin.Default

    1. r.Use
    2. r.Get
    3. r.Post
    4. r.Run

```Go
package main

import (
	"fmt"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
)

// API密钥验证中间件
func ApiKeyMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		// 从请求头获取API密钥
		apiKey := c.GetHeader("X-API-Key")

		fmt.Printf("🔐 验证API密钥: %s\n", apiKey)

		// 检查API密钥是否存在
		if apiKey == "" {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
				"error":   "缺少API密钥",
				"message": "请在请求头中添加 X-API-Key",
				"hint":    "有效的密钥: blue-mountain-2024",
			})
			fmt.Println("❌ API密钥验证失败: 密钥为空")
			return
		}

		// 验证API密钥（这里使用硬编码验证，实际项目中应该查询数据库）
		if apiKey != "blue-mountain-2024" {
			c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
				"error":   "无效的API密钥",
				"message": "提供的API密钥不正确",
				"hint":    "请联系管理员获取有效密钥",
			})
			fmt.Printf("❌ API密钥验证失败: 无效密钥 '%s'\n", apiKey)
			return
		}

		// 验证通过，继续处理
		fmt.Printf("✅ API密钥验证通过: %s\n", apiKey)

		// 可以在上下文中存储用户信息或其他数据
		c.Set("user", "蓝山工作室成员")
		c.Set("api_key", apiKey)

		c.Next()
	}
}

// 日志中间件
func LoggerMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		fmt.Printf("🌐 %s %s\n", c.Request.Method, c.Request.URL.Path)
		c.Next()
	}
}

func main() {
	r := gin.Default()

	// 注册全局中间件
	r.Use(LoggerMiddleware())

	// 公共路由 - 不需要API密钥
	r.GET("/public", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message":   "这是一个公开接口，无需认证",
			"timestamp": time.Now().Unix(),
		})
	})

	// 受保护的路由 - 需要API密钥
	r.GET("/protected", ApiKeyMiddleware(), func(c *gin.Context) {
		// 从上下文中获取中间件设置的数据
		user, _ := c.Get("user")
		apiKey, _ := c.Get("api_key")

		c.JSON(200, gin.H{
			"message":   "这是一个受保护的接口",
			"user":      user,
			"api_key":   apiKey,
			"data":      "敏感数据在这里",
			"timestamp": time.Now().Unix(),
		})
	})

	// 另一个受保护的路由
	r.GET("/user/profile", ApiKeyMiddleware(), func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message":    "用户个人信息",
			"name":       "张三",
			"role":       "学生",
			"department": "蓝山工作室",
		})
	})

	// 带参数的保护路由
	r.GET("/order/:id", ApiKeyMiddleware(), func(c *gin.Context) {
		orderId := c.Param("id")
		c.JSON(200, gin.H{
			"message":  "订单详情",
			"order_id": orderId,
			"status":   "已完成",
			"amount":   299.99,
		})
	})

	fmt.Println("🚀 服务器启动在 http://localhost:8080")
	fmt.Println("\n测试指南:")
	fmt.Println("1. 公开接口 (无需API密钥):")
	fmt.Println("   GET /public")
	fmt.Println("")
	fmt.Println("2. 受保护接口 (需要API密钥):")
	fmt.Println("   GET /protected")
	fmt.Println("   GET /user/profile")
	fmt.Println("   GET /order/123")
	fmt.Println("")
	fmt.Println("3. 有效API密钥: blue-mountain-2024")
	fmt.Println("")

	r.Run(":8080")
}
```

---

## JWT

1. var jwtSecret []byte("")
2. jwt.RegisteredClaims
3. jwt.NewNumericDate
4. token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)

    1. token.SignedString(jwtSecret)
    2. token.Claims.(*MyClaims)
    3. token.Method.(*jwt.SigningMethodHMAC)
5. jwt.ParseWithClaims(tokenString, &Myclaims{}, func(token *jwt.Token)(interfere{}, error{......}))

```Go
package utils

import (
	"errors"
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("unknown_key")

type MyClaims struct {
	Username string `json:"username"`
	jwt.RegisteredClaims
}

func MakeToken(username string, expirationTime time.Time) (string, error) {
	claims := MyClaims{
		Username: username,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(expirationTime),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
			NotBefore: jwt.NewNumericDate(time.Now()),
			Issuer:    "first_web",
		},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(jwtSecret)
}

func ParseToken(tokenString string) (*MyClaims, error) {
	token, err := jwt.ParseWithClaims(tokenString, &MyClaims{}, func(token *jwt.Token) (interface{}, error) {
		if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("意外的方法: %v", token.Header["alg"])
		}
		return []byte("unknown_key"), nil
	})
	if err != nil {
		return nil, err
	}
	if claims, ok := token.Claims.(*MyClaims); ok && token.Valid {
		return claims, nil
	}
	return nil, errors.New("无效的token")
}
```

---

**乱乱的**

‍
