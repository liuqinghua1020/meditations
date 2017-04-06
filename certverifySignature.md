#一个证书验证的错误分析

在工作中遇到一个问题，错误如下：

    java.security.SignatureException: Signature encoding error 
        at sun.security.rsa.RSASignature.engineVerify(RSASignature.java:208) 
        at java.security.Signature$Delegate.engineVerify(Signature.java:1217) 
        at java.security.Signature.verify(Signature.java:651) 
        at sun.security.x509.X509CertImpl.verify(X509CertImpl.java:446) 
        at sun.security.x509.X509CertImpl.verify(X509CertImpl.java:394) 
        at net.cnca.gdltax.pkicenter.appintf.CertVerifier.VerifyCertWithP7B(CertVerifier.java:160) 
        at net.cnca.gdltax.pkicenter.appintf.CertVerifier.VerifyUserSignCert(CertVerifier.java:107) 
        at net.cnca.gdltax.pkicenter.appintf.CertVerifier.VerifyUserSignCert(CertVerifier.java:42) 
        at net.cnca.gdltax.pkicenter.appintf.PKIIntfImpl.checkYzqmfwjkPKCS1(PKIIntfImpl.java:278) 
        at sun.reflect.GeneratedMethodAccessor48.invoke(Unknown Source) 
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) 
        at java.lang.reflect.Method.invoke(Method.java:606) 
        at com.taobao.hsf.remoting.provider.ProviderProcessor.handleRequest0(ProviderProcessor.java:466) 
        at com.taobao.hsf.remoting.provider.ProviderProcessor.handleRequest(ProviderProcessor.java:205) 
        at com.taobao.hsf.remoting.server.RPCServerHandler.handleRequest(RPCServerHandler.java:42) 
        at com.taobao.hsf.remoting.server.RPCServerHandler.handleRequest(RPCServerHandler.java:20) 
        at com.taobao.hsf.remoting.netty.server.NettyServerHandler$HandlerRunnable.run(NettyServerHandler.java:122) 
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145) 
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615) 
        at java.lang.Thread.run(Thread.java:745) 
    Caused by: java.io.IOException: Sequence tag error 
        at sun.security.util.DerInputStream.getSequence(DerInputStream.java:297) 
        at sun.security.rsa.RSASignature.decodeSignature(RSASignature.java:233) 
        at sun.security.rsa.RSASignature.engineVerify(RSASignature.java:197) 
        ... 19 more 

简化调试代码在如下地址：
https://github.com/liuqinghua1020/PKIUtil/blob/master/src/main/java/com/shark/PKIUtil/cert/verifyCertPathSignature.java

其中出问题的地方是

    certobj.verify(cacertobj.getPublicKey());

其中 certobj 和 cacertobj 都是X509Certificate的对象。
从java抛出的异常信息来看，这个是一个 签名证书本的签名值是存在问题，从下来的错误描述可以看出：
    
    java.security.SignatureException: Signature encoding error 
        at sun.security.rsa.RSASignature.engineVerify(RSASignature.java:208) 
        at java.security.Signature$Delegate.engineVerify(Signature.java:1217) 
        at java.security.Signature.verify(Signature.java:651) 
        at sun.security.x509.X509CertImpl.verify(X509CertImpl.java:446) 
        at sun.security.x509.X509CertImpl.verify(X509CertImpl.java:394) 
    Caused by: java.io.IOException: Sequence tag error 
        at sun.security.util.DerInputStream.getSequence(DerInputStream.java:297) 
        at sun.security.rsa.RSASignature.decodeSignature(RSASignature.java:233) 
        at sun.security.rsa.RSASignature.engineVerify(RSASignature.java:197) 

这已经是一个比较深入到密码学的内容了，如何确认问题所在捏。首先，通过部门专家的分析， 确定了大致的方向，签名值通过对CA公钥进行指模运算后，应该是一个标准的P1结构，
那么一个标准的P1结构是什么样的捏：

P1标准：
01 ff(n个) 00 后面是DigestInfo结构，是个Sequence，里面包含摘要算法和摘要值。
相关信息可以参考：

https://www.emc.com/collateral/white-papers/h11300-pkcs-1v2-2-rsa-cryptography-standard-wp.pdf

的9.2 章节。

那么如何得到这个结构捏，首先我们看下 签名和验证签名的过程

l  签名过程：

    1. A计算消息m的消息摘要,记为 h(m)
    2. A使用私钥(n,d)对h(m)加密，生成签名s ,s满足：
            s=(h(m))^d mod n;
由于A是用自己的私钥对消息摘要加密，所以只用使用s的公钥才能解密该消息摘要，这样A就不可否认自己发送了该消息给B。

    3. A发送消息和签名(m,s)给B。

l  验签过程：

    1. B计算消息m的消息摘要,记为h(m);
    2. B使用A的公钥(n,e)解密s,得到
            H(m) = s^e mod n;
    3. B比较H(m)与h(m),相同则证明

目前怀疑的是H(m)的值不正确，所以需要先算出H(m)，现在有证书，所以知道的是签名值，根据验证签名的过程公式
    
    H(m) = s^e mod n

即可。
相关代码如下

    byte[] signature= certobj.getSignature();
    BigInteger bsi= new BigInteger(signature);
    RSAPublicKey capubkey= (RSAPublicKey) cacertobj.getPublicKey();
    BigInteger tmp= bsi.modPow(capubkey.getPublicExponent(), capubkey.getModulus());
    String strtmp= tmp.toString(16);
    System.out.println("P1签名的原文摘要值+填充数据为 \r\n " + strtmp);

这里可以对比没有问题的和有问题的证书的情况

没问题的证书的签名值，对CA的公钥进行指模运算后得到：
01ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff003021300906052b0e03021a05000414d63ea1cf4945fd28053ce988eef6450f050c0bc0
是个P1标准的签名；


有问题的证书的签名值，对CA的公钥进行指模运算后得到：
01ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff0076933034a0696f83c938192c3af73ef9a746ef08
是非P1标准的签名。







