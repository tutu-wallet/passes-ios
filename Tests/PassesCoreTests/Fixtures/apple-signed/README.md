# `apple-signed/` fixture

Real Apple-signed pkpass artifacts from a Tixly Chroniques ticket, copied verbatim from the
Android kernel's `apple-signed` fixture. Used by
`SignatureVerifierTests.realAppleSignedPkpassIsAppleVerified` to run the full production verifier
path against a wire shape Apple ships, on top of the bundled Apple Root CA anchor set.

## Why this fixture matters

The `SignerInfo` uses the bare `rsaEncryption` OID (`1.2.840.113549.1.1.1`, parameters absent)
for `signatureAlgorithm` and conveys the hash separately in `digestAlgorithm` (`sha256`).
swift-certificates 1.19.x does not recognize the bare OID, so without the
`normalizeCMSSignatureAlgorithm` pre-pass the verifier reports this valid pass as
`.tampered(.manifestSignatureMismatch)`. This fixture is the regression guard for that fix.

## What's here

- `manifest.json`: file digests, no PII. `sha256(manifest.json)` equals the signed
  `messageDigest` attribute, proving the content is untampered.
- `signature`: detached PKCS#7 / CMS blob. Embeds the Apple WWDR **G4** intermediate and the
  Tixly leaf certificate (CN `Pass Type ID: pass.com.tixly`, an issuer identity, not user data).
  Chain: leaf -> WWDR G4 (embedded) -> Apple Root CA (bundled anchor).

`pass.json` and the localized `*.lproj/pass.strings` files are deliberately not included: they
hold the passenger, event and ticket-number content, and the manifest digest comparison does not
need them.

## Shelf life

The Tixly leaf's `notAfter` is **2027-02-04T10:18:41 UTC**. iOS verifies under
`PermissivePolicy`, which drops the RFC 5280 expiry check, so the fixture stays `.appleVerified`
after the leaf expires as long as the chain still builds to the bundled Apple Root CA. It breaks
only if that anchor is unbundled or the embedded chain structure changes.

## Renewal procedure

1. Obtain any current Apple-signed pkpass (any vendor; Apple Wallet sample passes work).
2. Unzip it and copy `manifest.json` and `signature` here, replacing the existing files. Do not
   copy `pass.json`, `*.lproj/` or image assets: they carry PII and the verifier does not need
   them.
3. Keep the Android and iOS copies byte-identical (`shasum -a 256`).
4. Run `swift test --filter SignatureVerifierTests`.
