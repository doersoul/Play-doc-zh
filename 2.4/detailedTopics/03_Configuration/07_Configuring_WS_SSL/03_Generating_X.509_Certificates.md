#生成 X.509 证书


##X.509 Certificates
Public key certificates are a solution to the problem of identity. Encryption alone is enough to set up a secure connection, but there’s no guarantee that you are talking to the server that you think you are talking to. Without some means to verify the identity of a remote server, an attacker could still present itself as the remote server and then forward the secure connection onto the remote server. Public key certificates solve this problem.

The best way to think about public key certificates is as a passport system. Certificates are used to establish information about the bearer of that information in a way that is difficult to forge. This is why certificate verification is so important: accepting **any** certificate means that even an attacker’s certificate will be blindly accepted.


##Using Keytool
Use the keytool version that comes with JDK 8:

* [1.8](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/keytool.html)

The examples below use keytool 1.8 for marking a certificate for CA usage or for a hostname.


##Generating a random password
Create a random password using pwgen (`brew install pwgen` if you’re on a Mac):

```shell
export PW=`pwgen -Bs 10 1`
echo $PW > password
```


##Server Configuration
You will need a server with a DNS hostname assigned, for hostname verification. In this example, we assume the hostname is `example.com`.

###Generating a server CA
The first step is to create a certificate authority that will sign the example.com certificate. The root CA certificate has a couple of additional attributes (ca:true, keyCertSign) that mark it explicitly as a CA certificate, and will be kept in a trust store.

```shell
export PW=`cat password`

# Create a self signed key pair root CA certificate.
keytool -genkeypair -v \
  -alias exampleca \
  -dname "CN=exampleCA, OU=Example Org, O=Example Company, L=San Francisco, ST=California, C=US" \
  -keystore exampleca.jks \
  -keypass:env PW \
  -storepass:env PW \
  -keyalg RSA \
  -keysize 4096 \
  -ext KeyUsage:critical="keyCertSign" \
  -ext BasicConstraints:critical="ca:true" \
  -validity 9999

# Export the exampleCA public certificate as exampleca.crt so that it can be used in trust stores.
keytool -export -v \
  -alias exampleca \
  -file exampleca.crt \
  -keypass:env PW \
  -storepass:env PW \
  -keystore exampleca.jks \
  -rfc
```

###Generating example.com certificates
The example.com certificate is presented by the `example.com` server in the handshake.

```shell
export PW=`cat password`

# Create a server certificate, tied to example.com
keytool -genkeypair -v \
  -alias example.com \
  -dname "CN=example.com, OU=Example Org, O=Example Company, L=San Francisco, ST=California, C=US" \
  -keystore example.com.jks \
  -keypass:env PW \
  -storepass:env PW \
  -keyalg RSA \
  -keysize 2048 \
  -validity 385

# Create a certificate signing request for example.com
keytool -certreq -v \
  -alias example.com \
  -keypass:env PW \
  -storepass:env PW \
  -keystore example.com.jks \
  -file example.com.csr

# Tell exampleCA to sign the example.com certificate. Note the extension is on the request, not the
# original certificate.
# Technically, keyUsage should be digitalSignature for DHE or ECDHE, keyEncipherment for RSA.
keytool -gencert -v \
  -alias exampleca \
  -keypass:env PW \
  -storepass:env PW \
  -keystore exampleca.jks \
  -infile example.com.csr \
  -outfile example.com.crt \
  -ext KeyUsage:critical="digitalSignature,keyEncipherment" \
  -ext EKU="serverAuth" \
  -ext SAN="DNS:example.com" \
  -rfc

# Tell example.com.jks it can trust exampleca as a signer.
keytool -import -v \
  -alias exampleca \
  -file exampleca.crt \
  -keystore example.com.jks \
  -storetype JKS \
  -storepass:env PW << EOF
yes
EOF

# Import the signed certificate back into example.com.jks 
keytool -import -v \
  -alias example.com \
  -file example.com.crt \
  -keystore example.com.jks \
  -storetype JKS \
  -storepass:env PW

# List out the contents of example.com.jks just to confirm it.  
# If you are using Play as a TLS termination point, this is the key store you should present as the server.
keytool -list -v \
  -keystore example.com.jks \
  -storepass:env PW
```

You should see:

```shell
Alias name: example.com
Creation date: ...
Entry type: PrivateKeyEntry
Certificate chain length: 2
Certificate[1]:
Owner: CN=example.com, OU=Example Org, O=Example Company, L=San Francisco, ST=California, C=US
Issuer: CN=exampleCA, OU=Example Org, O=Example Company, L=San Francisco, ST=California, C=US
```

> **NOTE**: Also see the [Configuring HTTPS](https://playframework.com/documentation/2.4.x/ConfiguringHttps) section for more information.

###Configuring example.com certificates in Nginx
If example.com does not use Java as a TLS termination point, and you are using nginx, you may need to export the certificates in PEM format.

Unfortunately, keytool does not export private key information, so openssl must be installed to pull private keys.

```shell
export PW=`cat password`

# Export example.com's public certificate for use with nginx.
keytool -export -v \
  -alias example.com \
  -file example.com.crt \
  -keypass:env PW \
  -storepass:env PW \
  -keystore example.com.jks \
  -rfc

# Create a PKCS#12 keystore containing the public and private keys.
keytool -importkeystore -v \
  -srcalias example.com \
  -srckeystore example.com.jks \
  -srcstoretype jks \
  -srcstorepass:env PW \
  -destkeystore example.com.p12 \
  -destkeypass:env PW \
  -deststorepass:env PW \
  -deststoretype PKCS12

# Export the example.com private key for use in nginx.  Note this requires the use of OpenSSL.
openssl pkcs12 \
  -nocerts \
  -nodes \
  -passout env:PW \
  -passin env:PW \
  -in example.com.p12 \
  -out example.com.key

# Clean up.
rm example.com.p12
```

Now that you have both `example.com.crt` (the public key certificate) and `example.com.key` (the private key), you can set up an HTTPS server.

For example, to use the keys in nginx, you would set the following in `nginx.conf`:

```scala
ssl_certificate      /etc/nginx/certs/example.com.crt;
ssl_certificate_key  /etc/nginx/certs/example.com.key;
```

If you are using client authentication (covered in **Client Configuration** below), you will also need to add:

```scala
ssl_client_certificate /etc/nginx/certs/clientca.crt;
ssl_verify_client on;
```

You can check the certificate is what you expect by checking the server:

```scala
keytool -printcert -sslserver example.com
```

> **NOTE**: Also see the [Setting up a front end HTTP server](https://playframework.com/documentation/2.4.x/HTTPServer) section for more information.


##Client Configuration
There are two parts to setting up a client – configuring a trust store, and configuring client authentication.

###Configuring a Trust Store
Any clients need to see that the server’s example.com certificate is trusted, but don’t need to see the private key. Generate a trust store which contains only the certificate and hand that out to clients. Many java clients prefer to have the trust store in JKS format.

```shell
export PW=`cat password`

# Create a JKS keystore that trusts the example CA, with the default password.
keytool -import -v \
  -alias exampleca \
  -file exampleca.crt \
  -keypass:env PW \
  -storepass changeit \
  -keystore exampletrust.jks << EOF
yes
EOF

# List out the details of the store password.
keytool -list -v \
  -keystore exampletrust.jks \
  -storepass changeit
```

You should see a `trustedCertEntry` for exampleca:

```shell
Alias name: exampleca
Creation date: ...
Entry type: trustedCertEntry

Owner: CN=exampleCA, OU=Example Org, O=Example Company, L=San Francisco, ST=California, C=US
Issuer: CN=exampleCA, OU=Example Org, O=Example Company, L=San Francisco, ST=California, C=US
```

The `exampletrust.jks` store will be used in the TrustManager.

```scala
play.ws.ssl {
  trustManager = {
    stores = [
      { path = "/Users/wsargent/work/ssltest/conf/exampletrust.jks" }
    ]
  }
}
```

> **NOTE**: Also see the [Configuring Key Stores](https://playframework.com/documentation/2.4.x/KeyStores) and Trust Stores section for more information.

###Configure Client Authentication
Client authentication can be obscure and poorly documented, but it relies on the following steps:

1. The server asks for a client certificate, presenting a CA that it expects a client certificate to be signed with. In this case, `CN=clientCA` (see the [debug example](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jsse/ReadDebug.html)).
2. The client looks in the KeyManager for a certificate which is signed by `clientCA`, using `chooseClientAlias` and `certRequest.getAuthorities`.
3. The KeyManager will return the `client` certificate to the server.
4. The server will do an additional ClientKeyExchange in the handshake.

The steps to create a client CA and a signed client certificate are broadly similiar to the server certificate generation, but for convenience are presented in a single script:

```shell
export PW=`cat password`

# Create a self signed certificate & private key to create a root certificate authority.
keytool -genkeypair -v \
  -alias clientca \
  -keystore client.jks \
  -dname "CN=clientca, OU=Example Org, O=Example Company, L=San Francisco, ST=California, C=US" \
  -keypass:env PW \
  -storepass:env PW \
  -keyalg RSA \
  -keysize 4096 \
  -ext KeyUsage:critical="keyCertSign" \
  -ext BasicConstraints:critical="ca:true" \
  -validity 9999

# Create another key pair that will act as the client.
keytool -genkeypair -v \
  -alias client \
  -keystore client.jks \
  -dname "CN=client, OU=Example Org, O=Example Company, L=San Francisco, ST=California, C=US" \
  -keypass:env PW \
  -storepass:env PW \
  -keyalg RSA \
  -keysize 2048

# Create a certificate signing request from the client certificate.
keytool -certreq -v \
  -alias client \
  -keypass:env PW \
  -storepass:env PW \
  -keystore client.jks \
  -file client.csr

# Make clientCA create a certificate chain saying that client is signed by clientCA.
keytool -gencert -v \
  -alias clientca \
  -keypass:env PW \
  -storepass:env PW \
  -keystore client.jks \
  -infile client.csr \
  -outfile client.crt \
  -ext EKU="clientAuth" \
  -rfc

# Export the client-ca certificate from the keystore.  This goes to nginx under "ssl_client_certificate"
# and is presented in the CertificateRequest.
keytool -export -v \
  -alias clientca \
  -file clientca.crt \
  -storepass:env PW \
  -keystore client.jks \
  -rfc

# Import the signed certificate back into client.jks.  This is important, as JSSE won't send a client
# certificate if it can't find one signed by the client-ca presented in the CertificateRequest.
keytool -import -v \
  -alias client \
  -file client.crt \
  -keystore client.jks \
  -storetype JKS \
  -storepass:env PW

# Export the client CA's certificate and private key to pkcs12, so it's safe.
keytool -importkeystore -v \
  -srcalias clientca \
  -srckeystore client.jks \
  -srcstorepass:env PW \
  -destkeystore clientca.p12 \
  -deststorepass:env PW \
  -deststoretype PKCS12

# Import the client CA's public certificate into a JKS store for Play Server to read.  We don't use
# the PKCS12 because it's got the CA private key and we don't want that.
keytool -import -v \
  -alias clientca \
  -file clientca.crt \
  -keystore clientca.jks \
  -storepass:env PW << EOF
yes
EOF

# Then, strip out the client CA alias from client.jks, just leaving the signed certificate.
keytool -delete -v \
 -alias clientca \
 -storepass:env PW \
 -keystore client.jks

# List out the contents of client.jks just to confirm it.
keytool -list -v \
  -keystore client.jks \
  -storepass:env PW
```

There should be one alias `client`, looking like the following:

```shell
Your keystore contains 1 entry

Alias name: client
Creation date: ...
Entry type: PrivateKeyEntry
Certificate chain length: 2
Certificate[1]:
Owner: CN=client, OU=Example Org, O=Example Company, L=San Francisco, ST=California, C=US
Issuer: CN=clientCA, OU=Example Org, O=Example Company, L=San Francisco, ST=California, C=US
```

And put `client.jks` in the key manager:

```scala
play.ws.ssl {
  keyManager = {
    stores = [
      { type = "JKS", path = "conf/client.jks", password = $PW }
    ]
  }
}
```

> **NOTE**: Also see the [Configuring Key Stores](https://playframework.com/documentation/2.4.x/KeyStores) and Trust Stores section for more information.


##Certificate Management Tools
If you want to examine certificates in a graphical tool than a command line tool, you can use [Keystore Explorer](http://keystore-explorer.sourceforge.net/) or [xca](http://sourceforge.net/projects/xca/). [Keystore Explorer](http://keystore-explorer.sourceforge.net/) is especially convenient as it recognizes JKS format. It works better as a manual installation, and requires some tweaking to the export policy.

If you want to use a command line tool with more flexibility than keytool, try [java-keyutil](https://code.google.com/p/java-keyutil/), which understands multi-part PEM formatted certificates and JKS.


##Certificate Settings
###Secure
If you want the best security, consider using [ECDSA](https://blog.cloudflare.com/ecdsa-the-digital-signature-algorithm-of-a-better-internet) as the signature algorithm (in keytool, this would be `-sigalg EC`). ECDSA is also known as “ECC SSL Certificate”.

###Compatible
For compatibility with older systems, use RSA with 2048 bit keys and SHA256 as the signature algorithm. If you are creating your own CA certificate, use 4096 bits for the root.


##Further Reading

* [JSSE Reference Guide To Creating KeyStores](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jsse/JSSERefGuide.html#CreateKeystore)
* [Java PKI Programmer’s Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/security/certpath/CertPathProgGuide.html)
* [Fixing X.509 Certificates](https://tersesystems.com/2014/03/20/fixing-x509-certificates/)