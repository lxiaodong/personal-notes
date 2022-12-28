---
title: "jwt包的封装"
tags: 
- GO JWT
weight: 3
---

## 1.问题
jwt的使用方式？

## 2.Go JWT
```go
package utils

import (
	"time"

	"github.com/golang-jwt/jwt/v4"
	"ijiwei.com/project-hive/internal/pkg/conf"
)

type JWT struct {
	SigningKey []byte
	jwt.RegisteredClaims
	Account   string
	UserId    int
	Nickname  string
	Type      string
	ExpiresAt int64
}

var (
	cf          = conf.Get()
	BackendJwt  = NewJWT(cf, "backend")
	FrontendJwt = NewJWT(cf, "frontend")
)

func NewJWT(conf *conf.Config, tokenType string) *JWT {
	return &JWT{
		SigningKey: []byte(conf.JWT.Signingkey),
		RegisteredClaims: jwt.RegisteredClaims{
			Issuer: conf.JWT.Issuer,
		},
		Type:      tokenType,
		ExpiresAt: conf.JWT.ExpiresTime * int64(time.Hour),
	}
}

func (j *JWT) CreateToken(account, nickname string, uid int) (string, error) {
	j.RegisteredClaims.ExpiresAt = jwt.NewNumericDate(time.Now().Add(time.Duration(j.ExpiresAt)))
	j.RegisteredClaims.NotBefore = jwt.NewNumericDate(time.Now())
	j.Account = account
	j.Nickname = nickname
	j.UserId = uid
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, j)
	return token.SignedString(j.SigningKey)
}

func (j *JWT) ParseToken(tokenString string) (*JWT, error) {
	token, err := jwt.ParseWithClaims(tokenString, &JWT{}, func(t *jwt.Token) (interface{}, error) {
		return j.SigningKey, nil
	})
	if err != nil {
		if ve, ok := err.(*jwt.ValidationError); ok {
			if ve.Errors&jwt.ValidationErrorMalformed != 0 {
				return nil, jwt.ErrTokenMalformed
			} else if ve.Errors&jwt.ValidationErrorExpired != 0 {
				// Token is expired
				return nil, jwt.ErrTokenExpired
			} else if ve.Errors&jwt.ValidationErrorNotValidYet != 0 {
				return nil, jwt.ErrTokenNotValidYet
			} else {
				return nil, jwt.ErrTokenNotValidYet
			}
		}
	}
	if token != nil {
		if claims, ok := token.Claims.(*JWT); ok && token.Valid {
			return claims, nil
		}
		return nil, jwt.ErrTokenNotValidYet

	} else {
		return nil, jwt.ErrTokenNotValidYet
	}
}

```