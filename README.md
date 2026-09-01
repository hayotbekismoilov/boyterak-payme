# Payme песочница — to'liq test qo'llanmasi

Bu hujjat `test.paycom.uz` песочница'sida BoyTerakPay integratsiyasini
**noldan oxirigacha** o'tkazish uchun. Har bir maydonga nima yozilishi,
har bir testdan nima kutilishi va xato chiqqanda nima qilish kerakligi
shu yerda.

Protokolning o'zi — [payme-integration.md](payme-integration.md).
Bu hujjat esa **amaliy**: qaysi tugma, qaysi input, qaysi son.

> **Holat (2026-09-01):** песочница 14/14 test yashil. Bitta ochiq
> masala qoldi — `ChangePassword` (§9).

---

## 1. Boshlashdan oldin

Uchta shart bajarilishi kerak. Har birini tekshiradigan buyruq berilgan.

### 1.1 `.env` sozlangan

```bash
grep '^PAYME' BackendBoyTerakPay/.env
```

Kutilgan:

```bash
PAYME_ENABLED=true
PAYME_ENV=test
PAYME_MERCHANT_ID=6a95e980d01de7ce0f2a51cb
PAYME_MERCHANT_KEY=                                    # test muhitida BO'SH
PAYME_TEST_KEY='#hbzp6mfRFBGcSVzWM1Bbz?OvuPsmoOsdcB&'  # BITTA TIRNOQ shart
PAYME_MERCHANT_LOGIN=Paycom
PAYME_ACCOUNT_FIELD=payment_id
PAYME_CHECKOUT_LANG=uz
```

> ⚠️ **`PAYME_TEST_KEY` bitta tirnoq ichida bo'lishi SHART.** Kalit `#`
> bilan boshlanadi, `#` esa `.env` da izoh boshlaydi:
>
> | Yozilishi | django-environ o'qigani |
> |---|---|
> | `KEY=#hbzp…` | **bo'sh satr** |
> | `KEY="#hbzp…"` | **1 belgi** (`"`) |
> | `KEY='#hbzp…'` | ✅ to'g'ri |
>
> Kalit bo'sh bo'lsa endpoint yopiq qoladi va **hamma so'rov `-32504`**
> oladi. Sabab ko'rinmaydi — chalg'itadigan xato shu.

Tekshirish:

```bash
cd BackendBoyTerakPay && ./venv/bin/python -c "
import os,django; os.environ.setdefault('DJANGO_SETTINGS_MODULE','config.settings.dev'); django.setup()
from apps.payments.providers.payme import PaymeAdapter
a = PaymeAdapter()
print('yoqilgan  :', a.is_enabled())
print('muhit     :', a.environment)
print('kalit uz. :', len(a.merchant_key), '(kutilgan 36)')
print('xatolar   :', a.configuration_errors() or 'yo\'q')
"
```

### 1.2 Xizmat ishlayapti

```bash
systemctl is-active boyterakpay-api.service   # -> active
```

`.env` o'zgargan bo'lsa **qayta ishga tushirish shart** — gunicorn
sozlamani faqat startda o'qiydi:

```bash
systemctl restart boyterakpay-api.service
```

### 1.3 Endpoint tashqaridan javob beryapti

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST \
  https://boyterak.myweb.uz/api/v1/payments/payme/ \
  -H "Content-Type: application/json" -d '{}'
```

`200` bo'lishi kerak.

---

## 2. Kabinet sozlamalari

Payme Business → kassa → sozlamalar.

| Maydon | Qiymat |
|---|---|
| G'azna vazifasi | **Elektron to'lovlarni billing bilan qabul qilish** |
| Endpoint URL | `https://boyterak.myweb.uz/api/v1/payments/payme/` |
| Rekvizit nomi | `payment_id` |
| Rekvizit turi | matn (string) |
| Hisob turi | bir martalik (одноразовый) |
| Return URL | `https://boyterak.myweb.uz/payment/result` |

### ⚠️ Endpoint URL oxirida `/` BO'LISHI SHART

```
…/api/v1/payments/payme/   ->  HTTP 200  ✅
…/api/v1/payments/payme    ->  HTTP 301  ❌
```

Payme 200 dan farqli **har qanday** statusni `-32400` deb qabul qiladi.
`/` tushib qolsa **bitta ham test o'tmaydi**, sabab esa javob tanasida
ko'rinmaydi.

### Rekvizit validatsiyasi

| Maydon | Qiymat |
|---|---|
| Doimiy ifoda | `^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$` |
| Raqamlar / Harflar / Belgilar | **belgilamang** — regex yetarli |
| Opsiyalar | bo'sh (dropdown kerak emas) |

Checkboxlar regexga qo'shimcha emas, uning **muqobili**. Belgilansa `-`
chiziqcha rad etilib, haqiqiy UUID o'tmay qolishi mumkin.

---

## 3. Test to'lovlarini yaratish

Песочница bazadagi **haqiqiy PENDING to'lov** ustida ishlaydi.

```bash
cd BackendBoyTerakPay && ./venv/bin/python - <<'PY'
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.dev")
django.setup()
import logging; logging.disable(logging.CRITICAL)
from datetime import timedelta
from django.utils import timezone
from apps.contracts.models import RentalContract, ContractStatus
from apps.payments import services as payment_services
from apps.payments.constants import PaymentMethod, PaymentProvider, PaymentSource

ADAD  = 20          # nechta kerak
SUMMA = 5_000_000   # tiyin = 50 000 so'm

c = (RentalContract.objects.filter(status=ContractStatus.ACTIVE)
     .select_related("tenant", "market_place").order_by("market_place__code").first())
for _ in range(ADAD):
    p = payment_services.create_payment(
        tenant=c.tenant, amount_tiyin=SUMMA, method=PaymentMethod.ONLINE,
        source=PaymentSource.SELLER_ONLINE, market_place=c.market_place,
        note="Payme sandbox testi")
    p.provider = PaymentProvider.PAYME
    # 10 daqiqalik sukut песочница testiga YETMAYDI — 12 soat qo'yamiz.
    p.checkout_expires_at = timezone.now() + timedelta(hours=12)
    p.save(update_fields=["provider", "checkout_expires_at", "updated_at"])
    print(p.public_id)
PY
```

Toza (hali ishlatilmagan) ID larni ko'rish:

```bash
cd BackendBoyTerakPay && ./venv/bin/python - <<'PY'
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.dev")
django.setup()
from apps.payments.models import Payment, ProviderTransaction
for p in Payment.objects.filter(provider="PAYME", status="PENDING").order_by("id"):
    if not ProviderTransaction.objects.filter(
            payment=p, provider="PAYME", provider_state__in=(1, 2)).exists():
        print(p.public_id, p.amount_tiyin)
PY
```

> **Muddat.** `PAYMENT_SESSION_TTL_SECONDS` sukut bo'yicha **600 soniya
> (10 daqiqa)**. Песочница bir necha bo'limni ketma-ket o'tkazadi va
> shu vaqtga sig'maydi — muddati o'tgan to'lov `-31052` beradi. Yuqoridagi
> skript har bir to'lovga 12 soat qo'yadi.

---

## 4. Oltin qoidalar

### 4.1 Bitta `payment_id` = bitta run

Bu eng ko'p vaqt yeydigan tuzoq. To'lov quyidagi hollarda **band** bo'ladi:

| Nima bo'ldi | Keyingi holat |
|---|---|
| `CreateTransaction` muvaffaqiyatli o'tdi | band — yangi tranzaksiya ochilmaydi (`-31053`) |
| `PerformTransaction` o'tdi | to'langan — `-31051` |
| `CancelTransaction` o'tdi | bekor qilingan — `-31051` |

Test to'plamini qayta ishga tushirsangiz **yangi ID oling**. Eskisi bilan
2-qadam (`CreateTransaction`) `-31053` beradi va butun ssenariy qulaydi.

### 4.2 Ikki xil summa maydoni

Песочница formasida ikki xil maydon bor va ular **almashtirilmasligi** kerak:

| Maydon nomi | Qachon chiqadi | Nima yozish |
|---|---|---|
| **Cумма оплаты** | oddiy testlarda | to'lovning **aynan** summasi — `5000000` |
| **Неверная сумма** | «неверная сумма» testida | **farqli** son — `500000` |

Qoida: test nomida «**неверная**» so'zi bo'lsa — summani **ataylab buzing**.

Bu ikkovini chalkashtirish sessiyaning yarmini yegan eng ko'p uchraydigan xato.

### 4.3 Summa — tiyinda

Payme summani **tiyinda** yuboradi, konvertatsiya yo'q.

```
50 000 so'm  =  5000000 tiyin
```

`50000` yozsangiz `-31001` (Неверная сумма) olasiz.

### 4.4 `ChangePassword` ni ishga tushirmang

Sabab §9 da.

---

## 5. Har bir test bo'limi

### 5.1 Неверная авторизация

Parametrsiz. Песочница noto'g'ri kalit bilan ~14 ta so'rov yuboradi.

**Kutilgan:** hammasi `-32504`, HTTP 200.

### 5.2 Неверная сумма

| Maydon | Qiymat |
|---|---|
| Номер платежа | toza ID |
| **Неверная сумма** | `500000` |

**Kutilgan:** `CheckPerformTransaction` → `-31001`, `CreateTransaction` → `-31001`.

### 5.3 Несуществующий счет

| Maydon | Qiymat |
|---|---|
| Номер платежа | toza ID (песочница o'zi buzib yuboradi) |

**Kutilgan:** `-31050`, javobda `"data": "payment_id"` bo'lishi shart.

### 5.4 CheckPerformTransaction · Ожидает оплаты

| Maydon | Qiymat |
|---|---|
| Номер платежа | toza ID |
| Cумма оплаты | `5000000` |

**Kutilgan:** `{"result": {"allow": true}}`

### 5.5 CreateTransaction

| Maydon | Qiymat |
|---|---|
| Номер платежа | toza ID |
| Cумма оплаты | `5000000` |

**Kutilgan** (5 ta kichik test):

| Qadam | Javob |
|---|---|
| yangi tranzaksiya | `state: 1` |
| o'sha `id` bilan takroriy | **aynan o'sha** javob, `state: 1` |
| boshqa `id`, o'sha hisob | `-31053` |

### 5.6 PerformTransaction

| Maydon | Qiymat |
|---|---|
| ID созданной транзакции | avtomatik to'ladi |

**Kutilgan:** `state: 2`, takroriy chaqiruvda **aynan o'sha** javob.

Shu qadamdan keyin to'lov CONFIRMED bo'ladi va qarzga taqsimlanadi.

### 5.7 CancelTransaction · holat 2

**Kutilgan:** `state: -2`. To'lov REVERSED bo'ladi, taqsimotlar
qaytariladi (hisoblanmalar yana qarzga tushadi).

### 5.8 CheckTransaction

**Kutilgan:** `create_time`, `perform_time`, `cancel_time`, `transaction`,
`state`, `reason` — oltalasi ham.

Mavjud bo'lmagan tranzaksiya → `-31003`.

### 5.9 GetStatement

Parametrsiz. **Kutilgan:** oraliqdagi tranzaksiyalar ro'yxati, yaratilish
vaqti bo'yicha o'sish tartibida.

### 5.10 Hisob holati stsenariylari

| Holat | Qanday tayyorlanadi | Kutilgan |
|---|---|---|
| Ожидает оплаты | toza ID | `allow: true` |
| **Платеж обрабатывается** (band) | `CreateTransaction` qilingan ID | `-31053` |
| **Заблокирован** (to'langan/bekor) | `Perform` yoki `Cancel` qilingan ID | `-31051` |
| Не существует | mavjud bo'lmagan UUID | `-31050` |

> Ishlatilgan ID larni tashlamang — ular **tayyor stsenariy**. Bir marta
> `Create` qilingan ID «band hisob» testiga, `Cancel` qilingani esa
> «bloklangan hisob» testiga qayta-qayta yaraydi.

---

## 6. Xato kodlari ma'lumotnomasi

### Hisob (account) xatolari — `-31099 … -31050`

Bu oraliq **majburiy**: hisob holatiga oid har qanday muammo shu yerda
bo'lishi shart.

| Kod | Qachon | Foydalanuvchi ko'radi |
|---|---|---|
| `-31050` | topilmadi · format buzuq · sotuvchi bloklangan | «To'lov topilmadi yoki muddati tugagan» |
| `-31051` | allaqachon to'langan yoki bekor qilingan | «Bu to'lov allaqachon amalga oshirilgan yoki bekor qilingan» |
| `-31052` | checkout / QR sessiya muddati tugagan | «To'lov havolasining muddati tugagan. Yangisini oling» |
| `-31053` | hisobda tugallanmagan tranzaksiya bor | «Bu to'lov bo'yicha boshqa amal bajarilmoqda. Biroz kuting» |

### Boshqa kodlar

| Kod | Ma'nosi | Qaysi metodda |
|---|---|---|
| `-32504` | avtorizatsiya xatosi | hammasida |
| `-32601` | metod topilmadi | noma'lum metod |
| `-32400` | ichki tizim xatosi | hammasida |
| `-31001` | summa mos emas | CheckPerform, Create |
| `-31003` | tranzaksiya topilmadi | Perform, Cancel, Check |
| `-31007` | bajarilgan, bekor qilib bo'lmaydi | Cancel |
| `-31008` | **tranzaksiya** holati ruxsat bermaydi | Perform, Cancel |

### ⚠️ `-31008` ni hisob xatosida ISHLATMANG

Bu eng nozik qoida:

| Nima haqida | Qaysi kod |
|---|---|
| **HISOB** holati (to'langan, band, muddati o'tgan) | `-31050…-31099` |
| **TRANZAKSIYA** holati (state 2 dan 1 ga qaytib bo'lmaydi) | `-31008` |

Песочница `CheckPerformTransaction` dan `-31008` kelsa uni **yiqilish**
deb belgilaydi. Kodda bu chegara `_assert_payable()` va
`_perform_transaction()` / `_cancel_transaction()` orasidan o'tadi.

---

## 7. Muammolarni bartaraf etish

| Alomat | Sabab | Yechim |
|---|---|---|
| Hamma so'rov `-32504` | kalit bo'sh (tirnoqsiz `#`) yoki `PAYME_ENABLED=false` | §1.1, keyin restart |
| Hamma so'rov HTTP 301 | Endpoint URL oxirida `/` yo'q | §2 |
| HTTP 429 | throttling (tuzatilgan, qaytsa regressiya) | §8 |
| `-31001` to'g'ri summada | «Cумма оплаты» ga tiyin emas, so'm yozilgan | `5000000` yozing |
| `allow: true` kutilganda `-31001` | «неверная сумма» maydoniga to'g'ri summa yozilgan | `500000` yozing |
| 2-qadamda `-31053` | ID allaqachon ishlatilgan | yangi ID oling |
| `-31051` kutilmaganda | ID `Perform`/`Cancel` qilingan | yangi ID oling |
| `-31052` | checkout muddati o'tgan (10 daqiqa) | §3 dagi skript bilan yangi to'lov |
| `-32601` | metod amalga oshirilmagan (`ChangePassword`) | §9 |

Serverda nima bo'lganini ko'rish:

```bash
cd BackendBoyTerakPay && ./venv/bin/python - <<'PY'
import os, django
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.dev")
django.setup()
from apps.payments.models import PaymentWebhookEvent
for e in PaymentWebhookEvent.objects.order_by("-received_at")[:25]:
    pid = str(e.payment.public_id)[:8] if e.payment_id else "—"
    print(f"{e.received_at:%H:%M:%S} {e.operation or '(auth xato)':<26} "
          f"kod={e.response_code:<7} to'lov={pid} {e.note}")
PY
```

> Bu jadval **xom tanani saqlamaydi** — faqat SHA-256 xeshi. Kalit,
> `Authorization` sarlavhasi va PII u yerga hech qachon tushmaydi.

---

## 8. Sertifikatsiya davomida topilgan va tuzatilgan xatolar

Uchtasi ham песочница'ni to'sib qo'ygan edi. Regressiya testlari bilan
qoplangan — qaytib kelsa testlar yiqiladi.

### 8.1 Throttling provayder callbacklarini sindirardi

**Alomat:** `Недопустимый HTTP статус … получен 429`

`payment_write` (30/min) scope'i provayder endpointlariga ham qo'yilgan
edi. Barcha callbacklar Payme serverining bitta IP'sidan kelgani uchun
**butun bozor bitta chelakni bo'lishardi**, va DRF'ning 429 javobi
protokolni buzardi (status 200 emas, `error` esa satr).

**Tuzatish:** [api_providers.py](../BackendBoyTerakPay/apps/payments/api_providers.py) —
provayder callbacklarida throttling butunlay o'chirildi. Himoya
provayder autentifikatsiyasida (u yopiq holatda ishlamay qoladi); hajm
cheklovi kerak bo'lsa nginx darajasida.

**Yo'l-yo'lakay:** natija sahifasi 2 soniyada bir, 30 marta so'raydi —
bu ham roppa-rosa 30/min edi. Unga alohida `payment_status: 120/min`
berildi.

### 8.2 Hisob holati `-31008` qaytarardi

**Alomat:** `Состояние счета "Заблокирован"` testi yiqilardi.

`_assert_payable()` «allaqachon to'langan» va «muddati o'tgan» uchun
`-31008` berardi — bu esa tranzaksiya holati kodi.

**Tuzatish:** uchta aniq account kodi joriy qilindi (`-31051`, `-31052`
va mavjud `-31050`). Xabar ham aniqlashtirildi: allaqachon to'lagan odam
«to'lov topilmadi» degan chalg'ituvchi matnni ko'rardi.

### 8.3 Ikki metod bir-biriga zid javob berardi

**Alomat:** `Состояние счета "Платеж обрабатывается"` testi yiqilardi.

«Band hisob» qulfi faqat `CreateTransaction` ichida edi.
`CheckPerformTransaction` esa `allow: true` deb javob berardi. Payme
har doim avval `CheckPerform` chaqirgani uchun foydalanuvchi «to'lash
mumkin» ekranini ko'rib, keyin xatoga urilardi.

**Tuzatish:** qulf `_assert_payable()` ga ko'chirildi — endi u ikkala
metod uchun **yagona manba**, ya'ni ular ajralib keta olmaydi. Yangi
kod: `-31053`.

---

## 9. Ochiq masala — `ChangePassword`

**Hozirgi holat:** metod **amalga oshirilmagan**, `-32601` qaytaradi.

```json
{"error": {"code": -32601, "message": {...}, "data": "ChangePassword"}, "id": 1}
```

Ya'ni uni песочница'da ishga tushirish **kalitni almashtirmaydi** — u
shunchaki yiqiladi. Xavfsizlik nuqtai nazaridan zararsiz, lekin
«barcha metodlar tekshirildi» deb aytib bo'lmaydi.

### Nima uchun yozilmagan

Payme bu metod orqali kassa kalitini almashtiradi. Uni qo'llab-quvvatlash
demak — **web so'rovi orqali kelgan yangi sirni doimiy saqlash**. Bu
loyihaning asosiy qoidasiga tegadi: sirlar faqat `.env` da, kod va baza
ularni yozmaydi ([06-threat-model.md](06-threat-model.md)).

### Variantlar

| Variant | Nima bo'ladi | Xavf |
|---|---|---|
| **Yozmaslik** (hozirgi) | Payme kalitni almashtira olmaydi, qo'lda almashtiriladi | sertifikatsiyada so'ralishi mumkin |
| Bazaga saqlash | kalit `.env` dan bazaga ko'chadi | baza zaxirasi endi sir saqlaydi |
| `.env` ga yozish | jarayon o'z konfigini o'zgartiradi | web so'rovi fayl tizimiga yozadi + restart kerak |

Payme sertifikatsiyada bu metodni **talab qilsa** — variant tanlash
kerak. Talab qilmasa, hozirgi holat qoladi.

> Qaror qabul qilinmaguncha песочница'da bu bo'limni ishga tushirmang:
> u yashil bo'lmaydi va hisobotni chalkashtiradi.

---

## 10. Foydali buyruqlar

```bash
cd BackendBoyTerakPay

# To'lov testlari
./venv/bin/python -m pytest apps/payments -q

# Sozlama tekshiruvi (payments.E001 / E002)
./venv/bin/python manage.py check

# Solishtirish — HECH NARSANI o'zgartirmaydi
./venv/bin/python manage.py reconcile_payments --provider=payme
./venv/bin/python manage.py reconcile_payments --provider=payme --json

# .env o'zgargach MAJBURIY
systemctl restart boyterakpay-api.service
```

Jonli endpointni qo'lda sinash:

```bash
cd BackendBoyTerakPay
KEY=$(grep '^PAYME_TEST_KEY=' .env | cut -d= -f2- | sed "s/^'//;s/'$//")
curl -s -X POST https://boyterak.myweb.uz/api/v1/payments/payme/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic $(printf 'Paycom:%s' "$KEY" | base64 -w0)" \
  -d '{"method":"CheckPerformTransaction","params":{"amount":5000000,
       "account":{"payment_id":"<TOZA-ID>"}},"id":1}'
```

---

## 11. Productionga o'tish

Песочница yashil bo'lgach:

- [ ] `PAYME_ENV=production`
- [ ] `PAYME_MERCHANT_KEY` — kabinet kalitini kiriting (maxsus belgilar bo'lsa **bitta tirnoq**)
- [ ] `PAYME_TEST_KEY` — bo'shatish shart emas, `PAYME_ENV` o'zi tanlaydi
- [ ] Kabinetdagi Endpoint URL — o'sha manzil, oxirida `/`
- [ ] `systemctl restart boyterakpay-api.service`
- [ ] `manage.py check` — `payments.E001` yo'qligiga ishonch
- [ ] Checkout URL `https://checkout.paycom.uz/…` ga o'tganini tekshiring
- [ ] Test to'lovlarini yopish (§12)
- [ ] `PAYMENT_SESSION_TTL_SECONDS` ni qayta ko'rib chiqing — 10 daqiqa
      3-D Secure + SMS oqimi uchun tor bo'lishi mumkin

> Sertifikatsiya paytida ochiq ko'ringan kalitlarni **almashtiring**.

To'liq ro'yxat: [payment-production-checklist.md](payment-production-checklist.md).

---

## 12. Test yozuvlarini tozalash

Test to'lovlari bazada `note="Payme sandbox testi"` bilan qoladi. Ular
PENDING holatda turib `reconcile_payments` da `PENDING_STALE` beradi.

Payme ularni 12 soatdan keyin taymaut bo'yicha o'zi bekor qiladi
(`state=-1`, `reason=4`). Erta yopish kerak bo'lsa —
**o'chirmang**, qonuniy o'tish bilan yoping:

```python
from apps.payments import services
services.fail_payment(payment, reason="Sandbox test yozuvi")
```

> Tasdiqlangan moliyaviy yozuv hech qachon o'chirilmaydi — faqat
> reversal/adjustment ([05-financial-invariants.md](05-financial-invariants.md), F-3).

---

## Qisqa xotira kartasi

```
Cумма оплаты      -> 5000000        (aynan)
Неверная сумма    -> 500000         (ataylab boshqa)
payment_id        -> har runga YANGI
Endpoint URL      -> oxirida /
PAYME_TEST_KEY    -> 'bitta tirnoq ichida'
.env o'zgardi     -> restart
-31008            -> hisob xatosida ISHLATILMAYDI
```
