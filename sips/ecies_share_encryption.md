| Author      | Title                  | Category | Status |
|-------------|------------------------|----------|--------|
| Alon Muroch | ECIES Share Encryption | Core     | open-for-discussion  |

[Discussion](https://github.com/bloxapp/SIPs/discussions/19)

## Overview
When registering a validator for the SSV network the user needs to provide 2f+1 encrypted shares for the selected operators.  
The registration happens in a smart contract for data availability and synchronization.

Currently, RSA is used for encrypting the shares. We suggest replacing RSA with ECIES secp256k1 based curve.

## Background - Elliptic Curve Diffie Hellman(ECDH) 
`is anonymous key agreement scheme, which allows two parties, each having an elliptic-curve public–private key pair, to establish a shared secret over an insecure channel.`
[Source](https://cryptobook.nakov.com/asymmetric-key-ciphers/ecdh-key-exchange)

Shared Secret  = (a * G) * b = (b * G) * a  
        = alicePK * bobSK = bobPK * aliceSK 

1.Alice generates a random ECC key pair: {alicePrivKey, alicePubKey = alicePrivKey * G}
2.Bob generates a random ECC key pair: {bobPrivKey, bobPubKey = bobPrivKey * G}
3.Alice and Bob exchange their public keys through the insecure channel (e.g. over Internet)  
4.Alice calculates sharedKey = bobPubKey * alicePrivKey  
5.Bob calculates sharedKey = alicePubKey * bobPrivKey  
6.Now both Alice and Bob have the same sharedKey == bobPubKey * alicePrivKey == alicePubKey * bobPrivKey

![](https://60896510-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F-LhlOQMrG9bRiqWpegM0%2Fuploads%2Fgit-blob-19769dbf2cf56f28a6fab0b90fb10a8ab1506874%2Fecies.png?alt=media)

## Why?
* Smaller Keys and ciphers [comparable](https://www.ssl2buy.com/wiki/rsa-vs-ecc-which-is-better-algorithm-for-security) security
* [Faster encryption](https://www.researchgate.net/figure/Encryption-time-comparison-between-ECIES-and-RSA-AES_fig5_277941706)
* Same crypto primitives as ethereum, using [go-ethereum's implementation](https://github.com/ethereum/go-ethereum/blob/master/crypto/ecies/ecies.go)
* Aligned to same crypto primitives as used in [SSV DKG](https://github.com/bloxapp/SIPs/blob/main/sips/dkg.md)

## Specifications
[code](https://github.com/bloxapp/ssv-experiments/blob/master/ecies/ecies_test.go)

### Operator shares
Each operator, on registration, uploads his encryption pubkey

```go
import "github.com/ethereum/go-ethereum/crypto/ecies"
import "github.com/ethereum/go-ethereum/crypto"
import "crypto/rand"

prv1, err := ecies.GenerateKey(rand.Reader, crypto.S256(), nil)
pkBytes := crypto.CompressPubkey(&prv1.ExportECDSA().PublicKey)
```

### Encrypt

```go
import "github.com/ethereum/go-ethereum/crypto/ecies"
import "github.com/ethereum/go-ethereum/crypto"
import "crypto/rand"

func Encrypt(plain []byte, pubKey []byte) ([]byte, error) {
    pkECDSA, err := crypto.DecompressPubkey(pubKey)
    pk := ecies.ImportECDSAPublic(pkECDSA)
    ct, err := ecies.Encrypt(rand.Reader, pk, plain, nil, nil)
    return ct
}
```

### Decrypt

```go
import "github.com/ethereum/go-ethereum/crypto/ecies"
import "github.com/ethereum/go-ethereum/crypto"
import "crypto/rand"
import "math/big"

func privKeyECIESFromByts(privKey []byte) *ecies.PrivateKey {
    n := &big.Int{}
    n.SetBytes(privKey)
    skECDSA, _ := crypto.ToECDSA(privKey)
    return ecies.ImportECDSA(skECDSA)
}

func DecryptECIES(ct []byte, privKey []byte) []byte {
    sk := privKeyECIESFromByts(privKey)
    pt, _ := sk.Decrypt(ct, nil, nil)
    return pt
}
```

## Size Comparison

| Protocol    | Cipher Text (For encrypting BLS12381 secret key) |
|-------------|--------------------------------------------------|
| RSA 2048    | 256 bytes                                        |
| ECIES secp256k1 | 145 bytes                                        |