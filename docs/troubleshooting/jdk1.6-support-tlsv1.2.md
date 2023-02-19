---
title: 让 Oracle JDK 1.6 TLSv1.2
description: 这篇帖子讲述了通过安装额外的组件，使得 Oracle JDK 1.6 支持 TLSv1.2 协议
tags: ["jdk", "tlsv1.2", "bcprov", "bctls"]
---

# 让 Oracle JDK 1.6 TLSv1.2

Oracle JDK 是从 1.7 才开始支持 `TLSv1.2` 及以上版本，如果又不想降低配置来支持包含风险的 `TLSv1.1` ，解决版本是利用 ` Bouncy Castle` 包来替换 JDK 内置的默认 `TLS` 处理程序。
解决办法如下：

1. 从 [Java Archive Oracle website](https://www.oracle.com/java/technologies/javase-java-archive-javase6-downloads.html) 下载 JDK 1.6
2. 下载 [Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files 6](../assets/jce_policy-6.zip)，并去目录方式解压到 JDK 1.6 的 `jre/lib/security` 目录下
3. 下载 [bcprov-jdk15to18-165.jar](../assets/bcprov-jdk15to18-1.64.jar) 和 [bctls-jdk15to18-165.jar](../assets/bctls-jdk15to18-1.64.jar) 两个 jar 文件并拷贝到 `${JAVA_HOME}/jre/lib/ext` 目录
4. 修改 `${JAVA_HOME}/jre/lib/security/java.security` 文件，注释`security.provider`的若干行，并增加若干行，如下：

    ```ini
    #
     Original security providers (just comment it)
    # security.provider.1=sun.security.provider.Sun
    # security.provider.2=sun.security.rsa.SunRsaSign
    # security.provider.3=com.sun.net.ssl.internal.ssl.Provider
    # security.provider.4=com.sun.crypto.provider.SunJCE
    # security.provider.5=sun.security.jgss.SunProvider
    # security.provider.6=com.sun.security.sasl.Provider
    # security.provider.7=org.jcp.xml.dsig.internal.dom.XMLDSigRI
    # security.provider.8=sun.security.smartcardio.SunPCSC

    # Add the Bouncy Castle security providers with higher priority
    security.provider.1=org.bouncycastle.jce.provider.BouncyCastleProvider
    security.provider.2=org.bouncycastle.jsse.provider.BouncyCastleJsseProvider

    # Original security providers with different priorities
    security.provider.3=sun.security.provider.Sun
    security.provider.4=sun.security.rsa.SunRsaSign
    security.provider.5=com.sun.net.ssl.internal.ssl.Provider
    security.provider.6=com.sun.crypto.provider.SunJCE 
    security.provider.7=sun.security.jgss.SunProvider
    security.provider.8=com.sun.security.sasl.Provider
    security.provider.9=org.jcp.xml.dsig.internal.dom.XMLDSigRI
    security.provider.10=sun.security.smartcardio.SunPCSC

    # Here we are changing the default SSLSocketFactory implementation
    ssl.SocketFactory.provider=org.bouncycastle.jsse.provider.SSLSocketFactoryImpl
    ```

5. 测试
