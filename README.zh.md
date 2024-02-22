
# 查看 APK包 的 MD5 签名的命令行工具 #

[English README](README.md)

在申请微信支付、支付宝支付的过程中，都要求提交 APK 的签名。

如何获得签名呢？他们都提供了一个 Gen_Signagure_Android.apk，说是安装到安卓手机上，就能够读取第三方应用 APK 的签名。但是！这个 apk 有的手机不允许安装，比如小米手机。

那么有没有什么其他的方法获得签名呢？

有。有人建议用 jadx，这是个将 APK 包反编译得到 java 代码的强大工具。只不过，为了简单的获取指纹，而安装这个强大的工具，有点杀鸡用牛刀了。我尝试寻找更加简单易用的方法。

经过研究，我发现他们所验证的这个32个字节的签名，其实就是 APK 签名证书的 MD5 指纹。

使用命令
```bash
keytool -list -v -keystore  xxx.keystore
keytool -printcert -jarfile xxx.apk
```
都可以打印出签名信息，例如：
```
Signer #1:

Signature:

Owner: CN=Android, OU=Android, O=Google Inc., L=Mountain View, ST=California, C=US
Issuer: CN=Android, OU=Android, O=Google Inc., L=Mountain View, ST=California, C=US
Serial number: c2e08746644a308d
Valid from: Fri Aug 22 07:13:34 CST 2008 until: Tue Jan 08 07:13:34 CST 2036
Certificate fingerprints:
	 SHA1: 38:91:8A:45:3D:07:19:93:54:F8:B1:9A:F0:5E:C6:56:2C:ED:57:88
	 SHA256: F0:FD:6C:5B:41:0F:25:CB:25:C3:B5:33:46:C8:97:2F:AE:30:F8:EE:74:11:DF:91:04:80:AD:6B:2D:60:DB:83
Signature algorithm name: MD5withRSA (disabled)
Subject Public Key Algorithm: 2048-bit RSA key
Version: 3

Extensions:

#1: ObjectId: 2.5.29.35 Criticality=false
AuthorityKeyIdentifier [
KeyIdentifier [
0000: C7 7D 8C C2 21 17 56 25   9A 7F D3 82 DF 6B E3 98  ....!.V%.....k..
0010: E4 D7 86 A5                                        ....
]
[CN=Android, OU=Android, O=Google Inc., L=Mountain View, ST=California, C=US]
SerialNumber: [    c2e08746 644a308d]
]

#2: ObjectId: 2.5.29.19 Criticality=false
BasicConstraints:[
  CA:true
  PathLen:2147483647
]

#3: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: C7 7D 8C C2 21 17 56 25   9A 7F D3 82 DF 6B E3 98  ....!.V%.....k..
0010: E4 D7 86 A5                                        ....
]
]
```
但是，我们注意到这里并没有打印出 MD5 信息。经过调查，发现是 java8 以后的 keytool 版本已经去掉了 MD5 支持，因为发现 MD5 并不严谨，存在安全问题。

既然能够计算出 SHA1 和 SHA256 指纹，是不是也能够有办法计算出 MD5 指纹呢？让我们试试看。

我们都知道 APK 和 IPA 文件格式，其实都个 zip 文件。我们把 APK 解压，将会得到 META-INF/CERT.RSA。这个文件是包含了签名证书 以及 签名信息。
使用命令
```bash
keytool -printcert -file CERT.RSA
```
我们也可以打印出同样的签名信息来，只不过同样只显示 SHA1 和 SHA256，没有显示 MD5。

那么，能不能通过这个 CERT.RSA 文件获得 MD5 指纹呢？答案是肯定的。

使用 openssl 这个超级牛X 的密码工具箱，我们可以从 CERT.RSA 文件中提取出 签名证书（也就是PKI证书的公钥），并提取出指纹。

经过推敲，用如下的一系列命令，就能够打印出签名证书的指纹：
方法一：直接读取 keystore 的证书：
```bash
keytool -exportcert -keystore xxx.keystore | openssl dgst -md5
```
方法二，从 APK 包中提取证书：
```bash
# unzip the IPA file to tmp folder
mkdir ./tmp
unzip <path_to_apk_file> -d ./tmp

# run openssl to extract certificate file from CERT.RSA
openssl pkcs7 -inform DER -in ./tmp/META-INF/CERT.RSA -print_certs -out ./tmp/CERT.cert

# print the fingerprints in upper case
openssl x509 -in ./tmp/CERT.cert -fingerprint -noout -md5
openssl x509 -in ./tmp/CERT.cert -fingerprint -noout -sha1
openssl x509 -in ./tmp/CERT.cert -fingerprint -noout -sha256

# print the fingterprints in lower case
openssl x509 -in ./tmp/CERT.cert -outform DER | openssl dgst -md5
openssl x509 -in ./tmp/CERT.cert -outform DER | openssl dgst -sha1
openssl x509 -in ./tmp/CERT.cert -outform DER | openssl dgst -sha256

# clean up the tmp folder
rm -r ./tmp
```
最后，为了便于使用，我把这一系列命令，封装为一个 python 命令行程序，并发布到 pypi 上，通过 pip 安装就可以直接使用了。
```bash
python3 -m pip install apk-signature
apk-signature myapp.apk
```
效果如下：
```
Extracting APK: /Users/liming/workspace/apk/google-play-store.apk ...
--- Signature in upper case ---
md5 Fingerprint=CD:E9:F6:20:8D:67:2B:54:B1:DA:CC:0B:70:29:F5:EB
sha1 Fingerprint=38:91:8A:45:3D:07:19:93:54:F8:B1:9A:F0:5E:C6:56:2C:ED:57:88
sha256 Fingerprint=F0:FD:6C:5B:41:0F:25:CB:25:C3:B5:33:46:C8:97:2F:AE:30:F8:EE:74:11:DF:91:04:80:AD:6B:2D:60:DB:83
--- Signature in lower case ---
MD5(stdin)= cde9f6208d672b54b1dacc0b7029f5eb
SHA1(stdin)= 38918a453d07199354f8b19af05ec6562ced5788
SHA2-256(stdin)= f0fd6c5b410f25cb25c3b53346c8972fae30f8ee7411df910480ad6b2d60db83
------------ Done -------------
```

全文完。
