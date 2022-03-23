
## TLS mTLS 认证入门

### TLS
TLS 是Transport Layer Security的缩写，它的前身是SSL（Secure Sockets Layer）。TLS是一种加密协议，用于保证计算机网络传输的数据安全。

当浏览器想要链接到一个安全网站的时候，例如，https://developer.ibm.com/, 用的就是TLS协议。浏览器之所以能够判断这个链接是否安全，是基于第三方机构CA（certificate authority）颁发的证书，第三方机构给网站签发一个证书，来证明他们拥有这个域名。

浏览器和网站都共同信任这个第三方机构CA，浏览器提前预存了所有可信任网站的证书。当用户要访问某个https的网站时，网站把自己的证书发给浏览器做验证，验证通过就可以访问网站了。在这里浏览器是客户端，网站是服务器端。

典型的建立TLS的过程如下：
  1. 客户端连接服务器端
  2. 服务器发送它的TLS证书
  3. 客户端验证服务器的证书
  4. 客户端和服务器通过加密的TLS连接传输数据

### 双向认证 mTLS
在TLS里，客户端只验证服务器的证书，但是服务器并不验证客户端。大多数时候，服务器并不关心客户端的身份，这大多发生在公开的网站。但是有时候，特别是在企业应用中，需要限制可以访问服务的客户端，这种情况下就需要验证客户端的身份。

mTLS 是Mutual TLS, 意味着客户端和服务器端要相互认证。

双向认证是TLS标准的一部分，使用双向认证mTLS，要求客户端和服务器都有一套证书。与TLS相比，需要有额外的步骤来验证双方的身份。
[双向认证的握手过程](document/认证/images/mtls-handshake.png)

1. 客户端发送Hello消息到服务器
2. 服务器发送包含TLS证书的Hello消息给客户端
3. 服务器请求客户端发送客户端证书
4. 客户端验证服务器的证书
5. 客户端发送它的证书，并用服务器端的公钥加密
6. 服务器端验证客户端的证书
7. 客户端和服务器通过加密的TLS连接传输数据

### mTLS初体验：Hello-world
1. 第一步是创建一个客户端和服务器都信任的certificate authority (CA)。这个CA是一个X.509证书，由公开密钥加密的公钥和私钥组成。用到的命令是：
`openssl req   -new   -x509   -nodes   -days 365   -subj '/CN=my-ca'   -keyout ca.key   -out ca.crt`

创建后，可以查看ca.crt的详细情况，签发机构是`CN=my-ca`.
```
openssl x509 \
  --in ca.crt \
  -text \
  --nooutopenssl x509 \
  --in ca.crt \
  -text \
  --noout
```

2. 创建服务器端私钥和公开密钥
```openssl genrsa \
  -out server.key 2048

openssl req \
  -new \
  -key server.key \
  -subj '/CN=localhost' \
  -out server.csr

openssl x509 \
  -req \
  -in server.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -days 365 \
  -out server.crt
```
3. 创建客户端私钥和公开密钥
```
openssl genrsa \
  -out client.key 2048

openssl req \
  -new \
  -key client.key \
  -subj '/CN=my-client' \
  -out client.csr

openssl x509 \
  -req \
  -in client.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -days 365 \
  -out client.crt
```

4. 创建https服务器
这里有一个简单的nodejs服务器。把以下内容存成server.js. 然后运行`node server.js` 
```
const https = require('https');
const fs = require('fs');

const hostname = 'localhost';
const port = 3000;

const options = { 
    ca: fs.readFileSync('ca.crt'), 
    cert: fs.readFileSync('server.crt'), 
    key: fs.readFileSync('server.key'), 
    rejectUnauthorized: true,
    requestCert: true, 
}; 

const server = https.createServer(options, (req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello World');
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
```

服务器启动后，在另一个终端使用client证书访问服务器，会输出`Hello World`。
```
curl \
  --cacert ca.crt \
  --key client.key \
  --cert client.crt \
  https://localhost:3000
```
如果客户端不使用ca和客户端证书，那么会报错：
```
$ curl https://localhost:3000
curl: (60) SSL certificate problem: self signed certificate in certificate chain
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

如果客户端只使用ca.crt，或者client的证书，也会报错，所以ca.crt, client.key和client.crt，缺一不可。具体的错误信息这里就不罗列了，读者可以自己尝试一下。

参考文章：
https://en.wikipedia.org/wiki/Mutual_authentication
https://codeburst.io/mutual-tls-authentication-mtls-de-mystified-11fa2a52e9cf
https://www.openssl.org/docs/man1.0.2/man1/openssl-genrsa.html
https://cloud.ibm.com/docs/cis?topic=cis-mtls-features&mhsrc=ibmsearch_a&mhq=mtls



