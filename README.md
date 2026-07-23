# E-IMZO Integratsiya Qo'llanmasi — Web, Mobil va Backend

> **Biz o'zimiz orzu qilgan amaliy qo'llanma.** E-IMZO'ning rasmiy hujjatlari qisqa va tarqoq: bir qismi GitHub repolarда, bir qismi JS demoda, bir qismi Java namunasida. Bu qo'llanma sizga haqiqatan kerak bo'lgan hamma narsani — tushunchalar, aniq kontraktlar, ishlaydigan kod, va bizni kunlab qiynagan xatoliklar — bir joyga to'playdi. Shunda siz E-IMZO'ni web, mobil va backend'ga qiynalmasdan ulaysiz.
>
> **Kimlar uchun:** hamma daraja uchun. §0–§2 — hammaga (asosiy model + hamma xato qiladigan bitta narsa). §3–§7 — har bir yuza bo'yicha ulash qadamlari. §8 — xatolarni topish yo'riqnomasi. Oldindan GOST/kriptografiya bilimi shart emas.

---

## 0. Qisqacha (avval shuni o'qing)

**E-IMZO nima:** O'zbekistonning milliy elektron imzo tizimi (Soliq qo'mitasi). Foydalanuvchi o'zining **ID-kartasi** (yoki saqlangan kaliti) bilan imzolaydi, siz esa tekshirsa bo'ladigan **PKCS#7** imzoni olasiz.

**Oltin qoida:** **imzoni siz o'zingiz yasamaysiz.** PKCS#7'ni E-IMZO yasaydi — brauzerда (lokal plagin orqali) yoki davlat mobil ilovasi ichida. Sizning vazifangiz faqat: (1) E-IMZO'ga hujjatning **hash**ini berish, va (2) u qaytargan PKCS#7'ni **tekshirish**.

**Uchta ulash yuzasi:**

| Yuza | Imzoni kim yasaydi | Sizga qanday yetadi |
|---|---|---|
| **Web** | E-IMZO brauzer plagini (lokal, `CAPIWS`) | JS PKCS#7'ni backend'ingizga POST qiladi |
| **Mobil** | E-IMZO ID-Card Mobile ilovasi | E-IMZO serveri PKCS#7'ni sizning **callback URL**ingizga POST qiladi |
| **Backend** | — (siz tekshirasiz) | Siz `{hujjat, pkcs7}`ni **e-imzo-server**ga tekshirish uchun yuborasiz |

**Hamma xato qiladigan #1 narsa:** hash algoritmi. U — **O'z DSt 1106**, ya'ni **CryptoPro parametrlari bilan GOST R 34.11-94** — SHA-256 EMAS. Buni noto'g'ri qilsangiz, har bir imzo jimgina verify'dan o'tmaydi. §2'ni o'qing — bu eng muhim bo'lim.

---

## 1. Qismlar

- **ID-karta** — foydalanuvchining sertifikati + maxfiy kaliti turgan jismoniy smart-karta. Imzolash *karta ichida* bo'ladi (PIN + NFC/o'quvchi).
- **E-IMZO brauzer plagini** — lokal ilova, WebSocket API (`CAPIWS`, `wss://127.0.0.1:64322`) ochadi; web sahifalar uni `e-imzoapi.js` orqali boshqaradi. Imzoni kompyuterда yasaydi.
- **E-IMZO ID-Card Mobile ilovasi** — davlat mobil ilovasi. `eimzo://` deep-link orqali ochiladi, hashni ID-karta bilan imzolaydi va natijani E-IMZO markaziy serveriga beradi.
- **e-imzo-server** — server tomonidagi komponent (masalan `host:8428` da ishlaydigan JAR). PKCS#7'larni **tekshiradi**, timestamp qo'shadi, va (mobil moduli yoqilgan bo'lsa) mobil oqimini boshqaradi. **Og'ir GOST kriptografiyasini sizning o'rningizga qiladigan yagona qism shu.**
- **SiteID** — E-IMZO bergan, merchant-ID kabi identifikator. Mobil oqim uchun SiteID'ga **callback URL** registratsiya qilasiz.
- **PKCS#7** — imzoning "muhrlangan konverti": kim imzolagan, sertifikati, kriptografik imzo. *Attached* (hujjat ichida) yoki *detached* (faqat imzo) bo'lishi mumkin.
- **PINFL (ЖШШИР)** — imzolovchining milliy ID raqami, sertifikatдан olinadi — *aynan kerakli odam* imzolaganini tekshirish uchun ishlating.

**Asosiy model:** E-IMZO'ni "Google bilan kirish" yoki to'lov webhook'i kabi tasavvur qiling. Kriptografiyani o'zingiz yozmaysiz; E-IMZO'ga topshirasiz va natijani tekshirasiz. Ishonch **matematikaga (imzo tekshiruvdan o'tishiga)** asoslanadi, kim chaqirganiga emas.

---

## 2. Hash — O'z DSt 1106 (sizni kunlardan qutqaradigan bo'lim)

Har bir imzo hujjatning **hash**i ustidan bo'ladi, butun fayl ustidan emas. E-IMZO **O'z DSt 1106:2009** ishlatadi. Eng muhim fakt:

> **O'z DSt 1106 == CryptoPro parametrlari bilan GOST R 34.11-94 (S-box "D-A").**
> Bu 256-bitli hash. U **SHA-256 emas**, va GOST "test" parametrlari to'plami ham **emas**.

### 2.1 Java'da (backend) — BouncyCastle, ~5 qator

BouncyCastle'ning **default** `GOST3411Digest`i aynan shu parametrlar to'plami. Maxsus S-box shart emas:

```java
import org.bouncycastle.crypto.digests.GOST3411Digest;
import java.util.HexFormat;

public static String ozDst1106Hex(byte[] input) {
    GOST3411Digest d = new GOST3411Digest();      // default = CryptoPro/"D-A" = O'z DSt 1106
    d.update(input, 0, input.length);
    byte[] out = new byte[d.getDigestSize()];     // 32 bayt
    d.doFinal(out, 0);
    return HexFormat.of().formatHex(out);          // 64 hex belgi
}
```

### 2.2 Har qanday implementatsiyani shu vektorlar bilan tekshiring

GOST implementatsiyasiga u shu **e'lon qilingan CryptoPro GOST R 34.11-94** test vektorlarини qaytarmaguncha ishonmang (O'z DSt 1106 bilan bir xil). Ularni unit-testga qo'ying:

```
""                                            -> 981e5f3ca30c841487830f84fb433e13ac1101569b9c13584ac483234cd656c0
"a"                                           -> e74c52dd282183bf37af0079c9f78055715a103f17e3133ceff1aacf2f403011
"abc"                                         -> b285056dbf18d7392d7677369524dd14747459ed8143997e163b2986f92fd42c
"message digest"                              -> bc6041dd2aa401ebfa6e9886734174febdb4729aa972d60f549ac39b29721ba0
"The quick brown fox jumps over the lazy dog" -> 9004294a361a508c586fe53d1f1b02746765e71b765472786e4770d565830a76
```

Agar `hash("")` sizда `981e5f3c…` bo'lsa — algoritm to'g'ri. Boshqa narsa bo'lsa (masalan GOST *test* to'plami `ce85b99c…` beradi) — u E-IMZO bilan **verify'dan o'tmaydi**.

### 2.3 Flutter/Dart va JavaScript'da (client)

- **Dart/Flutter:** `github.com/jafar260698/E-IMZO-INTEGRATION` → `lib/crypto/` dagi tayyor `gost_hash.dart` (klass `GostHash`, `OzDSt1106Digest`, S-box `"D_A"`). Shu papkani ko'chirib oling.
- **JavaScript (web):** E-IMZO'ning o'z demosidagi `e-imzo-mobile.js` ichidagi `gosthash()`.

### 2.4 Hashni kim hisoblaydi — va bayt-aniqlik tuzog'i

Hashni **client** (web plagini / mobil SDK) ham, sizning **backend**ingiz ham hisoblashi mumkin — hujjat kimда bo'lsa. Ikkalasi ham to'g'ri. Yagona qat'iy qoida:

> **Verify aynan qaysi baytlarни qayta hashlasa, o'sha baytlarни hashlang.** Detached PKCS#7 uchun e-imzo-server keyin siz yuborgan *hujjat baytlarini* qayta hashlaydi. Demak **xom hujjat baytlarини** hashlang, ularning base64 satrini emas. Base64 matnni (dekod qilingan baytlar o'rniga) hashlash — klassik jim xatolik.

Server-markazли dizayn uchun (hujjat backend'да) hashni backend'да hisoblash eng oddiy va payload'ни kichik saqlaydi — butun fayl o'rniga 64-belgili `hashHex` qaytarasiz.

---

## 3. e-imzo-server (tekshirish serveri)

Bu asosiy ishchi qism. Odatda uni o'zingiz ishlatasiz (JAR) va backend'ingizni unga yo'naltirasiz.

**Tekshirish va timestamp endpointlari (doim mavjud):**

| Endpoint | Vazifasi |
|---|---|
| `POST /backend/pkcs7/verify/attached` | *Attached* PKCS#7'ni tekshirish (hujjat ichida) |
| `POST /backend/pkcs7/verify/detached` | *Detached* PKCS#7'ni `{documentBase64, pkcs7}`ga tekshirish — **server O'z DSt 1106'ni o'zi hisoblaydi** va imzoni tekshiradi |
| `POST /backend/timestamp/...` | RFC-3161 timestamp qo'shish |

**Mobil-modul endpointlari (faqat mobil modul yoqilgan + sozlangan bo'lsa):**

| Endpoint | Vazifasi |
|---|---|
| `POST /frontend/mobile/auth` | Mobil **login**ни boshlash; imzolash uchun tasodifiy `challange` qaytaradi |
| `POST /frontend/mobile/sign` | Mobil **sign** sessiyasini ochish; `{status, siteId, documentId}` qaytaradi — **body olmaydi va hash QAYTARMAYDI** (faqat documentId ajratgich) |
| `POST /frontend/mobile/status` | Sessiyani form param `documentId` orqali so'rash (`1`=tayyor, `2`=kutilmoqda) |
| `POST /frontend/mobile/upload` | E-IMZO PKCS#7'ni post qiladigan joy (callback e-imzo-server'ga registratsiya qilingan bo'lsa) |
| `GET /backend/mobile/authenticate/{documentId}` | Tekshirilgan **auth** natijasini olish |
| `POST /backend/mobile/verify` | Tekshirilgan **sign** natijasini olish |

**Bizni vaqt yo'qotgan ikki fakt:**

1. **Hash endpointi yo'q.** `/frontend/mobile/sign` hujjat *olmaydi* va hash *qaytarmaydi* — u faqat qisqa `documentId` chiqaradi. O'z DSt 1106 hashini siz (client yoki backend) o'zingiz hisoblashingiz kerak (§2).
2. **Mobil modul umuman bo'lmasligi mumkin.** Yangi e-imzo-server build'i ko'pincha faqat verify endpointlarини beradi; mobil modul yoqilib, `mobile.siteId` sozlanmaguncha `/frontend/mobile/*` `404` qaytaradi. Agar mobil sessiyalaringiz hech qachon tugamasa — avval shu endpointlarni zondlang.

---

## 4. Web integratsiya (brauzer)

Desktop/web oqim **E-IMZO brauzer plagini**dan foydalanadi — deep-link ham, server callback ham yo'q.

**Oqim:**

1. `e-imzoapi.js`ni yuklang; u lokal plagin bilan `CAPIWS` (`wss://127.0.0.1:64322`) orqali gaplashadi.
2. Foydalanuvchi sertifikatlarини ro'yxatlang (`CAPIWS.apikey` → `listAllUserKeys`); foydalanuvchi bittasini tanlaydi.
3. Kalitni yuklang, so'ng PKCS#7'ni **brauzer ichida** yasang — plagin hujjatni hashlaydi (O'z DSt 1106) va imzolaydi. Siz base64 PKCS#7 (attached yoki detached) olasiz.
4. PKCS#7'ni (detached bo'lsa hujjat bilan) **backend'ingizga** `POST` qiling.
5. Backend uni e-imzo-server orqali **tekshiradi** (`/backend/pkcs7/verify/{attached,detached}`), PINFL'ni tekshiradi va imzo yozuvini saqlaydi.

**Muhim nuqtalar:**
- **Plagin** hashni hisoblaydi va imzoni yasaydi — sizning JS'ingiz GOST'ga tegmaydi.
- Backend'ingizning vazifasi mobil oqimning verify qadami bilan bir xil (§6) — o'sha `verify/detached` chaqiruvi. Shu kodni umumiy qiling.
- Namuna: E-IMZO'ning `e-imzoapi.js` / `e-imzo-client.js` demosi va `qo0p/e-imzo-doc`.

---

## 5. Mobil integratsiya — E-IMZO ID-Card Mobile (deep-link + callback)

Telefonда sizning kodingiz boshqaradigan lokal plagin yo'q, shuning uchun mobil oqim **server-markazли**: backend'ingiz hashni tayyorlaydi, telefon E-IMZO ilovasiga deep-link orqali kiradi va uni imzolaydi, so'ng **E-IMZO markaziy serveri tayyor PKCS#7'ni sizning callback URL'ingizga qaytaradi**.

```
Mobil ilova         Sizning backend      E-IMZO ilova (tel)     E-IMZO markaz
    | session ochish     |                     |                     |
    |------------------->| documentId ajratadi |                     |
    |                    |  + O'z DSt 1106 hash |                     |
    |<-- siteId,         |                     |                     |
    |    documentId,     |                     |                     |
    |    hashHex --------|                     |                     |
    | deep-link quradi   |                     |                     |
    |------------------- eimzo://sign?qc=... ->|                     |
    |                    |            [PIN + NFC, hashni imzolaydi]   |
    |                    |                     |-- PKCS#7 ---------->|
    |                    |<===== callback: POST /sizning/upload ======|  (server-to-server)
    |                    |  verify + PINFL -> saqlash                 |
    |-- status so'rash ->|                     |                     |
    |<-- SIGNED ---------|                     |                     |
```

### 5.1 Qadam-baqadam

1. **Ilova → backend:** "shu hujjat uchun imzo sessiyasi och" (foydalanuvchi tokeni bilan).
2. **Backend:** huquqlarни tekshiradi; e-imzo-server `POST /frontend/mobile/sign` orqali `documentId` ajratadi; hujjat baytlarining **O'z DSt 1106 hash**ini hisoblaydi; `{siteId, documentId, hashHex}` qaytaradi.
3. **Ilova:** deep-link quradi va ochadi (5.2'ga qarang).
4. **Foydalanuvchi:** PIN + ID-kartani tegizadi (NFC). E-IMZO ilovasi PKCS#7 yasaydi.
5. **E-IMZO markaz → backend:** PKCS#7'ni SiteID'ingizga registratsiya qilingan callback URL'ga post qiladi (5.3'ga qarang). Server-to-server, **token yo'q**.
6. **Backend:** PKCS#7'ni (e-imzo-server orqali) o'sha hujjat baytlariga tekshiradi + PINFL'ni tekshiradi → saqlaydi (yoki FAILED belgilaydi).
7. **Ilova:** status endpointini `SIGNED` / `FAILED` bo'lguncha so'raydi.

### 5.2 Deep-link formati

```
eimzo://sign?qc=<siteId><documentId><hashHex><crc32>
```

`qc` — bitta birlashtirilgan satr; oxiriga `crc32 = CRC32(siteId + documentId + hashHex)` qo'shiladi. Dart'da (referencedан), backend allaqachon `hashHex` bergan deb faraz qilsak:

```dart
String code = siteId + documentId + hashHex;      // hashHex — backenddan
code += Crc32.calcHex(code);                        // faqat CRC32 client-side (crc32.dart)
final deepLink = 'eimzo://sign?qc=$code';
await launchUrl(Uri.parse(deepLink), mode: LaunchMode.externalApplication);
```

Agar backend `hashHex` yubormasa (client-hisoblaydi dizayni), avval hujjat baytlaridan `gost_hash.dart` bilan hisoblang (§2.3) — **dekod qilingan** baytlar ustidan, hech qachon base64 satr ustidan emas.

### 5.3 Callback kontrakti (mobil oqimning katta tuzog'i)

E-IMZO markaz PKCS#7'ni SiteID'ingizga registratsiya qilingan URL'ga **`application/x-www-form-urlencoded`** sifatida post qiladi. Maydon nomlari — **haqiqiy prod callback'дан tasdiqlangan**:

| Maydon | Ma'nosi |
|---|---|
| `document_id` | siz ajratgan sessiya id'si |
| **`pkcs7_b64`** | detached PKCS#7, base64 — **`pkcs7` nomli maydon EMAS** |
| `serial_number` | imzolovchi sertifikat seriyasi |
| `x_real_ip` | E-IMZO ko'rgan imzolovchi IP'si |

> **Bu bizni qattiq qiynadi.** Biz `pkcs7` nomli param o'qidik va bo'sh oldik → har bir callback'да `400` → sessiyalar mangu "PENDING"да qoldi, hash ham, URL registratsiyasi ham to'g'ri bo'lsa-da. **`pkcs7_b64` o'qing.** Shubha bo'lsa, birinchi haqiqiy callback'да butun payload'ни log qiling (`request.getParameterMap().keySet()`) va haqiqiy nomlarни o'qing.

### 5.4 Ilova PKCS#7'ga hech qachon tegmaydi

Mobil ilova faqat: sessiya ochadi, deep-link quradi, statusни so'raydi. PKCS#7 E-IMZO-ilova → E-IMZO-markaz → backend yo'lidан boradi. Ilova uni hech qachon yuklamaydi.

---

## 6. Backend integratsiya

### 6.1 Siz ochadigan endpointlar

| Metod & yo'l (namuna) | Auth | Vazifasi |
|---|---|---|
| `POST /api/v1/mobile/documents/{id}/sign/session` | foydalanuvchi tokeni | sessiya ochish → `{siteId, documentId, hashHex, expiresAt}` |
| `GET  /api/v1/mobile/sign/session/{documentId}` | foydalanuvchi tokeni | so'rash → `{status, signedAt, errorCode}` |
| `POST /api/v1/mobile/sign/upload` | **permitAll** | E-IMZO callback'i (§5.3) |

Sessiya holati **Redis**'ga juda mos tushadi (status + TTL + bir martalik lock — takroriy callback ikki marta ishlamasin). Sessiya uchun yangi DB jadval shart emas; doimiy imzoni odatdagi verify+saqlash yo'lingiz yozadi.

### 6.2 PKCS#7'ni tekshiring (web + mobil uchun umumiy)

Hujjat baytlari + PKCS#7'ni e-imzo-server'ga yuboring; u O'z DSt 1106'ni hisoblaydi va imzoni tekshiradi. So'ng imzolovchini tekshiring:

```
resp = eimzoServer.verifyDetached(base64(documentBytes), pkcs7, realIp, host);
if (!resp.signer.verified())            reject("imzo yaroqsiz");
if (!resp.signer.certificateValid())    reject("sertifikat yaroqsiz");
if (!pinflOf(resp.signer).equals(expectedPinfl)) reject("noto'g'ri imzolovchi");
persistSignature(resp);
```

**Detached** bloklar uchun server kontent↔imzo mosligini o'zi tekshiradi (mos kelmasa `verified() == false` bo'ladi). Agar hujjatning SHA-256'ini saqlasangiz — u faqat evidence uchun barmoq izi, imzolash hashi **emas**.

### 6.3 Callback handler (to'g'ri maydonlarni o'qing)

```java
@PostMapping("/upload")   // permitAll
public StatusResponse upload(HttpServletRequest req) {
    String documentId = firstNonBlank(req.getParameter("document_id"), req.getParameter("documentId"));
    String pkcs7      = firstNonBlank(req.getParameter("pkcs7_b64"), req.getParameter("pkcs7")); // pkcs7_b64!
    if (documentId == null || pkcs7 == null || pkcs7.isBlank())
        throw badRequest("document_id/pkcs7 yo'q");
    // Evidence IP: SERVER aniqlagan IP'ni ishlating, foydalanuvchi yuborgan x_real_ip form param'ni emas —
    // bu endpoint permitAll, shuning uchun client maydoni soxtalashtiriladi.
    return service.completeUpload(documentId, pkcs7, ClientIpResolver.resolve(req));
}
```

### 6.4 Xavfsizlik modeli (ochiq callback nega xavfsiz)

Callback `permitAll`, chunki **E-IMZO token olib yurmaydi**. Xavfsizlik IP allow-list'ga tayanmaydi. U quyidagilarga asoslanadi:

1. **Taxmin qilib bo'lmaydigan sessiya id'si** — `documentId` bitta imzoni chegaralaydi.
2. **Kriptografik tekshiruv** — soxta/o'zgartirilgan PKCS#7 `verifyDetached`'дан o'tmaydi va **hech narsa saqlanmaydi**.
3. **PINFL darvozasi** — sertifikatning milliy ID'si kutilgan imzolovchiga mos kelishi shart.

Xulosa: **foydalanuvchi yuborgan maydonlarга** (masalan `x_real_ip`) evidence sifatida saqlanadigan hech narsada **ishonmang** — `permitAll` endpointga har kim POST qila oladi. Server aniqlagan qiymatlarни ishlating.

---

## 7. SiteID registratsiyasi

- E-IMZO'дан **SiteID** oling (shartnoma asosida).
- O'sha SiteID uchun E-IMZO'ga **callback (UPLOAD) URL**ingizni registratsiya qiling — E-IMZO markaz PKCS#7'ni shu yerga post qiladi. Bu E-IMZO tomonidagi *tashqi* registratsiya, sizning kodingizда emas.
- e-imzo-server'ingizning mobil modulини o'sha `mobile.siteId` bilan sozlang va yoqing.
- Callback URL E-IMZO'дан yetib boradigan (ochiq HTTPS) va siz registratsiya qilgan bilan aynan bir xil bo'lishi kerak.

Agar E-IMZO callback'i umuman kelmasa (log'да `/upload` urilmalari yo'q) — SiteID→URL registratsiyasi noto'g'ri yoki yo'q. Agar kelsa-yu 400 bersa — maydon nomi masalasi (§5.3).

---

## 8. Xatolarni topish yo'riqnomasi — belgi → sabab

| Belgi | Ehtimoliy sabab | Yechim |
|---|---|---|
| Imzo **hech qachon verify'dan o'tmaydi** | Noto'g'ri hash algoritmi (SHA-256 yoki GOST *test* to'plami) | O'z DSt 1106 = CryptoPro GOST R 34.11-94 ishlating; §2.2 vektorlari bilan tekshiring |
| Faqat ba'zi hujjatlar verify bo'lmaydi | Dekod baytlar o'rniga **base64 satr** hashlangan | Xom hujjat baytlarини hashlang |
| Mobil sessiya **mangu PENDING**, log'да `/upload` yo'q | Callback URL registratsiya qilinmagan / noto'g'ri SiteID / mobil modul o'chiq | Registratsiyani tekshiring (§7); `/frontend/mobile/*`ни 404 uchun zondlang |
| `/upload` uriladi lekin **400** | `pkcs7_b64` o'rniga `pkcs7` o'qilyapti | `pkcs7_b64` o'qing (§5.3); `getParameterMap().keySet()`ni log qiling |
| `/upload` 200 lekin verify **FAILED** | form-urlencoded'да base64 `+` bo'sh joyga aylanган; yoki noto'g'ri hujjat baytlari | `pkcs7.replace(' ', '+')`; aynan bir xil baytlarни hashlang/verify qiling |
| `documentId` g'alati (uzun UUID) | Siz uni o'zingiz yasayapsiz (stub/placeholder), `/frontend/mobile/sign` chaqirmasdan | Haqiqiy ajratgichni ishlating; haqiqiy id qisqa (masalan `8572EE3E`) |
| Web plagin topilmadi | E-IMZO plagini o'rnatilmagan / `CAPIWS` bloklangan | Foydalanuvchini E-IMZO o'rnatishga yo'naltiring; `wss://127.0.0.1:64322` handshake'ni tekshiring |

**Oltin diagnostika harakati:** birinchi haqiqiy callback'да **butun payload shaklini log qiling** (content type + barcha param kalitlari + uzunliklar). E-IMZO'ning haqiqiy maydon nomlari — haqiqat manbai; hujjatlar realdan orqada qoladi.

---

## 9. Namuna implementatsiyalar va havolalar

- **`qo0p/e-imzo-doc`** — integratsiya spesifikatsiyasi (ketma-ketlik diagrammalari, endpoint ro'yxati).
- **`jafar260698/E-IMZO-INTEGRATION`** — Flutter/Dart client. `lib/crypto/`ni ko'chiring (`gost_hash.dart`, `ozdst/ozdst_1106_digest.dart`, `gost/gost_28147_engine.dart`, `hex.dart`, `crc32.dart`) — ishlaydigan, to'g'ri O'z DSt 1106 + deep-link quruvchi.
- **`jafar260698/E-IMZO-ANDROID`**, **`.../DeepLink-E-IMZO`** — Android/Java deep-link namunalari.
- **`e-imzo-mobile.js`** (E-IMZO demo) — web deep-link + `gosthash`.
- **`e-imzoapi.js` / `e-imzo-client.js`** — web oqim uchun brauzer-plagin (`CAPIWS`) API'si.
- **BouncyCastle** `org.bouncycastle:bcprov-jdk18on` — backend uchun `GOST3411Digest` (default = O'z DSt 1106).

---

## 10. Checklist'lar

**Web**
- [ ] E-IMZO plagini `CAPIWS` orqali aniqlandi
- [ ] Kalitlar ro'yxati → foydalanuvchi tanlaydi → PKCS#7 yaraladi (plagin hashlaydi + imzolaydi)
- [ ] PKCS#7 (detached bo'lsa hujjat bilan) backend'ga POST qilinadi
- [ ] Backend `verify/detached` + PINFL darvozasi + saqlash

**Mobil**
- [ ] Backend sessiya ochadi: haqiqiy `documentId` (qisqa) + O'z DSt 1106 `hashHex`
- [ ] Ilova `eimzo://sign?qc=<siteId><documentId><hashHex><crc32>` quradi; ochadi
- [ ] SiteID callback URL E-IMZO'ga registratsiya qilingan (ochiq HTTPS)
- [ ] Callback handler `pkcs7_b64` + `document_id` o'qiydi (form-urlencoded)
- [ ] `verify/detached` + PINFL darvozasi + saqlash; ilova SIGNED'gacha so'raydi

**Backend**
- [ ] O'z DSt 1106 = BouncyCastle default `GOST3411Digest`; §2.2 vektorlari bilan unit-test qilingan
- [ ] Verify yo'li web + mobil uchun umumiy
- [ ] Callback `permitAll`; xavfsizlik = kripto verify + PINFL + taxmin qilib bo'lmaydigan id
- [ ] Evidence/audit uchun foydalanuvchi yuborgan hech qanday maydonga ishonilmaydi

---

*Haqiqiy prod E-IMZO integratsiyasidan (web + mobil + backend) qurildi va jonli ishlab turibdi. Agar bu yerdagi maydon nomi yoki endpoint sizning jonli E-IMZO'ingizga zid kelsa — jonli callback'ga ishoning va bu qo'llanmani yangilang: E-IMZO'ning haqiqiy xatti-harakati haqiqat manbaidir.*
