# Gjenerimi i Certifikatës Fiskalizimi ATK me OpenSSL dhe Postman

Ky dokument shpjegon procedurën për krijimin e CSR request, marrjen e verification code nga ATK, dërgimin e CSR për nënshkrim, verifikimin e certifikatës dhe konvertimin në `.pfx`.

> **Kujdes:** Keto te dhena jane te simuluara. filet `private_key.pem`, `signedCert.pfx`, password-at, CSR/certifikata reale, NUI real, ose verification code real.

---

## Hapi 1 — Install OpenSSL

Shkarko dhe instalo OpenSSL për Windows:

```text
https://slproweb.com/download/Win64OpenSSL-3_6_2.msi
```

Pas instalimit, kontrollo në Command Prompt ose PowerShell:

```bash
openssl version
```

---

## Hapi 2 — Krijo file `csr.conf`

Krijo një file me emrin:

```text
csr.conf
```

Vendos këtë përmbajtje:

```ini
[ req ]
prompt = no
distinguished_name = dn
default_md = sha256
string_mask = nombstr

[ dn ]
CN = DEMO BUSINESS SH.P.K.
L = 1
OU = 1
O = 123456789
C = XK
```

Preferohet të përdoret **Notepad++** dhe të ruhet si tekst normal.

### Legjenda

| Fusha | Përshkrimi |
|---|---|
| `CN` | Business Name |
| `L` | Branch ID |
| `OU` | POS ID |
| `O` | NUI / Business ID |
| `C` | Country code, për Kosovë: `XK` |

---

## Hapi 3 — Gjenero private key

Me OpenSSL krijo private key:

```bash
openssl ecparam -name prime256v1 -genkey -noout -out private_key.pem
```

Do të krijohet file:

```text
private_key.pem
```

> **E rëndësishme:** `private_key.pem` nuk duhet të ndahet me askënd dhe nuk duhet të publikohet në GitHub.

---

## Hapi 4 — Gjenero CSR

Gjenero CSR request:

```bash
openssl req -new -sha256 -key private_key.pem -out request.csr -config csr.conf
```

Do të krijohet file:

```text
request.csr
```

---

## Hapi 5 — Kontrollo përmbajtjen e CSR

Ky hap është opcional, por rekomandohet:

```bash
openssl req -in request.csr -noout -text
```

Shembull rezultati:

```text
Subject: CN = DEMO BUSINESS SH.P.K., L = 1, OU = 1, O = 123456789, C = XK
Public Key Algorithm: id-ecPublicKey
ASN1 OID: prime256v1
```

---

## Hapi 6 — Install Postman

Shkarko dhe instalo Postman:

```text
https://dl.pstmn.io/download/latest/win64
```

---

## Hapi 7 — Merre verification code nga ATK

Bëhet kërkesa POST në URL:

```http
POST https://fiskalizimi-test.atk-ks.org/ca/verify/123456789
```

### JSON Body

```json
{
  "fiscalization_no": "111222333444",
  "pos_id": 1,
  "branch_id": 1,
  "application_id": 999888777666
}
```

### Shembull përgjigje

```json
{
  "business_name": "DEMO BUSINESS SH.P.K.",
  "verification_code": "12345678901234567890",
  "error": {
    "code": "",
    "message": ""
  }
}
```

Ruaje vlerën:

```text
verification_code
```

Sepse do të përdoret në hapin tjetër.

---

## Hapi 8 — Përgatit CSR për JSON

Hape file-in `request.csr` me Notepad ose Notepad++.

CSR duket afërsisht kështu:

```text
-----BEGIN CERTIFICATE REQUEST-----
MIIBFAKECSRDATAEXAMPLEONLYDO_NOT_USE_IN_PRODUCTION
REPLACE_THIS_WITH_YOUR_REAL_GENERATED_CSR_CONTENT
-----END CERTIFICATE REQUEST-----
```

Për ta dërguar në JSON, çdo rresht duhet të ndahet me `\n`, brenda një string-u:

```text
-----BEGIN CERTIFICATE REQUEST-----
MIIBFAKECSRDATAEXAMPLEONLYDO_NOT_USE_IN_PRODUCTION
REPLACE_THIS_WITH_YOUR_REAL_GENERATED_CSR_CONTENT
-----END CERTIFICATE REQUEST-----
```

---

## Hapi 9 — Dërgo CSR për nënshkrim

Bëhet kërkesa POST në URL:

```http
POST https://fiskalizimi-test.atk-ks.org/ca/signcsr
```

### JSON Body

```json
{
  "business_name": "DEMO BUSINESS SH.P.K.",
  "business_id": 123456789,
  "branch_id": 1,
  "verification_code": "12345678901234567890",
  "pos_id": 1,
  "application_id": 999888777666,
  "csr": "-----BEGIN CERTIFICATE REQUEST-----\nMIIBFAKECSRDATAEXAMPLEONLYDO_NOT_USE_IN_PRODUCTION\nREPLACE_THIS_WITH_YOUR_REAL_GENERATED_CSR_CONTENT\n-----END CERTIFICATE REQUEST-----"
}
```

---

## Hapi 10 — Ruaje signed certificate

Pasi të dërgohet kërkesa, do të kthehet një përgjigje me `signed_certificate`.

Shembull:

```json
{
  "signed_certificate": "-----BEGIN CERTIFICATE-----\nMIICFAKECERTIFICATEEXAMPLEONLYDO_NOT_USE_IN_PRODUCTION\nREPLACE_THIS_WITH_CERTIFICATE_RETURNED_FROM_ATK\n-----END CERTIFICATE-----\n"
}
```

Në Notepad++ zëvendëso `\n` me rreshta të rinj.

Rezultati duhet të jetë në këtë format:

```text
-----BEGIN CERTIFICATE-----
MIICFAKECERTIFICATEEXAMPLEONLYDO_NOT_USE_IN_PRODUCTION
REPLACE_THIS_WITH_CERTIFICATE_RETURNED_FROM_ATK
-----END CERTIFICATE-----
```

Ruaje si:

```text
signedCert.pem
```

---

## Hapi 11 — Verifiko certifikatën me OpenSSL

Për të kontrolluar certifikatën:

```bash
openssl x509 -in signedCert.pem -noout -text
```

Nëse komanda shfaq detajet e certifikatës, file është në format të rregullt PEM.

---

## Hapi 12 — Konverto certifikatën në PFX

Për instalim në kompjuter ose për përdorim në aplikacion, konverto certifikatën në `.pfx`.

```bash
openssl pkcs12 -export -inkey private_key.pem -in signedCert.pem -out signedCert.pfx -password pass:ChangeMeStrongPassword1!
```

Do të krijohet file:

```text
signedCert.pfx
```

> Ndrysho `ChangeMeStrongPassword1!` me një password të fortë dhe ruaje në vend të sigurt.

---

## File që krijohen gjatë procesit

| File | Përshkrimi |
|---|---|
| `csr.conf` | Konfigurimi për CSR |
| `private_key.pem` | Private key |
| `request.csr` | Certificate Signing Request |
| `signedCert.pem` | Certifikata e nënshkruar nga ATK |
| `signedCert.pfx` | Certifikata në format PFX |

---

## Përmbledhje komandash

```bash krijo private key
openssl ecparam -name prime256v1 -genkey -noout -out private_key.pem
```bash krijo request
openssl req -new -sha256 -key private_key.pem -out request.csr -config csr.conf
```bash kontrollo request
openssl req -in request.csr -noout -text
```bash kontrollo signedCert 
openssl x509 -in signedCert.pem -noout -text
```bash konverto nga signedCert.pem ne signedCert.pfx
openssl pkcs12 -export -inkey private_key.pem -in signedCert.pem -out signedCert.pfx -password pass:ChangeMeStrongPassword1!
```
