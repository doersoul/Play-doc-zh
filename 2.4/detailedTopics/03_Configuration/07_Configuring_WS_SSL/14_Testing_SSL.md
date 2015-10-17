#测试SSL

Testing an SSL client not only involves unit and integration testing, but also involves adversarial testing, which tests that an attacker cannot break or subvert a secure connection.


##Unit Testing
Play comes with `play.api.test.WsTestClient`, which provides two methods, `wsCall` and `wsUrl`. It can be helpful to use `PlaySpecification` and `in new WithApplication`

```scala
"calls index" in new WithApplication() {
  await(wsCall(routes.Application.index()).get())	
}
wsUrl("https://example.com").get()
```


##Integration Testing
If you want confirmation that your client is correctly configured, you can call out to [HowsMySSL](https://www.howsmyssl.com/s/api.html), which has an API to check JSSE settings.

```scala
import java.io.File

import play.api.{Mode, Environment}
import play.api.libs.json.JsSuccess
import play.api.libs.ws._
import play.api.libs.ws.ning._
import play.api.test._

import com.typesafe.config.{ConfigFactory, ConfigValueFactory}
import scala.concurrent.duration._

class HowsMySSLSpec extends PlaySpecification {

  def createClient(rawConfig: play.api.Configuration): WSClient = {
    val classLoader = Thread.currentThread().getContextClassLoader
    val parser = new WSConfigParser(rawConfig, new Environment(new File("."), classLoader, Mode.Test))
    val clientConfig = new NingWSClientConfig(parser.parse())
    // Debug flags only take effect in JSSE when DebugConfiguration().configure is called.
    //import play.api.libs.ws.ssl.debug.DebugConfiguration
    //clientConfig.ssl.map {
    //   _.debug.map(new DebugConfiguration().configure)
    //}
    val builder = new NingAsyncHttpClientConfigBuilder(clientConfig)
    val client = new NingWSClient(builder.build())
    client
  }

  def configToMap(configString: String): Map[String, _] = {
    import scala.collection.JavaConverters._
    ConfigFactory.parseString(configString).root().unwrapped().asScala.toMap
  }

  "WS" should {

    "verify common behavior" in {
      // GeoTrust SSL CA - G2 intermediate certificate not found in cert chain!
      // See https://github.com/jmhodges/howsmyssl/issues/38 for details.
      val geoTrustPem =
        """-----BEGIN CERTIFICATE-----
          |MIIEWTCCA0GgAwIBAgIDAjpjMA0GCSqGSIb3DQEBBQUAMEIxCzAJBgNVBAYT
          |AlVTMRYwFAYDVQQKEw1HZW9UcnVzdCBJbmMuMRswGQYDVQQDExJHZW9UcnVz
          |dCBHbG9iYWwgQ0EwHhcNMTIwODI3MjA0MDQwWhcNMjIwNTIwMjA0MDQwWjBE
          |MQswCQYDVQQGEwJVUzEWMBQGA1UEChMNR2VvVHJ1c3QgSW5jLjEdMBsGA1UE
          |AxMUR2VvVHJ1c3QgU1NMIENBIC0gRzIwggEiMA0GCSqGSIb3DQEBAQUAA4IB
          |DwAwggEKAoIBAQC5J/lP2Pa3FT+Pzc7WjRxr/X/aVCFOA9jK0HJSFbjJgltY
          |eYT/JHJv8ml/vJbZmnrDPqnPUCITDoYZ2+hJ74vm1kfy/XNFCK6PrF62+J58
          |9xD/kkNm7xzU7qFGiBGJSXl6Jc5LavDXHHYaKTzJ5P0ehdzgMWUFRxasCgdL
          |LnBeawanazpsrwUSxLIRJdY+lynwg2xXHNil78zs/dYS8T/bQLSuDxjTxa9A
          |kl0HXk7+Yhc3iemLdCai7bgK52wVWzWQct3YTSHUQCNcj+6AMRaraFX0DjtU
          |6QRN8MxOgV7pb1JpTr6mFm1C9VH/4AtWPJhPc48Obxoj8cnI2d+87FLXAgMB
          |AAGjggFUMIIBUDAfBgNVHSMEGDAWgBTAephojYn7qwVkDBF9qn1luMrMTjAd
          |BgNVHQ4EFgQUEUrQcznVW2kIXLo9v2SaqIscVbwwEgYDVR0TAQH/BAgwBgEB
          |/wIBADAOBgNVHQ8BAf8EBAMCAQYwOgYDVR0fBDMwMTAvoC2gK4YpaHR0cDov
          |L2NybC5nZW90cnVzdC5jb20vY3Jscy9ndGdsb2JhbC5jcmwwNAYIKwYBBQUH
          |AQEEKDAmMCQGCCsGAQUFBzABhhhodHRwOi8vb2NzcC5nZW90cnVzdC5jb20w
          |TAYDVR0gBEUwQzBBBgpghkgBhvhFAQc2MDMwMQYIKwYBBQUHAgEWJWh0dHA6
          |Ly93d3cuZ2VvdHJ1c3QuY29tL3Jlc291cmNlcy9jcHMwKgYDVR0RBCMwIaQf
          |MB0xGzAZBgNVBAMTElZlcmlTaWduTVBLSS0yLTI1NDANBgkqhkiG9w0BAQUF
          |AAOCAQEAPOU9WhuiNyrjRs82lhg8e/GExVeGd0CdNfAS8HgY+yKk3phLeIHm
          |TYbjkQ9C47ncoNb/qfixeZeZ0cNsQqWSlOBdDDMYJckrlVPg5akMfUf+f1Ex
          |RF73Kh41opQy98nuwLbGmqzemSFqI6A4ZO6jxIhzMjtQzr+t03UepvTp+UJr
          |YLLdRf1dVwjOLVDmEjIWE4rylKKbR6iGf9mY5ffldnRk2JG8hBYo2CVEMH6C
          |2Kyx5MDkFWzbtiQnAioBEoW6MYhYR3TjuNJkpsMyWS4pS0XxW4lJLoKaxhgV
          |RNAuZAEVaDj59vlmAwxVG52/AECu8EgnTOCAXi25KhV6vGb4NQ==
          |-----END CERTIFICATE-----
        """.stripMargin

      val configString = """
         |//play.ws.ssl.debug=["certpath", "ssl", "trustmanager"]
         |play.ws.ssl.protocol="TLSv1"
         |play.ws.ssl.enabledProtocols=["TLSv1"]
         |
         |play.ws.ssl.trustManager = {
         |  stores = [
         |    { type: "PEM", data = ${geotrust.pem} }
         |  ]
         |}
       """.stripMargin
      val rawConfig = ConfigFactory.parseString(configString)
      val configWithPem = rawConfig.withValue("geotrust.pem", ConfigValueFactory.fromAnyRef(geoTrustPem))
      val configWithSystemProperties = ConfigFactory.load(configWithPem)
      val playConfiguration = play.api.Configuration(configWithSystemProperties)

      val client = createClient(playConfiguration)
      val response = await(client.url("https://www.howsmyssl.com/a/check").get())(5.seconds)
      response.status must be_==(200)

      val jsonOutput = response.json
      val result = (jsonOutput \ "tls_version").validate[String]
      result must beLike {
        case JsSuccess(value, path) =>
          value must_== "TLS 1.0"
      }
    }
  }
}
```

Note that if you are writing tests that involve custom configuration such as revocation checking or disabled algorithms, you may need to pass system properties into SBT:

```scala
javaOptions in Test ++= Seq("-Dcom.sun.security.enableCRLDP=true", "-Dcom.sun.net.ssl.checkRevocation=true", "-Djavax.net.debug=all")
```


##Adversarial Testing
There are several points of where a connection can be attacked. Writing these tests is fairly easy, and running these adversarial tests against unsuspecting programmers can be extremely satisfying.

> **NOTE**:This should not be taken as a complete list, but as a guide. In situations where security is paramount, a review should be done by professional info-sec consultants.

###Testing Certificate Verification
Write a test to connect to "[https://example.com](https://example.com/)". The server should present a certificate which says the subjectAltName is dnsName, but the certificate should be signed by a CA certificate which is not in the trust store. The client should reject it.

This is a very common failure. There are a number of proxies like [mitmproxy](https://mitmproxy.org/) or [Fiddler](http://www.telerik.com/fiddler) which will only work if certificate verification is disabled or the proxy’s certificate is explicitly added to the trust store.

###Testing Weak Cipher Suites
The server should send a cipher suite that includes NULL or ANON cipher suites in the handshake. If the client accepts it, it is sending unencrypted data.

> **NOTE**: For a more in depth test of a server’s cipher suites, see [sslyze](https://github.com/iSECPartners/sslyze).

###Testing Certificate Validation
To test for weak signatures, the server should send the client a certificate which has been signed with, for example, the MD2 digest algorithm. The client should reject it as being too weak.

To test for weak certificate, The server should send the client a certificate which contains a public key with a key size under 1024 bits. The client should reject it as being too weak.

> **NOTE**: For a more in depth test of certification validation, see [tlspretense](https://github.com/iSECPartners/tlspretense) and [frankencert](https://github.com/sumanj/frankencert).

###Testing Hostname Verification
Write a test to "[https://example.com](https://example.com/)". If the server presents a certificate where the subjectAltName’s dnsName is not example.com, the connection should terminate.

> **NOTE**: For a more in depth test, see [dnschef](https://tersesystems.com/2014/03/31/testing-hostname-verification/).