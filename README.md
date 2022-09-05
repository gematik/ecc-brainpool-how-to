![gematik logo](images/gematik_logo.png)

# Using Brainpool Curves in Elliptic Curve Cryptography for the Telematics Infrastructure

In order to migrate from cryptographic methods using RSA to those that use ECC in the Telematics Infrastructure (TI) our specifications require the use of brainpool curves. Unfortunately many software frameworks do not support brainpool curve parameters according to RFC-5639. In this article we provide code examples and links to projects in which cryptographic methods using ECC and brainpool curves are applied. They are intended to serve as a guideline for producers of TI products struggling with the implementation. In the medium-term gematik will allow the use of NIST curves in the TI which are supported by software frameworks more frequently.

We provide examples for the following programming languages and tool frameworks:
- [Java and SpringBoot](#java-and-springboot)
- [Netty](#netty)
- [Android](#android)
- [iOS](#ios)

## Java and SpringBoot

Example code for Java (plain) and Spring Boot can be found in our project PKI Testsuite (https://github.com/gematik/app-PkiTestsuite). We use Bouncy Castle (https://www.bouncycastle.org/) in order to make Java-based applications support ECC with Brainpool curves. Our configuration ensures that the BouncyCastle SecurityProviders are registered with the application and are first in the list of SecurityProviders. Additionally PKIX has to be set as algorithm for the KeyManagerFactory and the system property jdk.tls.namedGroups needs to be set to the following value: brainpoolP256r1, brainpoolP384r1, brainpoolP512r1

In our example code the configuration is placed into a static code block.

    static {
        System.setProperty("jdk.tls.namedGroups", "brainpoolP256r1, brainpoolP384r1, brainpoolP512r1");
        Security.setProperty("ssl.KeyManagerFactory.algorithm", "PKIX");
        Security.removeProvider(BouncyCastleProvider.PROVIDER_NAME);
        Security.insertProviderAt(new BouncyCastleProvider(), 1);
        Security.removeProvider(BouncyCastleJsseProvider.PROVIDER_NAME);
        Security.insertProviderAt(new BouncyCastleJsseProvider(), 2);
    }

An example on how to configure a Java TLS client for ECC with Brainpool curves can be found at https://github.com/gematik/app-PkiTestsuite/blob/master/pkits-tls-client/src/main/java/de/gematik/pki/pkits/tls/client/TlsConnection.java.

A TLS server example using SpringBoot is available under https://github.com/gematik/app-PkiTestsuite/blob/master/pkits-sut-server-sim/src/main/java/de/gematik/pki/pkits/sut/server/sim/PkiSutServerSimApplication.java.

## Netty
Configuring Netty for Elliptic Curve Cryptography with brainpool curves is pretty straight forward using the BouncyCastle library (Provider and DTLS/TLS API/JSSE Provider, see [Bouncy Castle Latest Java Releases](https://www.bouncycastle.org/latest_releases.html)). The code excerpt below shows how an SslContext object can be generated that supports the cipher suites currently required by our specifications:

<pre><code>
    /**
     * Generates an SslObject using BouncyCastle as TLS provider for ECC and brainpool curves.
     * @param keyManagerFactory KeyManagerFactory for the server key material.
     * @param trustManagerFactory TrustManagerFactory for the trust store.
     * @return An SslObject using BouncyCastle as TLS provider for ECC and brainpool curves.
     * @throws SSLException In case the SslObject cannot be created.
     */
    public SslContext createSslContextForEccBrainpool(KeyManagerFactory keyManagerFactory,
                                                      TrustManagerFactory trustManagerFactory) throws SSLException {
        // FIPS mode is not required when instantiating the BouncyCastleJsseProvider (b=false)
        Provider sslContextProvider = new BouncyCastleJsseProvider(false, new BouncyCastleProvider());
        List<String> cipherSuiteList = Arrays.asList("TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384",
                "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256");
        return SslContextBuilder.forServer(keyManagerFactory)
                .sslProvider(SslProvider.JDK)
                .sslContextProvider(sslContextProvider)
                .trustManager(trustManagerFactory)
                .ciphers(cipherSuiteList)
                .clientAuth(ClientAuth.REQUIRE)
                .build();
    }
</code></pre>

From the SslContext that is created that way an SslEngine can be obtained using one of the *newEngine* methods. The SslEngine object can in turn be used to create an SslHandler.

Please note that the KeyManagerFactoryObject should also be created using the BouncyCastleTlsProvider.

<pre><code>
    KeyManagerFactory.getInstance("PKIX", new BouncyCastleJsseProvider(false, new BouncyCastleProvider());
</code></pre>

## E-Rezept
### Android

All cryptographic operations in the E-Rezept Android App (https://github.com/gematik/E-Rezept-App-Android) are implemented with [BouncyCastle](https://www.bouncycastle.org/).

- The VAU (Vertrauenswürdigen Ausführungsumgebung) bundles some useful utility functions for handling certificates and ocsp responses in general: [CertUtils.kt, OCSPUtils.kt, ...](https://github.com/gematik/E-Rezept-App-Android/tree/master/android/src/main/java/de/gematik/ti/erp/app/vau). The actual ECIES implementation can be found in Crypto.kt
- NFC resides in [.../de/gematik/ti/erp/app/cardwall/model/nfc](https://github.com/gematik/E-Rezept-App-Android/tree/master/android/src/main/java/de/gematik/ti/erp/app/cardwall/model/nfc) and is used in [AuthenticationUseCaseProduction.kt](https://github.com/gematik/E-Rezept-App-Android/blob/master/android/src/main/java/de/gematik/ti/erp/app/cardwall/usecase/AuthenticationUseCaseProduction.kt)
- Token encryption utilizes [jose4j](https://bitbucket.org/b_c/jose4j/wiki/Home). To enable ECC Brainpool support we inject the curves manually through reflection in [EllipticCurvesExtending.kt](https://github.com/gematik/E-Rezept-App-Android/blob/master/android/src/main/java/de/gematik/ti/erp/app/idp/EllipticCurvesExtending.kt)

### iOS
For cryptographic operations that are not provided by the SDK (for example that involve Brainpool*-curves) the E-Rezept iOS App (https://github.com/gematik/E-Rezept-App-iOS) use a [Swift extension wrapper](https://github.com/gematik/OpenSSL-Swift) with embedded OpenSSL.

This Xcode-project downloads, compiles and embeds a current OpenSSL version 3.x.y in a Swift framework that can be included in MacOS/iOS Frameworks and Apps. Supported actions include:
- X.509 Certificate
- OCSP Responses
- Cryptographic Message Syntax (CMS)
- Key Management
- ECDH Shared Secret computation
- MAC calculation

For usage examples refer to: https://github.com/gematik/OpenSSL-Swift/blob/master/README.md

We make usage of our framework in multiple cases:
- [VAU (vertrauenswürdigen Ausführungsumgebung)](https://github.com/gematik/E-Rezept-App-iOS/blob/master/Sources/VAU/internal/VAUCrypto.swift)
- [Kartenkommunikation (PACE)](https://github.com/gematik/ref-OpenHealthCardKit/blob/master/Sources/HealthCardControl/SecureMessaging/KeyAgreement.swift)
- [Tokenverschlüsselung](https://github.com/gematik/E-Rezept-App-iOS/blob/master/Sources/IDP/internal/IDPCrypto.swift)

References:
- https://github.com/gematik/OpenSSL-Swift
- https://github.com/gematik/E-Rezept-App-iOS
- https://github.com/gematik/ref-OpenHealthCardKit
