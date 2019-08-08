---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - HTTP API
---

# 一. 简介
HTTP接口是互联网各系统之间对接的重要方式之一，使用HTTP接口开发和调用都很方便，也是被大量采用的方式，它可以让不同系统之间实现数据的交换和共享。  
由于HTTP接口开放在互联网上，所以我们就需要有一定的安全措施来保证接口安全。

HTTP API接口安全性演进如下  
1. HTTP + 完全开放 （毫无安全可言）
2. HTTP + 参数签名 （基本安全）
3. HTTP + 私钥签名公钥验签  (安全性高)
4. HTTPS + 参数签名  (安全性更高)
5. HTTPS + 私钥签名公钥验签  (最安全)

总的来说HTTP接口安全主要靠以下3点  
1. 使用HTTPS代替HTTP，因为HTTP是明文传输，HTTPS使用加密传输，HTTPS代替HTTP一般是通过运维手段来实现
2. 参数签名，客户端和服务端事先约定好密钥，客户端使用密钥签名，服务端使用相同算法计算签名进行校验
3. 客户端使用私钥签名，服务端使用公钥验证签名

无论是使用参数签名法还是使用私钥签名公钥验签法，都是为了确认请求参数没有被篡改，保证请求是安全的。  
这篇文章主要介绍一下这2种方法的简单实现。

# 二. 参数签名
参数签名一般指的是使用hash算法对请求参数计算得到签名。

客户端计算签名步骤如下  
1. 客户端和服务端事先约定 密钥 secret 和盐 salt，密钥和盐一般是16或32位字符串
2. 客户端计算一个 `sig_timestamp`
3. 请求所有参数和 `sig_timestamp`，拼接成一个字符串并按字典序排序，记为data
4. `url path` + `data` + `secret` + `salt`，拼接成最终字符串`s`
5. 对`s`计算一个md5即为签名结果
6. 客户端请求的时候把参数传给服务端 `sig_timestamp={sig_timestamp}&signature={signature}`

服务端验证签名
1. 解析参数获取`sig_timestamp`和`signature`2个字段
2. 确认`sig_timestamp`是否小于当前时间，如果是说明签名已经过期了
3. 按照相同的算法计算一遍签名，比较计算的结果和signature是否一致，如果是验证通过

简单的代码如下，完整的代码可以参考 [简单的http签名算法](https://chenguolin.github.io/2017/07/25/HTTP-API-1-%E7%AE%80%E5%8D%95%E7%9A%84http%E7%AD%BE%E5%90%8D%E7%AE%97%E6%B3%95/)
```
// 1. url path
path := "http://localhost:8080/user/select"

// 2. sigTimestamp
sigTimestamp := "1550327907"

// 3. sort form values
params := make([]string, 0, 10)
params = append(params, sigTime)
// data form map[string]string
for k, v := range form {
    params = append(params, v)
}
sort.Strings(params)

// 3. calculator signature
// combine path + params + secret + salt
// - secret: 1234567890abcdef
// - salt: 0987654321abcdef
str := path + strings.Join(params, "") + secret + salt

// md5
sig := fmt.Sprintf("%x", md5.Sum([]byte(str)))
```

# 三. 私钥签名公钥验签
参数签名虽然能够用来确认参数是否有被篡改，但是存在一个问题`密钥和盐需要分别存储在客户端和服务端，如果泄露那算法很容易被破解`。  
所以为了解决这个问题，可以引入业界比较出名的`非对称加密公私钥对，私钥签名公钥验证签名`，需要事先生成公私钥对。

`由于私钥只有客户端持有，服务端只需要拿客户端上传的公钥即可，只要私钥不泄露任何人都没有办法篡改签名，所以安全性最高。`

客户端计算签名步骤如下  
1. 客户端生成公私钥对
2. 客户端计算一个 `sig_timestamp`
3. 请求所有参数和 `sig_timestamp`，拼接成一个字符串并按字典序排序，记为 `data`
4. 使用私钥对 `data` 进行签名，得到 `sig` 记为签名结果
5. 客户端请求的时候把相关的参数传给服务端 `public_key={public_key}&sig_timestamp={sig_timestamp}&signature={signature}`
   + public_key={public_key}: 公钥
   + sig_timestamp={sig_timestamp}: 签名时间
   + signature={signature}: 签名结果

服务端验证签名
1. 解析参数得到公钥 public_key，sig_timestamp，和签名结果 signature
2. 确认 sig_timestamp 是否小于当前时间，如果是说明签名已经过期了
3. 服务端使用公钥验证签名结果 signature，验证通过说明请求参数没有被篡改

简单的代码如下
```
// GenKeyPair generate private, public key
// - PrivateKey private key
// - PublicKey public key
func GenKeyPair() (PrivateKey, PublicKey, error) {
    var prk *ecdsa.PrivateKey
    var puk ecdsa.PublicKey
    var curve elliptic.Curve

    curve = elliptic.P256()
    prk, err := ecdsa.GenerateKey(curve, rand.Reader)
    if err != nil {
	return PrivateKey(""), PublicKey(""), err
    }
    puk = prk.PublicKey

    // base64 encode
    prvKey := PrivateKey(base64.StdEncoding.EncodeToString(prk.D.Bytes()))
    pubXKey := PublicKey(base64.StdEncoding.EncodeToString(puk.X.Bytes()))
    pubYKey := PublicKey(base64.StdEncoding.EncodeToString(puk.Y.Bytes()))

    return prvKey, pubXKey + pubYKey, nil
}

// GenSignatureByPriKey gen signature by private key
// @prvKey private key
// @data content data
//
// - Signature signature value
func GenSignatureByPriKey(prvKey PrivateKey, data []byte) (Signature, error) {
    // base64 decode
    prvBytes, err := base64.StdEncoding.DecodeString(string(prvKey))
    if err != nil {
	return Signature(""), err
    }

    // new ecdsa.PublicKey
    pubk := ecdsa.PublicKey{
	Curve: elliptic.P256(),
    }
    prvk := &ecdsa.PrivateKey{
	PublicKey: pubk,
	D:         fromBase10(string(prvBytes)),
    }

    // sign
    r, s, err := ecdsa.Sign(rand.Reader, prvk, data)
    if err != nil {
	return Signature(""), err
    }

    rt, err := r.MarshalText()
    if err != nil {
  	return Signature(""), err
    }
    st, err := s.MarshalText()
    if err != nil {
	return Signature(""), err
    }

    // base64 encode
    rtBase64 := base64.StdEncoding.EncodeToString(rt)
    stBase64 := base64.StdEncoding.EncodeToString(st)

    return Signature(rtBase64 + stBase64), nil
}

// VerifySignatureByPubKey verify signature by public key
// @pub public key
// @sig signature
// @data content data
func VerifySignatureByPubKey(pub PublicKey, sig Signature, data []byte) error {
    // pub = pubx + puby
    if len(pub) != 88 {
	return errors.New("VerifySignatureByPubKey invalid public key")
    }
    pubx, err := base64.StdEncoding.DecodeString(string(pub[:44]))
    if err != nil {
	return err
    }
    puby, err := base64.StdEncoding.DecodeString(string(pub[44:]))
    if err != nil {
	return err
    }

    // new ecdsa.PrivateKey
    pubk := ecdsa.PublicKey{
	Curve: elliptic.P256(),
	X:     fromBase10(string(pubx)),
	Y:     fromBase10(string(puby)),
    }

    // parse sig get r, s
    if len(sig) != 208 {
	return errors.New("VerifySignatureByPubKey invalid signature")
    }
    r, err := base64.StdEncoding.DecodeString(string(sig[:104]))
    if err != nil {
	return err
    }
    s, err := base64.StdEncoding.DecodeString(string(sig[104:]))
    if err != nil {
	return err
    }

    // verify signature
    if ecdsa.Verify(&pubk, data, fromBase10(string(r)), fromBase10(string(s))) {
	return errors.New("VerifySignatureByPubKey ecdsa.Verify failed ~")
    }

    return nil
}

func fromBase10(base10 string) *big.Int {
    i, ok := new(big.Int).SetString(base10, 10)
    if !ok {
	// TODO print error log
	// TODO default return 0
	return big.NewInt(0)
    }

    return i
}
```

