#配置 WS SSL

[Play WS](https://playframework.com/documentation/2.4.x/ScalaWS) allows you to set up HTTPS completely from a configuration file, without the need to write code. It does this by layering the Java Secure Socket Extension (JSSE) with a configuration layer and with reasonable defaults.

JDK 1.8 contains an implementation of JSSE which is [significantly more advanced](https://docs.oracle.com/javase/8/docs/technotes/guides/security/enhancements-8.html) than previous versions, and should be used if security is a priority.


##Table of Contents

* [Quick Start to WS SSL](02_Quick_Start_to_WS_SSL.md)
* [Generating X.509 Certificates](03_Generating_X.509_Certificates.md)
* [Configuring Trust Stores and Key Stores](04_Configuring_Trust_Stores_and_Key_Stores.md)
* [Configuring Protocols](05_Configuring_Protocols.md)
* [Configuring Cipher Suites](06_Configuring_Cipher_Suites.md)
* [Configuring Certificate Validation](07_Configuring_Certificate_Validation.md)
* [Configuring Certificate Revocation](08_Configuring_Certificate_Revocation.md)
* [Configuring Hostname Verification](09_Configuring_Hostname_Verification.md)
* [Example Configurations](10_Example_Configurations.md)
* [Using the Default SSLContext](11_Using_the_Default_SSLContext.md)
* [Debugging SSL Connections](12_Debugging_SSL.md)
* [Loose Options](13_Loose_Options.md)
* [Testing SSL](14_Testing_SSL.md)


##Further Reading
JSSE is a complex product. For convenience, the JSSE materials are provided here:

JDK 1.8:

* [JSSE Reference Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jsse/JSSERefGuide.html)
* [JSSE Crypto Spec](https://docs.oracle.com/javase/8/docs/technotes/guides/security/crypto/CryptoSpec.html#SSLTLS)
* [SunJSSE Providers](https://docs.oracle.com/javase/8/docs/technotes/guides/security/SunProviders.html#SunJSSEProvider)
* [PKI Programmer’s Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/security/certpath/CertPathProgGuide.html)
* [keytool](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/keytool.html)