# TLS and Certificate Security Audit

## Overview

This audit evaluates the TLS and certificate security configurations of four websites using Qualys SSL Labs and OpenSSL. The purpose of the assessment is to identify differences in certificate trust, protocol support, cipher configuration, and overall TLS security posture.

The websites assessed were:

- tls-v1-1.badssl.com
- incomplete-chain.badssl.com
- github.com
- duq.edu

## 1. tls-v1-1.badssl.com




**SSL Labs Grade:** B

### Certificate Status
The certificate was valid and trusted. OpenSSL successfully verified the certificate chain with a return code of `0 (ok)`.

### Protocol Support
- TLS 1.2: Enabled
- TLS 1.1: Enabled
- TLS 1.0: Enabled
- TLS 1.3: Not supported
- Deprecated protocols present: Yes — TLS 1.0 and TLS 1.1

### Cipher Configuration
SSL Labs identified multiple legacy cipher suites, including CBC-based and 3DES cipher suites.

### Command-Line Verification
OpenSSL successfully connected using TLS 1.2 with the `ECDHE-RSA-AES128-GCM-SHA256` cipher suite. Certificate verification returned `OK`.

An additional attempt was made to force a TLS 1.1 connection. The local OpenSSL configuration did not permit TLS 1.1, so the deprecated protocol support was confirmed through SSL Labs rather than the local client.

### Severity
**Medium**

### Potential Impact
Supporting deprecated TLS versions and legacy cipher suites increases exposure to weaknesses associated with older cryptographic protocols. It also allows older clients to negotiate less secure configurations instead of requiring modern TLS protections.

### Recommended Fix
Disable TLS 1.0 and TLS 1.1 and remove legacy weak cipher suites. The server should be configured to support modern TLS versions and strong cipher suites.

## 2. incomplete-chain.badssl.com






**SSL Labs Grade:** B

### Certificate Status
The server's certificate chain was incomplete. Although the leaf certificate itself was valid, the server did not provide the complete chain needed for the client to establish a trusted path to the root certificate authority.

### Protocol Support
- TLS 1.2: Enabled
- TLS 1.1: Enabled
- TLS 1.0: Enabled
- TLS 1.3: Not supported
- Deprecated protocols present: Yes — TLS 1.0 and TLS 1.1

### Cipher Configuration
SSL Labs identified legacy cipher suites, including CBC-based, RSA, and 3DES suites.

### Command-Line Verification
OpenSSL successfully established a TLS 1.2 connection using the `ECDHE-RSA-AES128-GCM-SHA256` cipher suite. However, certificate verification failed with:

`unable to get local issuer certificate`

and:

`unable to verify the first certificate`

The final verification result was `Verify return code: 21 (unable to verify the first certificate)`. This independently confirmed the incomplete certificate-chain finding identified by SSL Labs.

### Severity
**Medium**

### Potential Impact
An incomplete certificate chain can prevent clients from successfully verifying the server's identity. Some browsers may recover by obtaining missing intermediate certificates, while other applications or clients may reject the connection or display certificate trust errors.

### Recommended Fix
Configure the web server to provide the complete certificate chain, including the required intermediate certificate or certificates, so clients can build a valid trust path to a trusted root certificate authority.ss




## 3. github.com

**SSL Labs Grade:** A+

### Certificate Status
GitHub's certificate was valid and trusted. SSL Labs reported no certificate-chain issues, and OpenSSL successfully verified the certificate chain.

### Protocol Support
- TLS 1.3: Enabled
- TLS 1.2: Enabled
- TLS 1.1: Disabled
- TLS 1.0: Disabled
- SSL 3: Disabled
- SSL 2: Disabled
- Deprecated protocols present: No

### Cipher Configuration
GitHub supports modern TLS 1.3 cipher suites, including AES-GCM and ChaCha20-Poly1305. SSL Labs also identified some older TLS 1.2 CBC and RSA cipher suites as weak.

### Command-Line Verification
OpenSSL successfully connected to GitHub using TLS 1.3 with the `TLS_AES_128_GCM_SHA256` cipher suite.

Certificate verification returned `OK` with a final `Verify return code: 0 (ok)`. The test confirmed that GitHub provides a complete certificate chain and supports modern TLS 1.3 connections.

### Severity
**Low**

### Potential Impact
GitHub demonstrated a strong overall TLS configuration. Deprecated TLS versions are disabled, the certificate chain is trusted, and modern TLS 1.3 is supported. The remaining legacy TLS 1.2 cipher support represents a comparatively low risk but could increase exposure for older clients that negotiate those suites.

### Recommended Fix
Maintain the current TLS 1.3 and certificate configuration. Where compatibility requirements allow, continue reducing support for legacy TLS 1.2 cipher suites and maintain regular certificate and TLS configuration reviews.





## 4. duq.edu

**SSL Labs Grade:** A-

### Certificate Status
The certificate was valid and trusted. SSL Labs reported no certificate-chain issues, and OpenSSL successfully verified the certificate chain.

### Protocol Support
- TLS 1.3: Not supported
- TLS 1.2: Enabled
- TLS 1.1: Disabled
- TLS 1.0: Disabled
- SSL 3: Disabled
- SSL 2: Disabled
- Deprecated protocols present: No

### Cipher Configuration
The server supports a limited set of strong TLS 1.2 cipher suites using ECDHE with AES-GCM. Forward Secrecy is supported.

### Command-Line Verification
OpenSSL successfully connected using TLS 1.2 with the `ECDHE-ECDSA-AES256-GCM-SHA384` cipher suite.

Certificate verification returned `OK` with a final `Verify return code: 0 (ok)`. The test confirmed a complete and trusted certificate chain.

### Severity
**Low**

### Potential Impact
The server has a strong TLS configuration with deprecated protocols disabled and strong cipher suites enabled. However, TLS 1.3 is not currently supported, so the server does not benefit from the security and efficiency improvements available in the newer protocol. SSL Labs also reported that HTTP Strict Transport Security (HSTS) is not enabled.

### Recommended Fix
Enable TLS 1.3 where infrastructure and compatibility requirements allow while maintaining TLS 1.2 support. HSTS should also be considered to help ensure browsers consistently access the site over HTTPS and reduce the risk of protocol downgrade or SSL-stripping attacks.


### Comparison Overview

The four assessments demonstrated how different TLS configuration choices can significantly affect a server's overall security posture. The two BadSSL targets intentionally demonstrated weaknesses, with tls-v1-1.badssl.com supporting deprecated TLS versions and legacy cipher suites, while incomplete-chain.badssl.com showed how a missing intermediate certificate can prevent clients from establishing a trusted certificate path. GitHub demonstrated the strongest overall configuration with an A+ rating, TLS 1.3 support, a trusted certificate chain, and deprecated protocols disabled. Duquesne University also maintained a strong configuration by disabling deprecated protocols and using strong TLS 1.2 cipher suites, although it did not support TLS 1.3 and did not have HSTS enabled. The testing showed that a valid certificate alone does not guarantee a secure TLS implementation because protocol versions, cipher suites, certificate-chain configuration, and additional HTTPS protections must also be considered. OpenSSL provided useful command-line confirmation of the SSL Labs findings and demonstrated how certificate and protocol issues appear from a client's perspective. Overall, maintaining modern TLS versions, trusted certificate chains, strong cipher suites, and appropriate HTTPS security controls reduces unnecessary exposure while improving the reliability of secure communications.