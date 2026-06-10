# Styx Token Manager — API hujjati (README)

Asaka bank uchun elektron raqamli imzo (ERI) klienti. Lokal kompyuterda ishlab, brauzerdagi sahifaga imzolash va sertifikat amallarini taqdim etadi. C# / .NET Framework 4.8.

## Ulanish

Klient ikkita kanalni ko'taradi va **ikkalasi ham bir xil amallarni bajaradi**:

| Kanal | Manzil | Yo'nalish | Method |
|-------|--------|-----------|--------|
| HTTP / HTTPS | `http://localhost:6210` / `https://localhost:6211` | `RawUrl` (masalan `/crypto/signMSG`) | POST (tana = JSON), OPTIONS = CORS |
| WebSocket | `ws://127.0.0.1:8181` | JSON ichidagi `function` maydoni | — |

Portlar `app.config` dan o'zgartiriladi (`httpPort`, `httpsPort`, `port`). CORS ochiq (`*`).

So'rov tanasi `Request` modeli JSON ko'rinishida (quyida har endpointda faqat kerakli maydonlar ko'rsatilgan). Javob asosan `HTTPResponse` JSON ko'rinishida.

---

## Endpointlar

### 1. getVersion
Klient o'rnatilganini va versiyasini tekshirish uchun.

- **Method:** POST `/crypto/getVersion` · WS `function: "getVersion"`
- **Qabul qiladi:** —
- **Qaytaradi:**
```json
{ "status": "success", "version": "1.2.3.4" }
```

### 2. getTokenStatus
Berilgan sertifikat tokenga ulanganmi yoki yo'qligini tekshiradi. *(faqat HTTP)*

- **Method:** POST `/crypto/getTokenStatus`
- **Qabul qiladi:**
```json
{ "cert_sn": "00AABBCC" }
```
- **Qaytaradi:**
```json
{ "status": "success", "errorCode": 0, "tokenStatus": "token present" }
```

### 3. getTokenSN
Ulangan token seriya raqamini, turini va undagi sertifikat/litsenziya holatini qaytaradi. BST/CBS tokenlarda CSR ham generatsiya qilishi mumkin.

- **Method:** POST `/crypto/getTokenSN` · WS `function: "getTokenSN"`
- **Qabul qiladi:**
```json
{ "token_type": "ePass/iKey", "status": 0 }
```
- **Qaytaradi:** (`var1` = token SN, `var2` = token turi/CSR, `var3` = sertifikat SN lar yoki litsenziya bayrog'i)
```json
{ "status": "success", "var1": "1234567890", "var2": "3", "var3": ["false"], "macaddresses": ["00-11-22-33-44-55"] }
```

### 4. getCertInfo
Tokendagi (yoki virtual) barcha sertifikatlar haqida to'liq ma'lumot (egasi, beruvchi, muddati, ochiq sertifikat).

- **Method:** POST `/crypto/getCertInfo` · WS `function: "getCertInfo"`
- **Qabul qiladi:** —
- **Qaytaradi:**
```json
{
  "status": "success", "errorCode": 0, "tokenStatus": "token present",
  "certInfos": [{
    "tokenSN": "123", "issuer": "CN=...", "notafter": "2026-01-01T00:00:00",
    "notbefore": "2024-01-01T00:00:00", "serialnumber": "00AABB",
    "thumbprint": "abc...", "subject": "CN=...", "pubCert": "BASE64..."
  }]
}
```

### 5. getCertAndTokenSN
getCertInfo bilan bir xil, lekin `subject` va `pubCert` siz — qisqaroq javob.

- **Method:** POST `/crypto/getCertAndTokenSN` · WS `function: "getCertAndTokenSN"`
- **Qabul qiladi:** —
- **Qaytaradi:**
```json
{
  "status": "success", "errorCode": 0, "tokenStatus": "token present",
  "certInfos": [{
    "tokenSN": "123", "issuer": "CN=...", "notafter": "2026-01-01T00:00:00",
    "notbefore": "2024-01-01T00:00:00", "serialnumber": "00AABB", "thumbprint": "abc..."
  }]
}
```

### 6. signMSG
Ma'lumotni ERI (PKCS#7/CMS) bilan imzolaydi. Bitta (`obj`) yoki bir nechta (`objArray`) xabarni imzolaydi. Asosiy imzolash funksiyasi.

- **Method:** POST `/crypto/signMSG` · WS `function: "signMSG"`
- **Qabul qiladi:**
```json
{ "obj": "Salom dunyo", "detached": false, "cms": false, "silent": false, "cert_sn": "00AABB" }
```
- **Qaytaradi:**
```json
{ "status": "success", "errorCode": 0, "signedMsg": "BASE64_CMS", "signedMsgArray": null }
```

### 7. cryptoSign
signMSG bilan aynan bir xil amal. *(faqat WebSocket)*

- **Method:** WS `function: "cryptoSign"`
- **Qabul qiladi / qaytaradi:** signMSG bilan bir xil.

### 8. verifyCMS
CMS/PKCS#7 imzosini tekshiradi. **Diqqat:** JSON emas, oddiy matn qaytaradi.

- **Method:** POST `/crypto/verifyCMS` (tana = Base64 CMS) · WS `function: "verifyCMS"` (`obj` = Base64 CMS)
- **Qabul qiladi:**
```json
{ "function": "verifyCMS", "obj": "BASE64_CMS" }
```
- **Qaytaradi:** `True` (imzo to'g'ri) yoki `False` (noto'g'ri).

### 9. getCertificate
Virtual (cloud) sertifikat olishning ikki bosqichli OTP oqimi. 1-bosqich (`password` bo'sh) — telefonga SMS-OTP yuboradi. 2-bosqich (`password` = OTP) — sertifikatni tokenga import qiladi.

- **Method:** POST `/crypto/getCertificate` · WS `function: "getCertificate"`
- **Qabul qiladi:**
```json
// 1-bosqich
{ "phone": "998901234567", "inn": "123456789", "password": "" }
// 2-bosqich
{ "phone": "998901234567", "inn": "123456789", "cert_sn": "00AABB", "password": "123456", "token_sn": "123" }
```
- **Qaytaradi:**
```json
{ "result": 1, "status": "success", "comments": "SMS sent" }
{ "status": "success", "comments": "The certificate was successfully imported into the token!" }
```

### 10. pfxToToken / importCert
Base64 kodlangan PFX (PKCS#12) sertifikatni tokenga import qiladi.

- **Method:** POST `/crypto/pfxToToken` · WS `function: "importCert"`
- **Qabul qiladi:**
```json
{ "obj": "BASE64_PFX", "token_sn": "123", "token_type": "ePass/iKey", "password": "pfx_parol" }
```
- **Qaytaradi:**
```json
{ "status": "success", "comments": "Сертификат успешно импортирован в токен." }
```

### 11. licToToken
Tokenga AFRL litsenziyasini (Base64) yozadi.

- **Method:** POST `/crypto/licToToken` · WS `function: "licToToken"`
- **Qabul qiladi:**
```json
{ "obj": "BASE64_LICENSE" }
```
- **Qaytaradi:**
```json
{ "status": "success2", "comments": "Лицензия успешно сгенерировано!" }
```

### 12. clearToken
Tokenni tozalaydi — sertifikat/kalit obyektlarini o'chiradi (AFRL litsenziya saqlanadi). *(faqat WebSocket)*

- **Method:** WS `function: "clearToken"`
- **Qabul qiladi:**
```json
{ "token_type": "ePass/iKey", "status": 0 }
```
- **Qaytaradi:**
```json
{ "status": "success", "comments": "Токен успешно очищен" }
```

### 13. hashKey / HashKey
INN/PINFL va ma'lumot asosida hash hisoblab imzolaydi (iABS/SAP integratsiyasi uchun). `only_formated_response` qiymatiga qarab javob `HTTPResponse` yoki `SAPResponse` formatida bo'ladi.

- **Method:** POST `/crypto/HashKey` · WS `function: "hashKey"`
- **Qabul qiladi:**
```json
{ "inn": "123456789", "pinfl": "12345678901234", "obj": "imzolanadigan_matn", "only_formated_response": false }
```
- **Qaytaradi:**
```json
// HTTPResponse
{ "status": "success", "signedMsg": "BASE64" }
// SAPResponse (only_formated_response: true)
{ "Status": 0, "Message": "...", "SignMSG": "BASE64", "SignMSGArray": null }
```

### 14. getVirtualCSR
Virtual token uchun PKCS#10 CSR (sertifikat so'rovi) generatsiya qiladi. **Diqqat:** `Request` emas, to'g'ridan-to'g'ri `Subject` JSON ni qabul qiladi. *(faqat HTTP)*

- **Method:** POST `/crypto/getVirtualCSR`
- **Qabul qiladi:**
```json
{ "commonName": "ISM FAMILIYA", "uzINN": "123456789", "uzPINFL": "12345678901234", "country": "UZ", "organisation": "..." }
```
- **Qaytaradi:** (`var1` = CSR, `var2` = fayl nomi)
```json
{ "status": "success", "comments": "Запрос успешно сгенерирован.", "var1": "CSR...", "var2": "abc123random", "macaddresses": ["00-11-22-33-44-55"] }
```

### 15. certToDB
Virtual sertifikatni lokal SQLite bazaga (`request.db`) saqlaydi. *(faqat HTTP)*

- **Method:** POST `/crypto/certToDB`
- **Qabul qiladi:**
```json
{ "pin": "1234", "obj": "BASE64_CERT", "container": "filename", "subject": { "commonName": "ISM", "uzINN": "123456789" } }
```
- **Qaytaradi:**
```json
{ "status": "success", "comments": "Certificate inserted." }
```

### 16. deleteKey
Virtual sertifikatni o'chiradi: PIN tekshiriladi, KMS serverda holati yangilanadi, so'ng lokal baza va registrydan o'chiriladi. *(faqat HTTP)*

- **Method:** POST `/crypto/deleteKey`
- **Qabul qiladi:**
```json
{ "cert_sn": "00AABB", "pin": "1234" }
```
- **Qaytaradi:**
```json
{ "status": "success", "comments": "Certificate deleted." }
```

### 17. auth/status
ActiveX komponentlar uchun avtorizatsiya holatini tekshiradi. *(faqat HTTP)*

- **Method:** GET/POST `/auth/status`
- **Qabul qiladi:** —
- **Qaytaradi:**
```json
{ "status": "ok", "result": "true", "comment": "Authenticated, expires in 14.3 minutes" }
{ "status": "ok", "result": "false", "comment": "Not authenticated" }
```

### 18. auth/request
ActiveX komponentdan avtorizatsiya so'raydi — PIN dialogini ko'rsatadi yoki mavjud sessiyani qayta ishlatadi. *(faqat HTTP)*

- **Method:** POST `/auth/request`
- **Qabul qiladi:** —
- **Qaytaradi:**
```json
{ "status": "ok", "result": "true", "comment": "<autentifikatsiya xabari>" }
```

---

## Xatolik kodlari (`errorCode`)

| Kod | Ma'nosi |
|-----|---------|
| 0 | Xato yo'q (success) |
| 100 | So'rovni JSON ga o'girishda xato |
| 101 | TokenStatus ichki xatosi |
| 102 | Sertifikat o'qish xatosi / PINFL sertifikatga mos kelmadi |
| 104 | getCertInfo umumiy xatolik |
| 105 | Sertifikat muddati tugagan / INN va PINFL bo'sh |
| 107 | Token mavjud emas (Token not present) |
| 108 | Token ulanmagan (Token not connected) |
| 109 | Autentifikatsiya bekor qilindi yoki muvaffaqiyatsiz |

## `status` qiymatlari

`success` · `error` · `cancelled` · `success2` (litsenziya amallari uchun) · `ok` (auth endpointlari uchun).



# Admin PowerShell da ishga tushiring:
Start-Process "C:\Program Files\StyxClient\Register-ActiveX.bat" -Verb RunAs -Wait


Oldingi tekshiruv buyrug'ini qayta ishlatsa bo'ladi:


$dlls = @("SxCertCheckCntrl", "SxContViewCntrl", "SxWizardCntrl")
foreach ($dll in $dlls) {
    $found = Get-ChildItem "HKLM:\SOFTWARE\Classes" -ErrorAction SilentlyContinue | Where-Object { $_.PSChildName -match $dll }
    if ($found) {
        Write-Host "REGISTERED: $dll"
    } else {
        Write-Host "NOT REGISTERED: $dll"
    }
}
