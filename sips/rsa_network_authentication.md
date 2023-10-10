|     Author     |           Title            |  Category  |       Status        | Dependency SIP |    Date    |
| -------------- | -------------------------- | ---------- | ------------------- | -------------- | ---------- |
| Matheus Franco | RSA Network Authentication | Networking | open-for-discussion | -              | 2023-10-08 |

## Summary

In the current setup, our protocol employs BLS signatures for message authentication in our peer-to-peer (P2P) network. However, this process is resource-intensive and significantly impacts the scalability of our network, especially when dealing with a large volume of messages. To address this challenge, this proposal suggests replacing BLS verification with RSA verification, aiming to enhance network scalability and reduce message processing time.

## Motivation

The primary motivation behind this proposal is the need to improve the scalability of our network. BLS verification, while secure, is a costly operation in terms of computational resources. As we anticipate an increase in the number of messages processed by our network, it is imperative to explore more efficient alternatives.

## Rationale

The rationale behind adopting RSA verification lies in its superior performance compared to BLS signatures. While BLS verification takes approximately 2300 microseconds, RSA verification completes in just $\approx$ 58 microseconds. This drastic reduction in processing time ensures that our network can handle a larger volume of messages within the same computational resources, enhancing overall network scalability without compromising security standards. Below, it's shown the results of a benchmark test in go for several authentication algorithms, using 4 cores of Apple M1 Pro processors.


<p align="center">
<img src="./images/rsa_network_authentication/asymmetric_scheme_performance.png"  width="50%" height="10%">
</p>

Other schemes also demonstrate better performance than BLS, like ECDSA and EdDSA. We chose RSA because operators already have a coupled RSA key and it showed great verification performance, which is the current bottleneck, even though the signing time is higher.

## Drawbacks

- Signature Size Concerns: Presently, BLS signatures occupy 96 bytes. Shifting to RSA with 2048-bit keys would expand signatures to 256 bytes ($\approx 2.6$ times larger). This enlargement could escalate the size of network messages, potentially affecting bandwidth and message transmission times.


## Specification

At present, the operator employs their BLS private key share for verification. However, the operator also possesses an RSA key, which would replace the BLS key for signing messages. All other nodes store both the BLS and RSA public keys of every node, eliminating the need for a key exchange protocol.

The following security and design considerations should be thoroughly addressed:

1. Key Length and Security: The security of RSA encryption is highly dependent on the length of the key used. For instance, shorter keys are more vulnerable to brute-force attacks. The [US National Institute of Standard and Technology (NIST)](https://www.nist.gov) approves a minimum of 2048-bit RSA keys. Check the first table of section 1.5 of their [Security Policy](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp4172.pdf), released in 2023 July, for this reference.
2. Padding Schemes: RSA signatures require the use of padding schemes to ensure security. Poorly implemented or outdated padding schemes can expose the system to padding oracle attacks, where an attacker can gain unauthorized access to encrypted data. Employing secure padding schemes, such as PKCS#1 v1.5 or PSS, is essential to prevent these vulnerabilities.
3. Compliance and Standards: Ensure that the implementation adheres to industry standards and best practices, such as those outlined by NIST and other relevant regulatory bodies. Compliance with established standards helps validate the security of the RSA implementation and provides assurance against common vulnerabilities and exploits.
4. Abstraction to handle Cryptographic Agility: While RSA is currently a widely accepted encryption and digital signature algorithm, the field of cryptography is constantly evolving. It's important to design the system with cryptographic agility in mind, allowing for the future transition to more secure algorithms if necessary. It's worth noting that RSA, despite its prevalence, is among the most targeted cryptographic schemes. For instance, elliptic curve cryptography is often recommended due to its enhanced security features. However, even these might fall by the wayside with the impending rise of quantum computers, compelling a transition to advanced alternatives like Dilithium or Falcon.

