---
title: "解析openssl_encrypt加密数据"
tags: 
- PHP openssl_encrypt
- go aes ecb
weight: 1
---

## 1.问题
如何用Go解析PHP openssl_encrypt加密的数据？

```text
请求参数：
ts=1665713553&token=8D496872B950D6B6B72C797A4D5BE779A2701A131937192244ADBD53FF7C9F830D631D6C447D2A163BA0482FB6401DA9&loginid=wangjh
```
    
## 2.PHP openssl_encrypt 加密
```php
class Aes {
    const KEY = 'L1KJEwNL2fruhPtWdl9A2S';
    const SALT = 'L1K3HR2iveYmUlCV8tOA2S';

    //验证签名
    public function verifySignature($loginid, $ts, $token){
        if((!$loginid) ||(!$ts) ||(!$token)){
            return false;
        }

        $time = time();
        $ts = $this->getSecondTime($ts);
        $key = substr(openssl_digest(openssl_digest(self::KEY, 'sha1',true), 'sha1',true), 0, 16);
        $encrypt = openssl_encrypt($loginid.'|'.$ts.'|'.self::SALT, 'aes-128-ecb', $key, OPENSSL_RAW_DATA, self::SALT);
        $encrypt = strtoupper(bin2hex($encrypt));
        
        if($encrypt != $token){
            return false;
        }
        return true;
    }

    private function getSecondTime($time){
        if(strlen($time)>10){
            return intval($time/1000);
        }
        return $time;
    }
}
```

## 3.GO aes ecb 解密

###### 3.1 token解密文件
```go
package oa

import (
	"bytes"
	"crypto/aes"
	"crypto/sha1"
	"encoding/hex"
	"fmt"
	"io"
	"strconv"
	"strings"

	"ijiwei.com/project-hive/internal/pkg/utils"
)

const (
	key  = "L1KJEwNL2fruhPtWdl9A2S"
	salt = "L1K3HR2iveYmUlCV8tOA2S"
)

var (
	// 处理时间匿名函数
	dealTs = func(ts int) string {
		timestamp := strconv.Itoa(ts)
		if len(timestamp) > 10 {
			ts, _ = strconv.Atoi(timestamp)
			ts = ts / 1000
		}
		return strconv.Itoa(ts)
	}
	// 对应PHP substr(openssl_digest(openssl_digest($key, 'sha1',true), 'sha1',true), 0,16);
	sha1Func = func(str []byte) []byte {
		sha1Handler := sha1.New()
		io.Copy(sha1Handler, bytes.NewBuffer(str))
		return sha1Handler.Sum(nil)
	}
)

// 用户授权请求参数
type OaUserAccessToken struct {
	Ts      int    `form:"ts" binding:"required"`
	Token   string `form:"token" binding:"required"`
	LoginId string `form:"loginid" binding:"required"`
}

func (o *OaUserAccessToken) CheckToken() bool {
	if o.Ts == 0 || o.Token == "" || o.LoginId == "" {
		return false
	}
	ts := dealTs(o.Ts)
	token := o.Token
	id := o.LoginId
	sha1Key := sha1Func(sha1Func([]byte(key)))
	enStr := id + "|" + ts + "|" + salt

	// New Cipher
	block, _ := aes.NewCipher(sha1Key[0:16])
	//判断加密快的大小
	blockSize := block.BlockSize()
	//填充
	encryptBytes := padding([]byte(enStr), blockSize)
	//初始化加密数据接收切片
	crypted := make([]byte, len(encryptBytes))
	blockMode := utils.NewECBEncrypter(block)
	blockMode.CryptBlocks(crypted, encryptBytes)
	newToken := strings.ToUpper(hex.EncodeToString(crypted))
	fmt.Println(token)
	fmt.Println(newToken)
	return token == newToken
}

func padding(ciphertext []byte, blockSize int) []byte {
	padding := blockSize - len(ciphertext)%blockSize
	padtext := bytes.Repeat([]byte{byte(padding)}, padding)
	return append(ciphertext, padtext...)
}
```

###### 3.2 aes ecb 解密算法
```go
package utils

import "crypto/cipher"

type ecb struct {
	b         cipher.Block
	blockSize int
}

func newECB(b cipher.Block) *ecb {
	return &ecb{
		b:         b,
		blockSize: b.BlockSize(),
	}
}

type ecbEncrypter ecb

func NewECBEncrypter(b cipher.Block) cipher.BlockMode {
	return (*ecbEncrypter)(newECB(b))
}

func (x *ecbEncrypter) BlockSize() int { return x.blockSize }

func (x *ecbEncrypter) CryptBlocks(dst, src []byte) {
	if len(src)%x.blockSize != 0 {
		panic("crypto/cipher: input not full blocks")
	}
	if len(dst) < len(src) {
		panic("crypto/cipher: output smaller than input")
	}
	for len(src) > 0 {
		x.b.Encrypt(dst, src[:x.blockSize])
		src = src[x.blockSize:]
		dst = dst[x.blockSize:]
	}
}

type ecbDecrypter ecb

func NewECBDecrypter(b cipher.Block) cipher.BlockMode {
	return (*ecbDecrypter)(newECB(b))
}
func (x *ecbDecrypter) BlockSize() int { return x.blockSize }
func (x *ecbDecrypter) CryptBlocks(dst, src []byte) {
	if len(src)%x.blockSize != 0 {
		panic("crypto/cipher: input not full blocks")
	}
	if len(dst) < len(src) {
		panic("crypto/cipher: output smaller than input")
	}
	for len(src) > 0 {
		x.b.Decrypt(dst, src[:x.blockSize])
		src = src[x.blockSize:]
		dst = dst[x.blockSize:]
	}
}
```