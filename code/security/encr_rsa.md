# 常用加密方式-RSA非对称加密算法

RSA加密算法是一种非对称加密算法，是目前最常用的公钥加密算法，被ISO推荐为公钥数据加密标准。是1977年由Ron Rivest，Adi Shamir，Leonard Adleman在麻省理工学院工作时一起提出的。

> 非对称加密算法，就是指加密和解密使用不同的密钥，即使用加密密钥进行加密、解密密钥进行解密。
>
> 之所以也叫公钥加密算法，因为使用时把其中加密密钥（公钥）公开，而解密密钥（私钥）保密（也可以私钥加密公钥解密）。
>
> 常见的公钥密码算法有：RSA、ECC、ElGamal、Diffie-Hellman、ECDH、ECDSA 和 EdDSA等等。

它的使用场景非常多，主要有三大用途：加密与解密、签名与验证、密钥交换。比较常见在HTTPS、SSH等协议中都使用了RSA算法来加密通信过程中的数据（协商会话密钥的过程），数字证书也使用RSA算法。

RSA 可以使用不同长度的密钥：1024、2048、3072、4096、8129、16384 甚至更多。不过小于2048位的密钥等于现在被破解的风险。但是越长的密钥，会消耗更多的计算时间。

RSA加密明文最大长度117字节，解密要求密文最大长度为128字节，所以在加密和解密的过程中需要分块进行。

### 算法的原理

非对称加密算法的核心设想就是，加密和解密使用不同规则，但是密文与明文间可以通过各自的计算逻辑转换。在数论中，用欧拉函数对两个互质数进行计算得到几个数字即可进行密文明文之间的转换。

> - 质数：若一个数 𝑝(𝑝>1) 满足它的因子只有 1 和它本身，那么我们称这个数为素数（质数）。
>
> - 互质：如果两个正整数，除了1以外，没有其他公因子，我们就称这两个数是互质关系。
> - 欧拉函数：任意给定正整数n，请问在小于等于n的正整数之中，有多少个与n构成互质关系，这个计算公式计算欧拉函数。

加密公式为：m^e ≡ c (mod N)。

解密公式为：c^d ≡ m (mod N)。

> - 模运算：mod（%），整数被一个正整数除时的余数。两个整数被同一个正整数除时的余数相同则称为同余数。

其中公钥就是（N，e），而私钥就是（N，d），m为明文，c为密文。

关于n，e，d这几个数字计算计算过程：

两个互质数P1、P2，计算出两互质数的乘积N，用欧拉函数计算出φ(N)，再1到φ(N)之间随机选择一个整数且与φ(N)互质的e，再计算e对于φ(n)的模反元素d，这需要使用"扩展欧几里得算法"来求解d。

> 这个较大的数N就是RSA算法的可靠性的关键，因为由它推导出d极其困难。
>
> 这一系列证明过程用到的数学知识有：求模运算、最大公约数、扩展欧几里得算法、中国余数定理、欧拉函数、费马小定理等等。

### 算法举例

A和B要相互通信：

1. A随机取大质数P1=53，P2=59，那N=53*59=3127，φ(N)=3016
2. 取一个e=3，计算出d=2011。
3. 只将N=3127，e=3 作为公钥传给B（公钥公开）
4. 假设B需要加密的明文m=89，c = 89^3 mod 3127=1394，于是B传回c=1394。 （公钥加密过程）
5. A使用c^d mod N = 1394^2011 mod 3127，就能得到明文m=89。 （私钥解密过程）

### 密钥的格式

密钥的生成过程使用到了6个数字：

- 两个互质数，**P1**、**P2**。这两个质数越大，就越难破解。
- 两个互质数的乘积**N**，它的长度就是密钥长度。RSA密钥一般是1024位，重要场合则为2048位。
- N的欧拉函数计算结果**φ(N)**。
- 在1到φ(N)之间，随机选择一个整数**e**。RSA常常选择65537。
- 计算e对于φ(n)的模反元素**d**。

将N和e封装成公钥，对外公开，N和d封装成私钥。公钥和私钥的数据都采用ASN.1语法定义，同时加了其他的信息。对ASN.1定义的二进制数据结构，输出就是 DER 格式。

> - ASN.1是一套标准，是描述数据的表示、编码、传输、解码的灵活的记法。它提供了一套正式、无歧义和精确的规则以描述独立于特定计算机硬件的对象结构。
>
> - DER，Distinguished Encoding Rules，可分辩编码规则，内容是二进制串。文件后缀通常为 “.der”。

公钥语法：

```
RSAPublicKey ::= SEQUENCE {
    modulus           INTEGER,  -- n
    publicExponent    INTEGER   -- e
}
```

私钥语法：

```
RSAPrivateKey ::= SEQUENCE {
  version           Version,
  modulus           INTEGER,  -- n
  publicExponent    INTEGER,  -- e
  privateExponent   INTEGER,  -- d
  prime1            INTEGER,  -- p
  prime2            INTEGER,  -- q
  exponent1         INTEGER,  -- d mod (p-1)
  exponent2         INTEGER,  -- d mod (q-1)
  coefficient       INTEGER,  -- (inverse of q) mod p
  otherPrimeInfos   OtherPrimeInfos OPTIONAL
}
```

一般而言，密钥都是通过 PEM 的格式进行存储的，它是对DER的Base64的编码形式，明文格式。

> - PEM，Privacy-Enhanced Mail，隐私增强邮件。PEM格式文件后缀通常为".pem"、“.cer”、“.crt”、“.key”。

在ASN.1语法基础上，RSA制定了一组公钥密码学标准PKCS，PKCS #1 标准是专门为 RSA 密钥进行定义的。

> PKCS，（Public-Key Cryptography Standards，公钥加密标准），是一系列公钥密码学标准，包括了许多标准，涵盖了密钥管理、数字签名、加密算法等多个方面。

PKCS#1 对应的 PEM 格式如下：

```
-----BEGIN RSA PRIVATE KEY-----
BASE64 ENCODED DATA
-----END RSA PRIVATE KEY-----
```

注：PKCS已经公布了15个标准，其编号分别为PCKS#1~15。

> PEM格式的密钥文件的后缀一般情况下都是. pem，下面几个也有不同：
>
> PKCS#8文件的后缀为 .key
> PKCS#12文件的后缀为 .p12, pfx
> X.509文件后缀为.cer, .crt（X.509 是密码学里公钥证书的格式标准）

### OpenSSL生成RSA密钥

1，生成一个RSA密钥

```csharp
openssl genrsa -out private_pkcs1.pem 2048
```

2，从生成的RSA密钥中提取RSA公钥

```
openssl rsa -in private_pkcs1.pem -out public_pkcs1.pem -pubout -RSAPublicKey_out
```
