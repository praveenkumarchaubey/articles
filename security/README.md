# Connecting to Secure Kafka in Go

# Overview

In this 3-part series, we will get to know about network security and a write a Go client that connects to a secure Kafka cluster. The orientation is around quickly building concepts and practical learning.

Security in an organisation is of outmost concern. It is essential to mitigate any risks of unauthorised access, data corruption and loss of confidentiality.

In Kafka, authentication and authorisation are disabled by default. Anyone is allowed to read, write or manage the topics. To mitigate this, Kafka offers various methods for securing clients. This document describes the steps to connect to a secure Kafka using a Go client. To do so, we first need to understand how Kafka's security mechanisms works in the background.

# Part 1 - Security Introduction

Apache Kafka comes with inbuilt configurable security protocols, the implementation of these are based on case to case basis. Each security feature has its own performance implications and this needs to be analysed thoroughly before implementation.

## Security Services

The following security services are available in Kafka:

- **Authentication:** Authentication in Kafka ensures that only clients that can prove their identity can connect to the secure Kafka. This is achieved using SSL or SASL.
- **Confidentiality:** Confidentiality is provided by encryption. This ensures that the data-in-transit between application and Kafka brokers is secure and no other client could intercept or read data. This is achieved using SSL (Secure socket layer)/TLS (Transport layer security).
- **Authorisation:** Once the client is authenticated, authorisation defines actions that it is allowed to perform. The Kafka brokers can run the client against access control lists (ACL) to determine whether a particular client is authorised to read or write to a particular topic.

## X.509 Certificates

A **certificate** contains a public key and other information and is created by a certificate authority. X.509 is a standard defining the pieces of information in a certificate. This includes:

- Version - X.509 standard version number.
- Serial Number - A sequence number given to each certificate.
- Signature Algorithm Identifier - Name of the algorithm used to sign this certificate by the issuer.
- Issuer Name - Name of the issuer.
- Validity Period - Period during which this certificate is valid.
- Subject Name - Name of the owner of the public key.
- Subject Public Key Information - The public key and its related information.

The public key is part of a key pair that also includes a private key. The private key is kept secure, and the public key is included in the certificate. This public/private key pair allows the owner of the private key to digitally sign documents. These signatures can be verified by anyone with the corresponding public key. It also allows third parties to send messages encrypted with the public key that only the owner of the private key can decrypt. RSA, ECC & DSA are the most common algorithms used to generate public keys.

A digital signature is an encoded hash of a document that has been encrypted with a private key. When an X.509 certificate is signed by a publicly trusted CA the certificate can be used by a third party to verify the identity of the entity presenting it.

## Ways of Encoding X.509 Certificates

There are four different ways to present X.509 certificates. These are nothing but different ways to encode ASN.1 formatted data.

- **PEM -** Privacy Enhanced Mail, governed by RFC 1422, is a base64 translation of the x509 ASN.1 keys generally used by open-source software because it is text-based and hence less prone to translation/transmission errors. It can have a variety of extensions like `.pem`, `.key`, `.cer`, `.crt` etc.
- **PKCS7** - Public-Key Cryptography Standards #7, is an open standard format used by Windows for certificate interchange. It does not contain private key material. Java understands these natively, and often uses `.keystore` as an extension instead.
- **PKCS12** - Public-Key Cryptography Standards #12, is a Microsoft private standard that was later defined in an RFC. This is a password-protected fully encrypted container format that contains both public and private certificate pairs. It can be freely converted to PEM format through use of openssl.
- **DER** - A way to encode ASN.1 syntax in binary. It is the parent format of PEM. A `.pem` file is just a Base64 encoded `.der` file. OpenSSL can convert these to `.pem`. These are not used very much outside of Windows.

## Java KeyStores and TrustStores

[Java KeyStore](https://docs.oracle.com/cd/E19509-01/820-3503/ggffo/index.html) (JKS) consists of a database containing a private key and an associated certificate, or an associated certificate chain. The certificate chain consists of the client certificate and one or more certification authority (CA) certificates. A TrustStore contains certificates of external hosts trusted by the client.

Both KeyStores and TrustStores are managed by means of a utility called `keytool`, which is a part of the Java SDK installation.

# Part 2 - Keys Management

Apache Kafka allows clients to connect over SSL. To enable SSL based security between Kafka brokers and clients, one has to follow below steps to generate the required self-signed certificates.

## Create CA

In this step a certificate authority (CA) will get created that will be responsible for signing the certificates. This step is important to prevent forged certificates.

```bash
openssl req -new -x509 -keyout ca.key -nodes -out ca.crt -days 365 \
-subj /C=IN/ST=Haryana/L=Gurgaon/O=Guavus/OU=MIQ/CN=www.guavus.com
```

The CA generated in the above step is simply a public-private key pair and certificate which is intended to sign other certificates.

## Add CA to Truststore

The next step is to add the generated CA to the client's truststore so that the clients can trust this CA. Importing a CA certificate into one's truststore means trusting all certificates that are signed by this certificate. This attribute is called the chain of trust.

```bash
keytool -keystore truststore.jks -alias CA -import -file ca.crt -storepass admin123 -keypass admin123 -noprompt
```

## Generate SSL Key and Certificate

Generate a public-private key pair into a key store so that we can export and sign it later with a CA. One should make sure that the common name (CN) matches exactly with the fully qualified domain name (FQDN) of the server. The client while connecting compares the CN with the DNS to ensure that it is connecting to the desired server.

```bash
keytool -keystore keystore.jks -alias localhost -validity 365 -genkey -keyalg RSA \
-dname 'CN=localhost, OU=MIQ, O=Guavus, L=Gurgaon, S=Haryana, C=IN' \
-storetype pkcs12 -storepass admin123 -keypass admin123 -noprompt
```

After the above step, the machine will have a public-private key pair and a certificate to identify the machine.

## Generate CSR

Before signing the certificates with the CA, one has to generate a certificate signing request (CSR) using the Keystore.

```bash
keytool -keystore keystore.jks -alias localhost -certreq -file client.csr \
-storepass admin123 -keypass admin123 -noprompt
```

## Sign CSR with CA

In this step, the CA present in the Truststore will sign the certificate present in the Keystore and as a result it will generate a `client.crt` file, which is the client's signed certificate. This step will also create a file called `ca.srl` which contains a unique serial number of the signed certificate.

```bash
openssl x509 -req -CA ca.crt -CAkey ca.key -in client.csr -out client.crt -days 365 -CAcreateserial
```

## Import CA into Keystore

```bash
keytool -keystore keystore.jks -alias CA -import -file ca.crt -storepass admin123 -keypass admin123 -noprompt
```

## Import Client's Signed Certificate into Keystore

Finally, import keystore's signed certificate back into the keystore.

```bash
keytool -keystore keystore.jks -alias localhost -import -file client.crt \
-storepass admin123 -keypass admin123 -noprompt
```

Apart from client's signed certificate i.e `client.crt` file, non Java based applications also needs the machine's private key to connect with the secure Kafka. For this one can make use of openssl to export it from the Keystore.

```bash
openssl pkcs12 -in keystore.jks -nodes -nocerts -out client.key -password pass:admin123
```

# Part 3 - Writing Go Client to connect to secure Kafka

This section describes the steps required to write a Go client that can successfully connect to a TLS enabled Kafka. 

## Import Kafka Consumer Dependencies

We start by importing dependencies for a minimal Kafka consumer:

```go
package main

import (
        "log"
        "time"

        "github.com/Shopify/sarama"
        cluster "github.com/bsm/sarama-cluster"
)
```

## main()

The minimal Kafka consumer is based on the [Sarama](https://github.com/bsm/sarama-cluster) Go library:

```go
func main() {
        config := cluster.NewConfig()
        config.Consumer.Offsets.Initial = sarama.OffsetOldest
        config.Consumer.Offsets.CommitInterval = time.Second
        brokers := []string{"localhost:9092"}
        topics := []string{"quickstart-events"}
        consumer, err := cluster.NewConsumer(brokers, "group", topics, config)
        if err != nil {
                panic(err)
        }
        defer consumer.Close()
        for i := 0; i < 10; i++ {
                select {
                case message := <-consumer.Messages():
                        log.Printf("%v-> %v\n", message.Key, message.Value)
                case err := <-consumer.Errors():
                        log.Println(err)
                }
        }
}
```

## Import Crypto Dependencies

We import TLS and x509 dependencies required to configure a TLS enabled Kafka client.

```go
import "crypto/tls"
import "crypto/x509"
import "io/ioutil"
```

## Load Certificate and Private Key

Load PEM encoded public/private key pair

```go

cert, err := tls.LoadX509KeyPair("client.crt", "client.key")
if err != nil {
        log.Fatalf("Error: %v", err)
}
```

## Load CA Certificate

Load and appends CA certificate generated to the certificate pool.

```go
caCert, err := ioutil.ReadFile(caCertFile)
            if err != nil {
                logg.Fatalf("Error: %v", err)
            }

caCertPool := x509.NewCertPool()
caCertPool.AppendCertsFromPEM(caCert)
```

## Configure TLS Client

A Config structure is used to configure a TLS client. 

```go
tlsConfig := tls.Config{}
tlsConfig.Certificates = []tls.Certificate{cert}
tlsConfig.RootCAs = caCertPool
tlsConfig.BuildNameToCertificate()

```

## Enable Secure Connection

Use TLS when connecting to the secure Kafka broker.

```go
config.Net.TLS.Enable = true
config.Net.TLS.Config = &tlsConfig
```
