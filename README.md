<img align="right" width="250" height="47" src="Gematik_Logo_Flag_With_Background.png"/> <br/> 

# Using Brainpool Curves in Elliptic Curve Cryptography for the Telematics Infrastructure

<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
       <ul>
        <li><a href="#release-notes">Release Notes</a></li>
      </ul>
	</li>
    <li><a href="#java-and-springboot">Java and SpringBoot</a></li>
    <li><a href="#netty">Netty</a></li>
    <li>
      <a href="#e-rezept">E-Rezept</a>
       <ul>
        <li><a href="#android">Android</a></li>
        <li><a href="#ios">iOS</a></li>
      </ul>
	</li>
    <li><a href="#license">License</a></li>
  </ol>
</details>

## About The Project

In order to migrate from cryptographic methods using RSA to those that use ECC in the Telematics Infrastructure (TI) our specifications require the use of brainpool curves. Unfortunately many software frameworks do not support brainpool curve parameters according to RFC-5639. In this article we provide code examples and links to projects in which cryptographic methods using ECC and brainpool curves are applied. They are intended to serve as a guideline for producers of TI products struggling with the implementation. In the medium-term gematik will allow the use of NIST curves in the TI which are supported by software frameworks more frequently.

We provide examples for the following programming languages and tool frameworks:
- [Java and SpringBoot](#java-and-springboot)
- [Netty](#netty)
- [Android](#android)
- [iOS](#ios)

### Release Notes
See [ReleaseNotes.md](./ReleaseNotes.md) for all information regarding the (newest) releases.

## Java and SpringBoot

Example code for Java (plain) and Spring Boot can be found in our project PKI Testsuite (https://github.com/gematik/app-PkiTestsuite). We use Bouncy Castle (https://www.bouncycastle.org/) in order to make Java-based applications support ECC with Brainpool curves. Our configuration ensures that the BouncyCastle SecurityProviders are registered with the application and are first in the list of SecurityProviders. Additionally PKIX has to be set as algorithm for the KeyManagerFactory and the system property jdk.tls.namedGroups needs to be set to the following value: brainpoolP256r1, brainpoolP384r1, brainpoolP512r1, secp256r1, secp384r1

In our example code the configuration is placed into a static code block.

<pre><code>
  static {
      System.setProperty("jdk.tls.namedGroups", "brainpoolP256r1, brainpoolP384r1, brainpoolP512r1, secp256r1, secp384r1");
      Security.setProperty("ssl.KeyManagerFactory.algorithm", "PKIX");
      Security.removeProvider(BouncyCastleProvider.PROVIDER_NAME);
      Security.insertProviderAt(new BouncyCastleProvider(), 1);
      Security.removeProvider(BouncyCastleJsseProvider.PROVIDER_NAME);
      Security.insertProviderAt(new BouncyCastleJsseProvider(), 2);
  }
</code></pre>  

An example on how to configure a Java TLS client for ECC with Brainpool curves can be found at https://github.com/gematik/app-PkiTestsuite/blob/2.2.0/pkits-tls-client/src/main/java/de/gematik/pki/pkits/tls/client/TlsConnection.java.

A TLS server example using SpringBoot is available under https://github.com/gematik/app-PkiTestsuite/blob/2.2.0/pkits-sut-server-sim/src/main/java/de/gematik/pki/pkits/sut/server/sim/PkiSutServerSimApplication.java.

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

## License
Copyright 2025 gematik GmbH

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License.

See the [LICENSE](./LICENSE) for the specific language governing permissions and limitations under the License.

## Additional Notes and Disclaimer from gematik GmbH

1. Copyright notice: Each published work result is accompanied by an explicit statement of the license conditions for use. These are regularly typical conditions in connection with open source or free software. Programs described/provided/linked here are free software, unless otherwise stated.
2. Permission notice: Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions::
    1. The copyright notice (Item 1) and the permission notice (Item 2) shall be included in all copies or substantial portions of the Software.
    2. The software is provided "as is" without warranty of any kind, either express or implied, including, but not limited to, the warranties of fitness for a particular purpose, merchantability, and/or non-infringement. The authors or copyright holders shall not be liable in any manner whatsoever for any damages or other claims arising from, out of or in connection with the software or the use or other dealings with the software, whether in an action of contract, tort, or otherwise.
    3. The software is the result of research and development activities, therefore not necessarily quality assured and without the character of a liable product. For this reason, gematik does not provide any support or other user assistance (unless otherwise stated in individual cases and without justification of a legal obligation). Furthermore, there is no claim to further development and adaptation of the results to a more current state of the art.
3. Gematik may remove published results temporarily or permanently from the place of publication at any time without prior notice or justification.
4. Please note: Parts of this code may have been generated using AI-supported technology.’ Please take this into account, especially when troubleshooting, for security analyses and possible adjustments.

## Contact
We take open source license compliance very seriously. We are always striving to achieve compliance at all times and to improve our processes. 
This software is currently being tested to ensure its technical quality and legal compliance. Your feedback is highly valued. 
If you find any issues or have any suggestions or comments, or if you see any other ways in which we can improve, please reach out to: OSPO@gematik.de
