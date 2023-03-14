# 快速链接Repository
在一台新的设备上连接仓库时，需要输入以下指令：
- 设置用户名和邮箱
```
git config --global user.name "Rickyuy"
git config --global user.email "Rickyuy@163.com"
```
- 创建ssh密钥
```
ssh-keygen -t rsa -C "Rickyuy@163.com"
```
- 在Github的用户设置中绑定该设备的ssh密钥

## 在拉取或上传时如果遇到如下问题：
```
fatal: unable to access 'http://github.com/我的库/': OpenSSL SSL_read: Connection was reset, errno 10054
```
解决办法:

1、修改设置，解除SSL验证。
```
git config --global http.sslVerify "false"
git config --global https.sslVerify "false"
```
