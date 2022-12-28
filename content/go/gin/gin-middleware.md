---
title: "gin 中间件的使用"
tags: 
- GO Gin Middleware
weight: 101
---

## 1.问题
如何使用Go的中间件呢？

## 2. middleware 语法
```go
    func xxx() gin.HandlerFunc {
        return func(c *gin.Context) {

        }
    }
```
    
## 3.Go JWT Middleware
```go
func JWT() gin.HandlerFunc {
	return func(c *gin.Context) {
		authorization := c.Request.Header.Get("Authorization")
		if authorization == "" {
			c.AbortWithStatus(401)
			return
		}
		token := strings.Split(authorization, " ")
		if len(token) <= 1 && token[0] == "Bearer" {
			c.AbortWithStatus(401)
			return
		}
		if len(token) == 2 && token[1] == "" {
			c.AbortWithStatus(401)
			return
		}

		var tokenType string
		if strings.Contains(c.Request.URL.Path, "backend") {
			tokenType = "backend"
		} else if strings.Contains(c.Request.URL.Path, "frontend") {
			tokenType = "frontend"
		} else {
			c.AbortWithStatus(401)
			return
		}
		jwt := utils.NewJWT(conf.Get(), tokenType)
		claims, err := jwt.ParseToken(token[1])

		if err != nil {
			responder.BadRequest(c, err)
			c.Abort()
			return
		}
		if claims.Type != tokenType {
			c.AbortWithStatus(401)
			return
		}

		c.Set("claims", claims)
		c.Next()
	}
}
```