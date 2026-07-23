# E-IMZO Integratsiya Qo'llanmasi — Web, Mobile va Backend

> Bu qo'llanma biz o'zimiz orzu qilgan hujjat. E-IMZO'ning rasmiy hujjatlari juda qisqa va bir necha [GitHub](https://github.com/qo0p/e-imzo-doc) repolari, JS demo va Java namunalari orasida tarqoq. Biz E-IMZO'ni real loyihaga ulaganda kunlab qiynaldik. Shu sababli hamma kerakli narsani — tushunchalar, aniq API kontraktlari, ishlaydigan kod va bizni eng ko'p aldab yurgan xatoliklar — bir joyga to'pladik. Maqsad: sizning integratsiyangiz bizникидан ancha oson o'tsin.
>
> Kimlar uchun: **hamma daraja uchun**. Agar siz junior bo'lsangiz — hech narsani oldindan bilishingiz shart emas, hammasi tushuntirilgan. Agar senior bo'lsangiz — to'g'ridan-to'g'ri kerakli bo'limga o'ting. GOST yoki kriptografiya bo'yicha bilim talab qilinmaydi.

**Mundarija**

1. [Eng avval o'qing — E-IMZO qanday ishlaydi](#1-eng-avval-oqing--e-imzo-qanday-ishlaydi)
2. [Ishtirokchilar (qismlar)](#2-ishtirokchilar-qismlar)
3. [Hash — O'z DSt 1106 (eng muhim bo'lim)](#3-hash--oz-dst-1106-eng-muhim-bolim)
4. [e-imzo-server (verify serveri)](#4-e-imzo-server-verify-serveri)
5. [Web integratsiya (browser)](#5-web-integratsiya-browser)
6. [Mobile integratsiya (deep-link + callback)](#6-mobile-integratsiya-deep-link--callback)
7. [Backend integratsiya](#7-backend-integratsiya)
8. [SiteID va callback URL registratsiyasi](#8-siteid-va-callback-url-registratsiyasi)
9. [Xatolarni topish jadvali](#9-xatolarni-topish-jadvali)
10. [Manba kodlar va havolalar](#10-manba-kodlar-va-havolalar)

---

## 1. Eng avval o'qing — E-IMZO qanday ishlaydi

**E-IMZO nima?** Bu O'zbekistonning milliy elektron imzo tizimi (Soliq qo'mitasi tomonidan). Foydalanuvchi o'zining **ID-kartasi** yordamida hujjatni imzolaydi, va siz natijada tekshirsa bo'ladigan raqamli imzo — **PKCS#7** — olasiz.

Bu yerda tushunib olish kerak bo'lgan eng asosiy g'oya bor, va u ko'pchilikning boshini qotiradi:

> **Imzoning o'zini siz yaratmaysiz.** PKCS#7 imzoni har doim E-IMZO yaratadi — yo browser ichidagi maxsus plugin orqali, yo davlatning mobile ilovasi ichida. Sizning ishingiz atigi ikki narsa: (1) E-IMZO'ga imzolanadigan hujjatning **hash**ini berish, va (2) u qaytargan PKCS#7'ni **verify qilish** (tekshirish).

Yaxshi taqqoslash: E-IMZO'ni "Google orqali kirish" yoki to'lov tizimining webhook'i kabi tasavvur qiling. Siz kriptografiyani o'zingiz yozmaysiz — ishni E-IMZO'ga topshirasiz, keyin natijaning haqiqiyligini tekshirasiz. Xavfsizlik "kim so'rov yubordi" degа emas, **"imzo matematik jihatdan to'g'rimi"** degа asoslanadi.

E-IMZO'ni **uch xil usulда** ulash mumkin. Har biri imzoni boshqacha yo'l bilan yaratadi:

| Usul | Imzoni kim yaratadi | Imzo sizga qanday yetib keladi |
|---|---|---|
| **Web** (kompyuter browser'i) | E-IMZO browser plugin'i (kompyuterda lokal ishlaydi) | Sizning JavaScript kodingiz PKCS#7'ni backend'ingizga `POST` qiladi |
| **Mobile** (telefon) | E-IMZO ID-Card Mobile ilovasi | E-IMZO'ning markaziy serveri PKCS#7'ni sizning **callback URL**ingizga `POST` qiladi |
| **Backend** (server) | — (bu yerda siz faqat verify qilasiz) | Siz `{hujjat, pkcs7}`ni **e-imzo-server**ga yuborib, tekshirtirasiz |

Amalda ko'pincha ikkalasi kerak bo'ladi: **web yoki mobile** imzoni oladi, **backend** esa uni verify qilib, bazaga saqlaydi.

> ⚠️ **Hammadan ko'p qiynaydigan narsa — hash algoritmi.** E-IMZO SHA-256 emas, **O'z DSt 1106** (bu — CryptoPro parametrli GOST R 34.11-94) ishlatadi. Agar algoritmni adashtirsangiz, hech qanday xato chiqmaydi — imzo shunchaki **jimgina verify'дан o'tmaydi**, va siz sababini soatlab qidirasiz. Shuning uchun [3-bo'lim](#3-hash--oz-dst-1106-eng-muhim-bolim) — eng muhimi.

---

## 2. Ishtirokchilar (qismlar)

Integratsiyaга kirishishдан oldin, kim nima ish qilishini bilib oling:

- **ID-karta** — foydalanuvchining jismoniy smart-kartasi. Uning ichida sertifikat va maxfiy kalit turadi. Imzolash aynan **karta ichida** sodir bo'ladi (PIN kod + NFC yoki karta o'quvchi orqali). Maxfiy kalit kartadan hech qachon chiqmaydi.
- **E-IMZO browser plugin'i** — foydalanuvchi kompyuteriga o'rnatiladigan kichik dastur. U `wss://127.0.0.1:64322` manzilida WebSocket API (`CAPIWS`) ochadi, va sizning web sahifangiz uni [`e-imzoapi.js`](https://github.com/qo0p/e-imzo-doc) kutubxonasi orqali boshqaradi. Imzoni aynan shu plugin yaratadi.
- **E-IMZO ID-Card Mobile ilovasi** — davlatning rasmiy mobile ilovasi. U sizning ilovangizdan `eimzo://` deep-link orqali ochiladi, hash'ni ID-karta bilan imzolaydi, va natijani E-IMZO'ning markaziy serveriga topshiradi.
- **e-imzo-server** — server tomonda ishlaydigan komponent (odatda `host:8428` da ishlaydigan JAR fayl). Uning ishi: PKCS#7'larni **verify qilish**, timestamp qo'shish, va (mobile moduli yoqilgan bo'lsa) mobile jarayonini boshqarish. **Og'ir GOST kriptografiyasini sizning o'rningizga qiladigan yagona qism — shu.** Siz odatda uni o'zingiz server'ga o'rnatib qo'yasiz va backend'ingizni unga yo'naltirasiz.
- **SiteID** — E-IMZO sizga beradigan identifikator (do'kon uchun "merchant ID"ga o'xshaydi). Mobile jarayonда siz SiteID uchun bitta **callback URL** registratsiya qilasiz.
- **PKCS#7** — imzoning "muhrlangan konverti". Uning ichida: kim imzolagan, uning sertifikati, va kriptografik imzoning o'zi bor. Ikki turi bor: *attached* (hujjat konvert ichiga solingan) va *detached* (faqat imzo, hujjatсиз).
- **PINFL (JSHSHIR)** — imzolovchining milliy ID raqami. U sertifikat ichidan olinadi. Siz undan **aynan kerakli odam imzolaganini** tekshirish uchun foydalanasiz.

---

## 3. Hash — O'z DSt 1106 (eng muhim bo'lim)

Har qanday imzo butun fayl ustidan emas, uning **hash**i (qisqa, o'zgarmas "barmoq izi") ustidan qo'yiladi. E-IMZO **O'z DSt 1106:2009** algoritmini ishlatadi. Mana eng muhim fakt, uni yaxshilab yodda tuting:

> **O'z DSt 1106 — bu CryptoPro parametrli GOST R 34.11-94 (S-box "D-A") algoritmi.** Natija — 256-bitli hash. Bu **SHA-256 emas**, va GOST'ning "test" parametrlari ham **emas**.

### 3.1 Java'da (backend) — BouncyCastle bilan atigi 5 qator

Eng yaxshi yangilik: [BouncyCastle](https://www.bouncycastle.org/download/bouncy-castle-java/) kutubxonasining **default** `GOST3411Digest`i aynan shu algoritm. Hech qanday maxsus S-box sozlash shart emas:

```java
import org.bouncycastle.crypto.digests.GOST3411Digest;
import java.util.HexFormat;

public static String ozDst1106Hex(byte[] input) {
    GOST3411Digest d = new GOST3411Digest();      // default = CryptoPro/"D-A" = O'z DSt 1106
    d.update(input, 0, input.length);
    byte[] out = new byte[d.getDigestSize()];     // 32 bayt
    d.doFinal(out, 0);
    return HexFormat.of().formatHex(out);          // 64 ta hex belgi
}
```

### 3.2 Har qanday implementatsiyani shu vektorlar bilan tekshiring

Qaysi tilда bo'lmasin, GOST implementatsiyangizga u quyidagi natijalarni bermaguncha ishonmang. Bular — hammaga ochiq, e'lon qilingan **CryptoPro GOST R 34.11-94 test vektorlari** (O'z DSt 1106 bilan bir xil). Ularni unit-test qilib qo'ying:

```
hash("")                                             = 981e5f3ca30c841487830f84fb433e13ac1101569b9c13584ac483234cd656c0
hash("a")                                            = e74c52dd282183bf37af0079c9f78055715a103f17e3133ceff1aacf2f403011
hash("abc")                                          = b285056dbf18d7392d7677369524dd14747459ed8143997e163b2986f92fd42c
hash("message digest")                               = bc6041dd2aa401ebfa6e9886734174febdb4729aa972d60f549ac39b29721ba0
hash("The quick brown fox jumps over the lazy dog")  = 9004294a361a508c586fe53d1f1b02746765e71b765472786e4770d565830a76
```

Agar sizning `hash("")` natijangiz `981e5f3c…` bo'lsa — algoritm to'g'ri. Agar boshqacha bo'lsa (masalan, GOST'ning *test* to'plami `ce85b99c…` beradi), u E-IMZO bilan **hech qachon verify'дан o'tmaydi**.

### 3.3 Flutter/Dart va JavaScript'da (client tomonda)

- **Dart/Flutter:** tayyor va ishlaydigan kod [jafar260698/E-IMZO-INTEGRATION](https://github.com/jafar260698/E-IMZO-INTEGRATION) repositoriyasining [`lib/crypto/`](https://github.com/jafar260698/E-IMZO-INTEGRATION/tree/dev/lib/crypto) papkasida (klass `GostHash`, `OzDSt1106Digest`, S-box `"D_A"`). Shu papkani o'z loyihangizga ko'chirib oling, o'zingiz GOST yozib o'tirmang.
- **JavaScript (web):** E-IMZO'ning o'z demosidagi [`e-imzo-mobile.js`](https://test.e-imzo.uz/demo/eimzoidcard/js/e-imzo-mobile.js) fayli ichidagi `gosthash()` funksiyasi.

### 3.4 Hash'ni kim hisoblaydi — va bir muhim tuzoq

Hash'ni **client** (web plugin yoki mobile SDK) ham, sizning **backend**ingiz ham hisoblashi mumkin — hujjat kimda bo'lsa, o'sha hisoblaydi. Ikkala yo'l ham to'g'ri. Faqat bitta qat'iy qoida bor:

> **verify aynan qaysi baytlarni qayta hashlasa, siz ham aynan o'sha baytlarni hashlashingiz kerak.** Detached PKCS#7 uchun e-imzo-server keyinchalik siz yuborgan *hujjat baytlarini* qayta hashlaydi. Demak, siz ham **xom hujjat baytlarini** (raw bytes) hashlashingiz kerak, ularning base64 satrini emas. Base64 matnni (dekod qilingan baytlar o'rniga) hashlash — juda ko'p uchraydigan, jimgina xatolik.

Agar sizning arxitekturangiz server tomonда bo'lsa (hujjat backend'да tursa), hash'ni backend'да hisoblash eng oson yo'l: shunда ilovaga butun faylni emas, atigi 64 belgili `hashHex` string qaytarasiz — bu trafik uchun ham yengil.

---

## 4. e-imzo-server (verify serveri)

Bu — integratsiyaning ishchi yuragi. Odatda siz uni o'zingiz server'ingizga o'rnatasiz.

**Verify va timestamp uchun endpoint'lar (bular doim mavjud):**

| Endpoint | Nima qiladi |
|---|---|
| `POST /backend/pkcs7/verify/attached` | *Attached* PKCS#7'ni verify qiladi (hujjat konvert ichida) |
| `POST /backend/pkcs7/verify/detached` | *Detached* PKCS#7'ni `{documentBase64, pkcs7}`ga qarab verify qiladi — **server O'z DSt 1106'ni o'zi hisoblab**, imzo shu hash'ga to'g'ri kelishini tekshiradi |
| `POST /backend/timestamp/...` | RFC-3161 timestamp qo'shadi |

**Mobile moduli endpoint'lari (bular faqat mobile moduli yoqilgan va sozlangan bo'lsa ishlaydi):**

| Endpoint | Nima qiladi |
|---|---|
| `POST /frontend/mobile/auth` | Mobile orqali **login**ни boshlaydi; imzolash uchun tasodifiy `challange` qaytaradi |
| `POST /frontend/mobile/sign` | Mobile **imzo** session'ini ochadi; `{status, siteId, documentId}` qaytaradi. **Diqqat: u body qabul qilmaydi va hash qaytarmaydi** — bu shunchaki `documentId` beruvchi (allocator) |
| `POST /frontend/mobile/status` | Session holatini `documentId` (form param) orqali so'raydi (`1`=tayyor, `2`=kutilmoqda) |
| `POST /frontend/mobile/upload` | E-IMZO PKCS#7'ni shu yerga `POST` qiladi (agar callback e-imzo-server'ga registratsiya qilingan bo'lsa) |
| `GET /backend/mobile/authenticate/{documentId}` | Tekshirilgan **login** natijasini oladi |
| `POST /backend/mobile/verify` | Tekshirilgan **imzo** natijasini oladi |

**Bizni ancha vaqt yo'qotgan ikkita fakt:**

1. **Hash uchun alohida endpoint yo'q.** `/frontend/mobile/sign` sizdan hujjat olmaydi va sizga hash qaytarmaydi — u faqat qisqa `documentId` beradi. O'z DSt 1106 hash'ini baribir siz (client yoki backend) o'zingiz hisoblashingiz kerak ([3-bo'lim](#3-hash--oz-dst-1106-eng-muhim-bolim)).
2. **Mobile moduli umuman bo'lmasligi mumkin.** Yangi o'rnatilgan e-imzo-server ko'pincha faqat verify endpoint'larini beradi; mobile moduli yoqilib, `mobile.siteId` sozlanmaguncha `/frontend/mobile/*` yo'llari `404` qaytaradi. Agar mobile session'laringiz hech qachon tugamayotgan bo'lsa — birinchi bo'lib shu endpoint'larni `curl` bilan tekshirib ko'ring.

---

## 5. Web integratsiya (browser)

Kompyuter/browser jarayoni **E-IMZO browser plugin'i** orqali ishlaydi. Bu yerda deep-link ham, server callback ham yo'q.

Jarayon quyidagicha:

1. Web sahifangizga [`e-imzoapi.js`](https://github.com/qo0p/e-imzo-doc)ni yuklaysiz. U kompyuterdagi plugin bilan `CAPIWS` (`wss://127.0.0.1:64322`) orqali bog'lanadi.
2. Foydalanuvchining sertifikatlari ro'yxatini olasiz (`listAllUserKeys`), foydalanuvchi ulardan birini tanlaydi.
3. Tanlangan kalitni yuklaysiz, so'ng PKCS#7'ni **plugin ichida** yaratasiz. Plugin hujjatni O'z DSt 1106 bilan hashlaydi va imzolaydi. Natijada base64 formatдаги PKCS#7 (attached yoki detached) olasiz.
4. Bu PKCS#7'ni (detached bo'lsa, hujjat bilan birga) **backend'ingizga** `POST` qilasiz.
5. Backend uni e-imzo-server orqali **verify qiladi** (`/backend/pkcs7/verify/...`), PINFL'ni tekshiradi, va imzoni bazaga saqlaydi.

Muhim jihatlar:

- Hash'ni ham, imzoni ham **plugin** yaratadi — sizning JavaScript kodingiz GOST'ga umuman tegmaydi.
- Backend'ning bu yerdagi vazifasi mobile jarayonidagi verify qadamiga aynan bir xil ([7-bo'lim](#7-backend-integratsiya)) — o'sha `verify/detached` chaqiruvi. Shu kodni ikkala jarayon uchun umumiy qilib yozing.
- Namuna kodlar: [qo0p/e-imzo-doc](https://github.com/qo0p/e-imzo-doc) va E-IMZO'ning `e-imzoapi.js` / `e-imzo-client.js` demolari.

---

## 6. Mobile integratsiya (deep-link + callback)

Telefonда sizning kodingiz boshqaradigan lokal plugin yo'q. Shuning uchun mobile jarayoni **server tomonда** quriladi: backend hash'ni tayyorlaydi, telefon E-IMZO ilovasiga deep-link orqali o'tib imzolaydi, va **E-IMZO'ning markaziy serveri tayyor PKCS#7'ni sizning callback URL'ingizga qaytaradi**.

```
Mobile ilova        Sizning backend      E-IMZO ilova (tel)     E-IMZO markaz
    | session ochish     |                     |                     |
    |------------------->| documentId oladi    |                     |
    |                    |  + O'z DSt 1106 hash |                     |
    |<-- siteId,         |                     |                     |
    |    documentId,     |                     |                     |
    |    hashHex --------|                     |                     |
    | deep-link quradi   |                     |                     |
    |------------------- eimzo://sign?qc=... ->|                     |
    |                    |            [PIN + NFC, hash imzolanadi]    |
    |                    |                     |-- PKCS#7 ---------->|
    |                    |<===== callback: POST /sizning/upload ======|  (server-to-server)
    |                    |  verify + PINFL -> saqlash                 |
    |-- status so'rash ->|                     |                     |
    |<-- SIGNED ---------|                     |                     |
```

### 6.1 Qadamlar

1. **Ilova → backend:** ilova "shu hujjat uchun imzo session'ini och" deb so'raydi (foydalanuvchining login token'i bilan).
2. **Backend:** avval foydalanuvchining huquqlarini tekshiradi. So'ng e-imzo-server'dan `POST /frontend/mobile/sign` orqali `documentId` oladi. Keyin hujjat baytlarining **O'z DSt 1106 hash**ini hisoblaydi. Va ilovaga `{siteId, documentId, hashHex}` qaytaradi.
3. **Ilova:** shu ma'lumotlardan deep-link quradi va uni ochadi (formati pastда, 6.2).
4. **Foydalanuvchi:** E-IMZO ilovasida PIN kodini kiritadi va ID-kartani NFC bilan tegizadi. E-IMZO ilovasi PKCS#7 imzoni yaratadi.
5. **E-IMZO markaz → backend:** PKCS#7'ni sizning SiteID'ingizga registratsiya qilingan callback URL'ga `POST` qiladi. Bu server-to-server so'rov, **login token'siz** keladi (6.3).
6. **Backend:** kelgan PKCS#7'ni (e-imzo-server orqali) o'sha hujjat baytlariga qarab verify qiladi va PINFL'ni tekshiradi. To'g'ri bo'lsa — imzoni saqlaydi; noto'g'ri bo'lsa — FAILED deb belgilaydi.
7. **Ilova:** bu vaqt ichida status endpoint'ini `SIGNED` yoki `FAILED` chiqmaguncha qayta-qayta so'rab turadi (polling).

### 6.2 Deep-link formati

```
eimzo://sign?qc=<siteId><documentId><hashHex><crc32>
```

`qc` — bu bitta uzun string bo'lib, u `siteId`, `documentId`, `hashHex` va `crc32`ning ketma-ket ulanишидан hosil bo'ladi. Bu yerda `crc32 = CRC32(siteId + documentId + hashHex)` bo'lib, u stringning oxiriga qo'shiladi.

Backend allaqachon `hashHex` bergan bo'lsa, Dart'da (reference kodдан):

```dart
String code = siteId + documentId + hashHex;      // hashHex — backend'dan keladi
code += Crc32.calcHex(code);                        // faqat CRC32 client tomonда (crc32.dart)
final deepLink = 'eimzo://sign?qc=$code';
await launchUrl(Uri.parse(deepLink), mode: LaunchMode.externalApplication);
```

Agar backend `hashHex` yubormasa (client hisoblaydigan dizayn), avval hujjat baytlaridan [`gost_hash.dart`](https://github.com/jafar260698/E-IMZO-INTEGRATION/tree/dev/lib/crypto) bilan hash'ni hisoblang (3.3) — yana bir bor eslatamiz: **dekod qilingan xom baytlar** ustidan, hech qachon base64 string ustidan emas.

### 6.3 Callback kontrakti (mobile jarayonining eng katta tuzog'i)

E-IMZO markaz PKCS#7'ni SiteID'ingizga registratsiya qilingan URL'ga **`application/x-www-form-urlencoded`** formatда `POST` qiladi. Maydon (field) nomlari quyidagicha — bularni biz **haqiqiy production callback'дан** ko'rib, tasdiqladik:

| Field nomi | Ma'nosi |
|---|---|
| `document_id` | siz bergan session id'si |
| **`pkcs7_b64`** | detached PKCS#7 imzo, base64 formatда — **e'tibor bering, `pkcs7` emas!** |
| `serial_number` | imzolovchi sertifikatining seriya raqami |
| `x_real_ip` | E-IMZO ko'rgan imzolovchining IP manzili |

> **Aynan shu bizni juda qattiq qiynadi.** Biz `pkcs7` degan field'ni o'qidik, u bo'sh chiqdi, va har bir callback `400` bilan rad etildi. Natijada session'lar hash ham, URL registratsiyasi ham to'g'ri bo'lsa-da, mangu "PENDING"да qolib ketaverdi. **Yechim: `pkcs7_b64` field'ini o'qing.** Umuman, birinchi haqiqiy callback kelganда butun `POST` body'sini log qiling (masalan, `request.getParameterMap().keySet()`) va E-IMZO aslida qanday nomlar yuborayotganini o'z ko'zingiz bilan ko'ring.

### 6.4 Mobile ilova PKCS#7'ga umuman tegmaydi

Ilovaning ishi faqat uchta: session ochish, deep-link qurib ochish, va status'ni polling qilish. PKCS#7 imzo E-IMZO-ilova → E-IMZO-markaz → sizning backend yo'li bilan boradi. Ilova uni hech qachon o'zi yuklamaydi. Bu — dizaynning muhim jihati.

---

## 7. Backend integratsiya

### 7.1 Siz ochadigan endpoint'lar

| Metod va yo'l (namuna) | Auth | Nima qiladi |
|---|---|---|
| `POST /api/v1/mobile/documents/{id}/sign/session` | foydalanuvchi token'i | session ochadi → `{siteId, documentId, hashHex, expiresAt}` |
| `GET  /api/v1/mobile/sign/session/{documentId}` | foydalanuvchi token'i | status so'raydi → `{status, signedAt, errorCode}` |
| `POST /api/v1/mobile/sign/upload` | **permitAll** (ochiq) | E-IMZO callback'i (6.3) |

Session holatini saqlash uchun **Redis** juda qulay: status + TTL (muddat) + bitta bir martalik lock (takroriy callback ikki marta ishlab ketmasligi uchun). Session uchun alohida bazа jadvali kerak emas; doimiy imzoni sizning oddiy verify+saqlash kodingiz yozadi.

### 7.2 PKCS#7'ni verify qilish (web va mobile uchun umumiy)

Hujjat baytlari va PKCS#7'ni e-imzo-server'ga yuboring — u O'z DSt 1106'ni hisoblab, imzoni tekshiradi. So'ng imzolovchini tekshiring:

```
resp = eimzoServer.verifyDetached(base64(documentBytes), pkcs7, realIp, host);
if (!resp.signer.verified())            reject("imzo yaroqsiz");
if (!resp.signer.certificateValid())    reject("sertifikat yaroqsiz");
if (!pinflOf(resp.signer).equals(expectedPinfl)) reject("noto'g'ri imzolovchi");
persistSignature(resp);
```

*Detached* imzolar uchun server hujjat va imzo bir-biriga mos kelishini o'zi tekshiradi (mos kelmasa, `verified()` `false` bo'ladi). Agar siz hujjatning SHA-256'ini saqlab qo'ysangiz — u faqat log/evidence uchun barmoq izi, imzolash hash'i **emas**. Adashtirmang.

### 7.3 Callback handler (to'g'ri field'larni o'qing)

```java
@PostMapping("/upload")   // permitAll
public StatusResponse upload(HttpServletRequest req) {
    String documentId = firstNonBlank(req.getParameter("document_id"), req.getParameter("documentId"));
    String pkcs7      = firstNonBlank(req.getParameter("pkcs7_b64"), req.getParameter("pkcs7")); // pkcs7_b64!
    if (documentId == null || pkcs7 == null || pkcs7.isBlank())
        throw badRequest("document_id/pkcs7 yo'q");
    // Evidence uchun IP'ni SERVER o'zi aniqlaydi, foydalanuvchi yuborgan x_real_ip field'idan olmaydi:
    // bu endpoint permitAll (ochiq), shuning uchun client yuborgan field'ni soxtalashtirish oson.
    return service.completeUpload(documentId, pkcs7, ClientIpResolver.resolve(req));
}
```

### 7.4 Xavfsizlik modeli (ochiq callback nega xavfsiz)

Callback `permitAll` (hammaga ochiq), chunki **E-IMZO login token olib yurmaydi**. Xavfsizlik IP allow-list'ga ham tayanmaydi. U quyidagi uch narsaga asoslanadi:

1. **Taxmin qilib bo'lmaydigan session id** — `documentId` tasodifiy, va u faqat bitta imzoga tegishli.
2. **Kriptografik verify** — soxta yoki o'zgartirilgan PKCS#7 `verifyDetached`'дан o'tmaydi, va natijada **hech narsa saqlanmaydi**.
3. **PINFL tekshiruvi** — sertifikatдаги milliy ID kutilgan imzolovchiga mos kelishi shart.

Muhim xulosa: **client yuborgan biror field'ga** (masalan, `x_real_ip`) evidence sifatida ishonmang. `permitAll` endpoint'ga har kim `POST` qila oladi, demak client yuborgan har qanday qiymatni soxtalashtirish mumkin. Har doim server o'zi aniqlagan qiymatlarni ishlating. (Biz aynan shu xatoni bir marta qildik, security review tuzatib berdi.)

---

## 8. SiteID va callback URL registratsiyasi

- E-IMZO'дан **SiteID** oling (shartnoma asosida beriladi).
- O'sha SiteID uchun E-IMZO'ga o'zingizning **callback (UPLOAD) URL**ingizni registratsiya qildiring — E-IMZO markaz PKCS#7'ni aynan shu manzilga yuboradi. Bu registratsiya E-IMZO tomonда qilinadi, sizning kodingizда emas.
- e-imzo-server'ingizning mobile modulini o'sha `mobile.siteId` bilan sozlang va yoqing.
- Callback URL E-IMZO'дан yetib boradigan ochiq HTTPS manzil bo'lishi, va siz registratsiya qilgan manzil bilan **aynan bir xil** bo'lishi kerak.

Diagnostika uchun oddiy qoida: agar E-IMZO callback'i log'да umuman ko'rinmasa (`/upload` ga hech qanday so'rov kelmasa) — demak SiteID→URL registratsiyasi noto'g'ri yoki yo'q. Agar so'rov kelsa-yu `400` qaytsa — demak field nomida muammo (6.3).

---

## 9. Xatolarni topish jadvali

| Belgi (symptom) | Ehtimoliy sabab | Yechim |
|---|---|---|
| Imzo **hech qachon verify'дан o'tmaydi** | Noto'g'ri hash algoritmi (SHA-256, yoki GOST'ning *test* to'plami) | O'z DSt 1106 = CryptoPro GOST R 34.11-94 ishlating; [3.2](#32-har-qanday-implementatsiyani-shu-vektorlar-bilan-tekshiring) vektorlari bilan tekshiring |
| Faqat ba'zi hujjatlar verify bo'lmaydi | Xom baytlar o'rniga **base64 string** hashlangan | Hujjatning xom baytlarini hashlang |
| Mobile session **mangu PENDING**, log'да `/upload` yo'q | Callback URL registratsiya qilinmagan / noto'g'ri SiteID / mobile moduli o'chiq | Registratsiyani tekshiring ([8-bo'lim](#8-siteid-va-callback-url-registratsiyasi)); `/frontend/mobile/*` ni `404` uchun `curl` qiling |
| `/upload` ga so'rov keladi, lekin **400** qaytadi | `pkcs7_b64` o'rniga `pkcs7` field o'qilyapti | `pkcs7_b64` o'qing (6.3); `getParameterMap().keySet()` ni log qiling |
| `/upload` `200` qaytaradi, lekin verify **FAILED** | `application/x-www-form-urlencoded`да base64 ичидаги `+` belgisi bo'sh joyга aylanган; yoki noto'g'ri hujjat baytlari | `pkcs7.replace(' ', '+')`; imzolash va verify aynan bir xil baytlar ustidan bo'lsin |
| `documentId` g'alati ko'rinadi (uzun UUID) | Siz uni o'zingiz yasayapsiz (stub/placeholder), `/frontend/mobile/sign` ni chaqirmasdan | Haqiqiy allocator'ни ishlating; haqiqiy id qisqa bo'ladi (masalan `A1B2C3D4`) |
| Web'da plugin topilmadi | E-IMZO plugin'i o'rnatilmagan / `CAPIWS` bloklangan | Foydalanuvchini E-IMZO'ni o'rnatishga yo'naltiring; `wss://127.0.0.1:64322` bilan bog'lanishni tekshiring |

**Eng foydali diagnostika usuli:** birinchi haqiqiy callback kelganда butun so'rovni (content type + barcha field nomlari + uzunliklari) log qiling. E-IMZO'ning haqiqiy field nomlari — asosiy haqiqat manbai; hujjatlar realдан ba'zan orqada qoladi.

---

## 10. Manba kodlar va havolalar

- [**qo0p/e-imzo-doc**](https://github.com/qo0p/e-imzo-doc) — integratsiya spetsifikatsiyasi (ketma-ketlik diagrammalari, endpoint'lar ro'yxati). Boshlash uchun eng yaxshi joy.
- [**jafar260698/E-IMZO-INTEGRATION**](https://github.com/jafar260698/E-IMZO-INTEGRATION) — Flutter/Dart uchun client. [`lib/crypto/`](https://github.com/jafar260698/E-IMZO-INTEGRATION/tree/dev/lib/crypto) papkasini ko'chiring: `gost_hash.dart`, `ozdst/ozdst_1106_digest.dart`, `gost/gost_28147_engine.dart`, `hex.dart`, `crc32.dart` — ishlaydigan, to'g'ri O'z DSt 1106 va deep-link quruvchi.
- [**jafar260698/E-IMZO-ANDROID**](https://github.com/jafar260698/E-IMZO-ANDROID) va [**DeepLink-E-IMZO**](https://github.com/jafar260698/DeepLink-E-IMZO) — Android/Java uchun deep-link namunalari.
- [**e-imzo-mobile.js**](https://test.e-imzo.uz/demo/eimzoidcard/js/e-imzo-mobile.js) — E-IMZO'ning web demosi (deep-link + `gosthash`).
- [**BouncyCastle**](https://www.bouncycastle.org/download/bouncy-castle-java/) (`org.bouncycastle:bcprov-jdk18on`) — backend uchun `GOST3411Digest` (default = O'z DSt 1106).
- [**E-IMZO rasmiy sayti**](https://e-imzo.uz) — plugin, hujjatlar, shartnoma masalalari.

---

*Bu qo'llanma haqiqiy, production'да ishlab turgan E-IMZO integratsiyasidan (web + mobile + backend) yozildi. Agar bu yerdagi biror field nomi yoki endpoint sizning jonli E-IMZO'ingizga to'g'ri kelmasa — jonli callback'ga ishoning va bu hujjatni yangilang: E-IMZO'ning real xatti-harakati eng ishonchli manba.*
