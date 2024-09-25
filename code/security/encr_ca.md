# CA数字证书

我们都知道加密通信HTTPS协议是HTTP协议基本之上再加上SSL/TLS的协议，他用非对称加密方式确定通信过程中的对称加密密钥。这个过程用到了CA证书，由CA证书提供的信任链，来验证通信对方的身份，从而确保数据传输的安全性。

CA证书是一种数字证书，由证书颁发机构（Certificate Authority, CA）签发，用于验证公开密钥的所有者的身份。CA证书是公钥基础设施（Public Key Infrastructure, PKI）的核心组成部分，广泛用于确保网络通信的安全性。

### 数字证书

数字证书(Digital Certificate)，也称公钥证书，用非对称加密算法，具有私钥和公钥。私钥主要用于签名和解密，由用户自定义，只有用户自己知道；公钥用于签名验证和加密，可被多个用户共享。

它具有安全、唯一、便利等特征，主要作为一种用于验证身份的电子文件，在网络通信中起到类似于身份证或护照的作用。广泛的用于HTTPS、邮件、FTPS、VPN、其他使用SSL协议的网络服务中。

数字证书受 X.509 格式的技术规范约束，X.509 V3规定数字证书由三个部分组成：

- 证书内容，包含证书持有者信息，公钥等等；包含签署人信息、证书颁发机构信息、证书有效期、签署人公钥信息等，用于证明签署人身份。

- 签名算法，包含哈希摘要算法和公钥算法。

- 证书签名值，使用签名算法对证书内容进行签名后的结果值，用于验证证书的有效性。

主要的生成数字证书的方式有：

- 由可信的证书颁发机构（CA）签发，这是最重要最权威的生成途径，也叫CA证书。
- 由证书持有者自己签发，自己使用OpenSSL等工具生成，一般用于测试或开发。
- 企业内部的CA。

### CA

权威的数字证书就是CA发行，CA（Certificate Authority）证书颁发机构，也叫“证书授权中心”。它是负责管理和签发证书的第三方机构，作用是检查证书持有者身份的合法性，并签发证书，以防证书被伪造或篡改。

CA在公钥基础设施（PKI）中扮演着核心角色，在PKI中，CA通常有层级结构，包括根CA和中间CA。

- 根CA（Root CA）是信任链的起点，其证书自签名并预装在操作系统和浏览器中，通常用于签发中间CA证书。如DigiCert、GlobalSign等。
- 中间CA，由根CA签发，负责签发最终用户证书。

CA签发过程大致如下：

1. **生成密钥对**：申请者生成一对公私钥。
2. **创建CSR（Certificate Signing Request）**：申请者创建一个包含其公钥和其他身份信息的证书签名请求（CSR），并将其发送给CA。
3. **验证身份**：CA验证申请者的身份，可能涉及域名验证、组织验证等步骤。
4. **签发证书**：CA使用其私钥对CSR进行签名，生成数字证书，并将其返回给申请者。

CA提供多种类型的数字证书，如DV SSL证书，OV SSL证书和EV SSL证书。

### 证书链及信任机制

要使颁发的证书可信，颁发 CA 必须可信。 CA 通过证书链建立信任。证书链是由一系列 CA 证书发出的证书序列，最终以根 CA 证书结束，通过证书链建立信任链。

1. **根CA的信任**：操作系统和浏览器预装了多个公认的根CA证书，这些根CA被视为可信任的起点。
2. **中间CA的信任链**：根CA签发中间CA证书，中间CA再签发最终用户证书，形成信任链。
3. **证书验证**：用户端使用根CA的公钥逐级验证证书签名，确保每一级证书都是由上一级CA签发的，最终验证到达根CA。

根CA的私钥高度保密，通常用于签发中间CA证书，而不直接签发最终用户证书。中间CA由根CA签发，负责签发最终用户证书，分担根CA的负担，提高系统的灵活性和安全性。