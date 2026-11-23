# Bundled Apple Trust Anchors

This directory holds the certificates `SignatureVerifier` consults when classifying a PKCS#7
signature blob's chain. They load once as a static set (`AppleTrustAnchors.swift`); the parser
never fetches certificates over the network. A valid signature whose chain extends beyond what is
bundled here surfaces as `SignatureStatus.certChainIncomplete`.

## Trust anchors (root CAs)

These are the trust roots for chain validation. A chain that terminates at any of them
yields `SignatureStatus.appleVerified`.

| File | Subject | SHA-256 fingerprint | Validity |
|------|---------|---------------------|----------|
| `apple-root-ca.cer` | CN=Apple Root CA, OU=Apple Certification Authority, O=Apple Inc., C=US | `B0:B1:73:0E:CB:C7:FF:45:05:14:2C:49:F1:29:5E:6E:DA:6B:CA:ED:7E:2C:68:C5:BE:91:B5:A1:10:01:F0:24` | 2006-04-25 to 2035-02-09 |
| `apple-root-ca-g2.cer` | CN=Apple Root CA - G2, OU=Apple Certification Authority, O=Apple Inc., C=US | `C2:B9:B0:42:DD:57:83:0E:7D:11:7D:AC:55:AC:8A:E1:94:07:D3:8E:41:D8:8F:32:15:BC:3A:89:04:44:A0:50` | 2014-04-30 to 2039-04-30 |
| `apple-root-ca-g3.cer` | CN=Apple Root CA - G3, OU=Apple Certification Authority, O=Apple Inc., C=US | `63:34:3A:BF:B8:9A:6A:03:EB:B5:7E:9B:3F:5F:A7:BE:7C:4F:5C:75:6F:30:17:B3:A8:C4:88:C3:65:3E:91:79` | 2014-04-30 to 2039-04-30 |

## Known intermediates (WWDR)

These are added as additional intermediates so a signature blob that omits its
intermediate can still chain to a bundled root. They are not trust anchors on their
own: a chain that stops at one of these without reaching a root above is
`certChainIncomplete`, not `appleVerified`.

| File | Subject | SHA-256 fingerprint | Validity |
|------|---------|---------------------|----------|
| `apple-wwdr-g3.cer` | CN=Apple Worldwide Developer Relations Certification Authority, OU=G3, O=Apple Inc., C=US | `DC:F2:18:78:C7:7F:41:98:E4:B4:61:4F:03:D6:96:D8:9C:66:C6:60:08:D4:24:4E:1B:99:16:1A:AC:91:60:1F` | 2020-02-19 to 2030-02-20 |
| `apple-wwdr-g6.cer` | CN=Apple Worldwide Developer Relations Certification Authority, OU=G6, O=Apple Inc., C=US | `BD:D4:ED:6E:74:69:1F:0C:2B:FD:01:BE:02:96:19:7A:F1:37:9E:04:18:E2:D3:00:EF:A9:C3:BE:F6:42:CA:30` | 2021-03-17 to 2036-03-19 |

The legacy WWDR CA (G1, expired 2023-02-07) is intentionally not bundled: any signature chaining
to it predates 2023, and bundling an expired anchor would muddy the trust posture without buying
coverage.

## Sources

The five certificates were exported from the macOS system keychain and cross-checked against
the copies inside `Xcode.app`'s `DVTFoundation.framework`. The fingerprints match the values
Apple publishes at
<https://www.apple.com/certificateauthority/>.

A reviewer can re-derive any of these locally with:

```sh
openssl x509 -in <file>.cer -inform DER -noout -subject -fingerprint -sha256
```

## Updating

Adding or removing a root or intermediate (for example a future G7 WWDR) is a deliberate trust
decision: it changes the set of signers considered Apple-issued. The change must:

1. Update this file (table row, source, SHA-256 fingerprint).
2. Update `bundledTrustAnchorFilenames` and `bundledIntermediateFilenames` in
   `AppleTrustAnchors.swift`.
3. Pass `swift test`: `AppleTrustAnchorsTests` enforces that the bundled files load and the
   counts match.
