# noble-ed25519 ![Node CI](https://github.com/paulmillr/noble-ed25519/workflows/Node%20CI/badge.svg) [![code style: prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square)](https://github.com/prettier/prettier)

Fastest JS implementation of [ed25519](https://en.wikipedia.org/wiki/EdDSA),
an elliptic curve that could be used for EDDSA signature scheme and X25519 ECDH key agreement.

Conforms to [RFC7748](https://datatracker.ietf.org/doc/html/rfc7748), [RFC8032](https://tools.ietf.org/html/rfc8032) and [ZIP215](https://zips.z.cash/zip-0215). Includes support for [ristretto255](https://ristretto.group): a technique for constructing prime order elliptic curve groups with non-malleable encodings.

Check out [the online demo](https://paulmillr.com/ecc).

### This library belongs to *noble* crypto

> **noble-crypto** — high-security, easily auditable set of contained cryptographic libraries and tools.

- No dependencies, one small file
- Easily auditable TypeScript/JS code
- Supported in all major browsers and stable node.js versions
- All releases are signed with PGP keys
- Check out [homepage](https://paulmillr.com/noble/) & all libraries:
  [secp256k1](https://github.com/paulmillr/noble-secp256k1),
  [ed25519](https://github.com/paulmillr/noble-ed25519),
  [bls12-381](https://github.com/paulmillr/noble-bls12-381),
  [hashes](https://github.com/paulmillr/noble-hashes)

## Usage

Use NPM in node.js / browser, or include single file from
[GitHub's releases page](https://github.com/paulmillr/noble-ed25519/releases):

> npm install @noble/ed25519

```js
// Common.js and ECMAScript Modules (ESM)
import * as ed from '@noble/ed25519';
// If you're using single file, use global variable: nobleEd25519

const privateKey = ed.utils.randomPrivateKey(); // 32-byte Uint8Array or string.
const message = 'deadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef';
(async () => {
  const publicKey = await ed.getPublicKey(privateKey);
  const signature = await ed.sign(message, privateKey);
  const isSigned = await ed.verify(signature, message, publicKey);
})();
```

To use the module with [Deno](https://deno.land),
you will need [import map](https://deno.land/manual/linking_to_external_code/import_maps):

- `deno run --import-map=imports.json app.ts`

- `app.ts`

    ```typescript
    import * as secp from "https://deno.land/x/secp256k1/mod.ts";
    const publicKey = ed.getPublicKey(ed.utils.randomPrivateKey());
    console.log(publicKey);
    ```
- `imports.json`

    ```json
    {
      "imports": {
        "crypto": "https://deno.land/std@0.119.0/node/crypto.ts"
      }
    }
    ```

## API

- [`getPublicKey(privateKey)`](#getpublickeyprivatekey)
- [`sign(message, privateKey)`](#signmessage-privatekey)
- [`verify(signature, message, publicKey)`](#verifysignature-message-publickey)
- [`getSharedSecret(privateKey, publicKey)`](#getsharedsecretprivatekey-publickey)
- [Curve25519](#curve25519)
- [Ristretto255](#ristretto255)
- [Helpers & Point](#helpers--point)

##### `getPublicKey(privateKey)`
```typescript
function getPublicKey(privateKey: Uint8Array | string | bigint): Promise<Uint8Array>;
```
- `privateKey: Uint8Array | string | bigint` will be used to generate public key. If you want to pass bigints,
  ensure they are Big-Endian.
- Returns `Promise<Uint8Array>`. Uses **promises**, because ed25519 uses SHA internally; and we're using built-in browser `window.crypto`, which returns `Promise`.

To generate ed25519 public key:

1. private key is hashed with sha512, then first 32 bytes are taken from the hash
2. 3 least significant bits of the first byte are cleared

- Use `Point.fromPrivateKey(privateKey)` if you want `Point` instance instead
- Use `Point.fromHex(publicKey)` if you want to convert hex / bytes into Point.
  It will use decompression algorithm 5.1.3 of RFC 8032.
- Use `utils.getExtendedPublicKey` if you need full SHA512 hash of seed

##### `sign(message, privateKey)`
```typescript
function sign(message: Uint8Array | string, privateKey: Uint8Array | string): Promise<Uint8Array>;
```
- `message: Uint8Array | string` - message which would be signed
- `privateKey: Uint8Array | string` - private key which will sign the hash
- Returns EdDSA signature. You can consume it with `Signature.fromHex()` method:
    - `Signature.fromHex(ed25519.sign(hash, privateKey))`

##### `verify(signature, message, publicKey)`
```typescript
function verify(
  signature: Uint8Array | string | Signature,
  message: Uint8Array | string,
  publicKey: Uint8Array | string | Point
): Promise<boolean>
```
- `signature: Uint8Array | string | Signature` - returned by the `sign` function
- `message: Uint8Array | string` - message that needs to be verified
- `publicKey: Uint8Array | string | Point` - e.g. that was generated from `privateKey` by `getPublicKey`
- Returns `Promise<boolean>`

Verifies signature. Compatible with [ZIP215](https://zips.z.cash/zip-0215): allows sigs/public keys bigger than curve prime, but less than 2^256.

##### `getSharedSecret(privateKey, publicKey)`

```typescript
function getSharedSecret(privateKey: Uint8Array | string | bigint, publicKey: Hex): Promise<Uint8Array>;
```

Converts ed25519 private / public keys to Curve25519 and calculates
Elliptic Curve Diffie Hellman (ECDH) with X25519.
Conforms to [RFC7748](https://datatracker.ietf.org/doc/html/rfc7748).

##### Curve25519

```js
const pub = ed25519.curve25519.scalarMultBase(privateKey);
const shared = ed25519.curve25519.scalarMult(privateKeyA, publicKeyB);
```

The library includes namespace `curve25519` that you could use to calculate
Curve25519 keys. It uses Montgomery Ladder specified in [RFC7748](https://datatracker.ietf.org/doc/html/rfc7748).

You cannot use ed25519 keys, because they are hashed with sha512. However, you can use
`Point#toX25519()` method on ed25519 public keys. See implementation of `ed25519.getSharedSecret` for details.

##### Ristretto255

To use Ristretto, simply use `fromRistrettoHash()` and `toRistrettoBytes()` methods.

```typescript
// The hash-to-group operation applies Elligator twice and adds the results.
ExtendedPoint.fromRistrettoHash(hash: Uint8Array | string): ExtendedPoint;

// Decode a byte-string s_bytes representing a compressed Ristretto point into extended coordinates.
ExtendedPoint.fromRistrettoBytes(bytes: Uint8Array | string): ExtendedPoint;

// Encode a Ristretto point represented by the point (X:Y:Z:T) in extended coordinates to Uint8Array.
ExtendedPoint#toRistrettoBytes(): Uint8Array
```

It extends Mike Hamburg's Decaf approach to cofactor elimination to support cofactor-8 curves such as Curve25519.

In particular, this allows an existing Curve25519 library to implement a prime-order group with only a thin abstraction layer, and makes it possible for systems using Ed25519 signatures to be safely extended with zero-knowledge protocols, with no additional cryptographic assumptions and minimal code changes.

##### Helpers & Point

`utils.randomPrivateKey()`

Returns cryptographically secure random `Uint8Array` that could be used as Private Key.

`utils.precompute(W = 8, point = Point.BASE)`

Returns cached point which you can use to `#multiply` by it.

This is done by default, no need to run it unless you want to
disable precomputation or change window size.

We're doing scalar multiplication (used in getPublicKey etc) with
precomputed BASE_POINT values.

This slows down first getPublicKey() by milliseconds (see Speed section),
but allows to speed-up subsequent getPublicKey() calls up to 20x.

You may want to precompute values for your own point.

`utils.TORSION_SUBGROUP`

The 8-torsion subgroup ℰ8. Those are "buggy" points, if you multiply them by 8, you'll receive Point.ZERO.

Useful to check implementations for signature malleability. See [the link](https://moderncrypto.org/mail-archive/curves/2017/000866.html)

`Point#toX25519`

You can use the method to use ed25519 keys for curve25519 encryption.

https://blog.filippo.io/using-ed25519-keys-for-encryption

```typescript
ed25519.CURVE.P // 2 ** 255 - 19
ed25519.CURVE.l // 2 ** 252 + 27742317777372353535851937790883648493
ed25519.Point.BASE // new ed25519.Point(Gx, Gy) where
// Gx = 15112221349535400772501151409588531511454012693041857206046113283949847762202n
// Gy = 46316835694926478169428394003475163141307993866256225615783033603165251855960n;


// Elliptic curve point in Affine (x, y) coordinates.
ed25519.Point {
  constructor(x: bigint, y: bigint);
  static fromHex(hash: string);
  static fromPrivateKey(privateKey: string | Uint8Array);
  toX25519(): Uint8Array; // Converts to Curve25519 u coordinate in LE form
  toRawBytes(): Uint8Array;
  toHex(): string; // Compact representation of a Point
  equals(other: Point): boolean;
  negate(): Point;
  add(other: Point): Point;
  subtract(other: Point): Point;
  multiply(scalar: bigint): Point;
}
// Elliptic curve point in Extended (x, y, z, t) coordinates.
ed25519.ExtendedPoint {
  constructor(x: bigint, y: bigint, z: bigint, t: bigint);
  static fromAffine(point: Point): ExtendedPoint;
  static fromRistrettoHash(hash: Uint8Array | string): ExtendedPoint;
  static fromRistrettoBytes(bytes: Uint8Array | string): ExtendedPoint;
  toRistrettoBytes(): Uint8Array;
  toAffine(): Point;
}
ed25519.Signature {
  constructor(r: bigint, s: bigint);
  toHex(): string;
}

// Precomputation helper
utils.precompute(W, point);
// returns { head, prefix, scalar, point, pointBytes }
utils.getExtendedPublicKey(privateKey);
```

## Security

Noble is production-ready.

1. No public audits have been done yet. Our goal is to crowdfund the audit.
2. It was developed in a very similar fashion to
[noble-secp256k1](https://github.com/paulmillr/noble-secp256k1), which **was** audited by a third-party firm.
3. It was fuzzed by [Guido Vranken's cryptofuzz](https://github.com/guidovranken/cryptofuzz),
no serious issues have been found. You can run the fuzzer by yourself to check it.

We're using built-in JS `BigInt`, which is "unsuitable for use in cryptography" as [per official spec](https://github.com/tc39/proposal-bigint#cryptography). This means that the lib is potentially vulnerable to [timing attacks](https://en.wikipedia.org/wiki/Timing_attack). But, *JIT-compiler* and *Garbage Collector* make "constant time" extremely hard to achieve in a scripting language. Which means *any other JS library doesn't use constant-time bigints*. Including bn.js or anything else. Even statically typed Rust, a language without GC, [makes it harder to achieve constant-time](https://www.chosenplaintext.ca/open-source/rust-timing-shield/security) for some cases. If your goal is absolute security, don't use any JS lib — including bindings to native ones. Use low-level libraries & languages.

We however consider infrastructure attacks like rogue NPM modules very important; that's why it's crucial to minimize the amount of 3rd-party dependencies & native bindings. If your app uses 500 dependencies, any dep could get hacked and you'll be downloading malware with every `npm install`. Our goal is to minimize this attack vector.

## Speed

Benchmarks done with Apple M1.

```
getPublicKey(utils.randomPrivateKey()) x 6,835 ops/sec @ 146μs/op
sign x 3,474 ops/sec @ 287μs/op
verify x 726 ops/sec @ 1ms/op
verifyBatch x 888 ops/sec @ 1ms/op
Point.fromHex decompression x 11,783 ops/sec @ 84μs/op
ristretto255#fromHash x 5,482 ops/sec @ 182μs/op
ristretto255 round x 5,621 ops/sec @ 177μs/op
curve25519.scalarMultBase x 1,042 ops/sec @ 959μs/op
ed25519.getSharedSecret x 801 ops/sec @ 1ms/op
```

Compare to alternative implementations:

```
# tweetnacl-fast@1.0.3
getPublicKey x 920 ops/sec @ 1ms/op # aka scalarMultBase
sign x 519 ops/sec @ 2ms/op

# ristretto255@0.1.1
getPublicKey x 877 ops/sec @ 1ms/op # aka scalarMultBase

# sodium-native@3.2.1, native bindings to libsodium, node.js-only
sodium-native#sign x 58,661 ops/sec @ 17μs/op
```

## Contributing

1. Clone the repository.
2. `npm install` to install build dependencies like TypeScript
3. `npm run compile` to compile TypeScript code
4. `npm run test` to run jest on `test/index.ts`

## License

MIT (c) 2019 Paul Miller [(https://paulmillr.com)](https://paulmillr.com), see LICENSE file.
