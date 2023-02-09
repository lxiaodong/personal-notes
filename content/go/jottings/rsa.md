---
title: "Rsa 解密"
tags: 
- php
- go
- rsa
- 随笔
weight: 2
description: "Rsa 解密"
---

## 1.问题
如何用Go实现与PHP一样的RSA加密？
    
## 2.PHP RSA 加密
```php
class Rsa {

	private $pubKey;

	public function __construct($pubKey = null)
	{
		$this->pubKey = $this->convertKey($pubKey)
	}

	//RSA公钥加密
	public function publicEncrypt($data)
	{
		$encrypted = '';
		$pubkey = openssl_pkey_get_public($this->pubKey);
		$plainData = str_split($data, 100);
		foreach ($plainData as $chunk) {
			$partialEncrypted = '';
			$encryptionOk = openssl_public_encrypt($chunk, $partialEncrypted, $pubkey);//公钥加密
			if ($encryptionOk === false) {
				return false;
			}
			$encrypted .= $partialEncrypted;
		}
		$encrypted = base64_encode($encrypted);
		return $encrypted;
	}

	private function convertKey($pubKey) {
		return "-----BEGIN PUBLIC KEY-----\n" .
			wordwrap($pubKey, 64, "\n", true) .
			"\n-----END PUBLIC KEY-----";
	}
}

```

## 3.GO RSA 解密
```go
package utils

import (
	"crypto/rand"
	"crypto/rsa"
	"crypto/x509"
	"encoding/base64"
	"encoding/pem"
	"errors"
	"fmt"

	"ijiwei.com/project-hive/internal/pkg/conf"
)

type ursa struct {
	PublicKey string
}

func NewURsa() *ursa {
	return &ursa{
		// 获取公钥
		PublicKey: conf.Get().OaService.PublicKey,
	}
}

func (r *ursa) Encrypt(data string) string {
	// pem包实现了PEM数据编码（源自保密增强邮件协议）
	// Decode函数会从输入里查找到下一个PEM格式的块（证书、私钥等）。它返回解码得到的Block和剩余未解码的数据。如果未发现PEM数据，返回(nil, data)。
	// type Block struct {
	// 	Type    string            // 得自前言的类型（如"RSA PRIVATE KEY"）
	// 	Headers map[string]string // 可选的头项
	// 	Bytes   []byte            // 内容解码后的数据，一般是DER编码的ASN.1结构
	// }
	block, _ := pem.Decode([]byte(r.PublicKey))

	if block == nil {
		panic(errors.New("public key error"))
	}

	// ParsePKIXPublicKey解析一个DER编码的公钥。这些公钥一般在以"BEGIN PUBLIC KEY"出现的PEM块中
	pubInterface, err := x509.ParsePKIXPublicKey(block.Bytes)
	if err != nil {
		fmt.Println("ParsePKIXPublicKey err", err)
		panic(err)
	}

	// rsa包实现了PKCS#1规定的RSA加密算法。
	pub := pubInterface.(*rsa.PublicKey)
	
	// func EncryptPKCS1v15(rand io.Reader, pub *PublicKey, msg []byte) (out []byte, err error)
	// EncryptPKCS1v15使用PKCS#1 v1.5规定的填充方案和RSA算法加密msg。信息不能超过((公共模数的长度)-11)字节。注意：使用本函数加密明文（而不是会话密钥）是危险的，请尽量在新协议中使用RSA OAEP
	signature, _ := rsa.EncryptPKCS1v15(rand.Reader, pub, []byte(data))

	// var StdEncoding = NewEncoding(encodeStd) //RFC 4648定义的标准base64编码字符集。
	// func (enc *Encoding) EncodeToString(src []byte) string // 返回将src编码后的字符串。
	return base64.StdEncoding.EncodeToString(signature)
}

```