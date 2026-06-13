---
title: Filtreleme ve Sayfalama
sidebar_label: Filtreleme ve Sayfalama
sidebar_position: 3
---

# Filtreleme ve Sayfalama

[`POST /v1/calls/list`](index.md) çağrılarını daraltma ve sayfalama ile ilgili her şey: `cursor`, `limit`, `from_date` ve `to_date` parametreleri.

Çağrılar **en eskiden en yeniye** sırayla döner; sıralama, her çağrının size müsait hale geldiği ana göredir — çağrının başladığı zamana göre değil.

---

## Parametreler birlikte nasıl çalışır?

| İstek | Ne döner? |
|---|---|
| `from_date`, `to_date`, `cursor` ve `limit` yok | Asistana (veya squad'a) ait **en eski 100** müsait çağrı. Daha fazlası varsa `has_more` `true` olur ve `next_cursor` döner — sonraki 100 için onu geri gönderin. |
| Yalnızca `limit` (örn. `500`) | Tek sayfada en eski *N* çağrı (en çok 500). |
| Yalnızca `from_date` | `from_date` anından **itibaren** (dahil) müsait olan çağrılar, en eskiden başlayarak. `cursor` ile devam edin. |
| Yalnızca `to_date` | `to_date` anından **kesinlikle önce** müsait olan çağrılar, en eskiden başlayarak. `cursor` ile devam edin. |
| `from_date` + `to_date` | Yarı açık `[from_date, to_date)` aralığındaki çağrılar, en eskiden başlayarak. |
| Yukarıdakilerden herhangi biri **+ `cursor`** | Aynı sorgunun **sonraki sayfası**. Sayfalar arasında diğer tüm parametreleri aynı tutun — yalnızca `cursor` değişir. |

---

## `limit`

- Varsayılan **100**, en fazla **500**; her sayfaya uygulanır.
- **1–500** aralığı dışındaki bir değer `400 VALIDATION_FAILED` ile reddedilir.
- `limit` yalnızca sayfa boyutunu belirler — toplamda kaç çağrı çekebileceğinizi **sınırlamaz**. Tümünü okumak için `cursor` ile sayfalamaya devam edin.

## `cursor` {#cursors}

- **İlk** istekte göndermeyin.
- Her yanıt bir `pagination.next_cursor` döner. `has_more` `true` olduğu sürece bu değeri `cursor` olarak geri gönderip sonraki sayfayı alın.
- `has_more` `false` olunca durun (bu noktada `next_cursor` `null`'dır).
- Cursor **opaktır** — oluşturmayın veya değiştirmeyin. Cursor ile sayfalarken **aynı `assistant_id`/`squad_id`, `from_date`, `to_date` ve `limit` değerlerini tekrar gönderin**; cursor yalnızca o sorgudaki konumunuzu işaretler.
- Cursor değerlerini uzun süre (örneğin günlerce) saklamayın — tek bir senkronizasyon oturumu içinde kullanın. Düzenli/**artımlı** senkron için çalıştırmalar arasında cursor saklamayın; bunun yerine en son çektiğiniz noktayı hatırlayıp sonraki çalıştırmada `from_date` olarak gönderin. Cursor, *tek bir sorgunun içindeki* konumu işaretler; kalıcı bir watermark değildir. Bkz. [artımlı senkron rehberi](../../guides/incremental-sync.md).

```bash
# İlk istek (cursor yok)
curl -X POST https://api-vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id":7,"limit":100}'

# Yanıt: { "data": [...], "pagination": { "next_cursor": "X", "has_more": true } }

# Sonraki istek (next_cursor değerini kullanın)
curl -X POST https://api-vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id":7,"limit":100,"cursor":"X"}'

# has_more: false olunca durun
```

Cursor hataları:

| Durum | Kod | Anlamı |
|---|---|---|
| `400` | `INVALID_CURSOR` | Cursor çözümlenemedi. Önceki bir yanıttan alınan güncel bir cursor kullanın. |
| `400` | `MALFORMED_CURSOR` | Cursor içeriği beklenen yapıda değil. Cursor'u değiştirmeyin — döndürüldüğü biçimde kullanın. |

## Sayfalama nesnesi {#paginated}

Her sayfa aynı yapıyla sarmalanır:

```json
{
  "data": [ /* çağrılar */ ],
  "pagination": {
    "next_cursor": "eyJ0IjoiMjAyNi0wNS0xNVQxMDozMToyOC4wMDBaIiwiaSI6MTIzNDZ9",
    "has_more": true,
    "limit": 100
  }
}
```

| Alan | Tür | Açıklama |
|---|---|---|
| `data` | array | Bu sayfadaki çağrılar. |
| `pagination.next_cursor` | string \| null | Sonraki sayfa için opak cursor. Son sayfada `null`. |
| `pagination.has_more` | boolean | Bu sayfadan sonra başka çağrı olup olmadığı. |
| `pagination.limit` | int | Bu istekte uygulanan limit. |

---

## Tarihler: `from_date` / `to_date` {#range-semantics}

İki parametre de bir çağrının size müsait hale geldiği anı, yarı açık **`[from_date, to_date)`** aralığında filtreler:

- `from_date` **dahildir** (>=) — "bu andan itibaren"
- `to_date` **hariçtir** (`<`) — "bu ana kadar, bu an dahil değil"

Ardışık aralıklar çakışmadan ve boşluk bırakmadan zincirlenir. Birini tek başına ya da ikisini birlikte gönderebilir; en baştan taramak için ikisini de boş bırakabilirsiniz.

**Yalnızca tarih içeren `to_date` için kolaylık**: `to_date=2026-05-23` (sadece tarih), otomatik olarak `< 2026-05-24T00:00:00Z`'ye genişletilir; böylece **23 Mayıs gününün tamamı dahil** olur.

`from_date >= to_date` durumu `DATE_RANGE_INVALID` (400) ile reddedilir.

### Kabul edilen biçimler

Tüm datetime değerleri **UTC referansıyla ISO 8601** biçimindedir.

| Biçim | Örnek | Anlamı |
|---|---|---|
| Yalnızca tarih | `2026-05-23` | UTC gününün başlangıcı (`00:00:00`) |
| UTC datetime | `2026-05-23T15:30:00Z` | UTC 15:30 |
| Milisaniye duyarlıklı UTC | `2026-05-23T15:30:00.123Z` | UTC 15:30 ve 123 milisaniye |
| Offset'li datetime | `2026-05-23T15:30:00+03:00` | Türkiye saatiyle 15:30 (= UTC 12:30) |

### Reddedilen biçimler

| Biçim | Hata Kodu | Sorun |
|---|---|---|
| `2026-05-23 15:30:00` | `INVALID_DATE_FORMAT` | `T` ayracı yerine boşluk |
| `2026-05-23T15:30:00` | `MISSING_TIMEZONE` | Saat dilimi yok (`Z` veya offset gerekir) |
| `05/23/2026` | `INVALID_DATE_FORMAT` | ISO 8601 değil — yıl/ay/gün sırası belirsiz |
| `23-05-2026` | `INVALID_DATE_FORMAT` | GG-AA-YYYY ISO 8601 değil |
| `2026-13-01` | `INVALID_DATE_FORMAT` | Geçersiz ay (13) |
| `2026-02-30` | `INVALID_DATE_FORMAT` | Geçersiz gün (30 Şubat) |

Reddedilen her değer, yukarıdaki hata koduyla birlikte yapılandırılmış bir 400 döner; makine-okunabilir ayrıntı (`extensions.details` içinde) ilgili alanı, gönderdiğiniz değeri ve örnek biçimleri içerir. Bkz. [Hata Kodları kataloğu](../../errors.md).

### Türkiye saatinin kullanımı {#turkey-time}

Türkiye, yıl boyunca sabit biçimde **UTC+3** dilimindedir; yaz/kış saati yoktur. Aynı anı belirtmenin üç eşdeğer yolu vardır:

```
A) Offset belirterek, Türkiye saatiyle (önerilen):
   "2026-05-23T15:30:00+03:00"

B) UTC'ye dönüştürerek (Türkiye saatinden 3 saat çıkararak):
   "2026-05-23T12:30:00Z"

C) UTC'ye dönüştürüp offset'i açıkça yazarak:
   "2026-05-23T12:30:00+00:00"
```

**Üçü de aynı ana işaret eder.** Hangisini gönderirseniz gönderin, sunucu değeri UTC'ye normalleştirir.

### Geçerli aralıklar

| Girdi | Geçerli Aralık (UTC) |
|---|---|
| `from_date=2026-05-23` | `>= 2026-05-23T00:00:00Z` |
| `to_date=2026-05-23` (kolaylık) | `< 2026-05-24T00:00:00Z` |
| `from=2026-05-23` + `to=2026-05-23` | 23 Mayıs gününün tamamı (UTC) |
| `from_date=2026-05-23T15:00:00+03:00` | `>= 2026-05-23T12:00:00Z` |
| `to_date=2026-05-23T18:00:00Z` | `< 2026-05-23T18:00:00Z` (kesin) |
| `from=2026-05-23T09:00:00+03:00` + `to=2026-05-23T17:00:00+03:00` | Türkiye mesai saatleri (UTC 06:00 — 14:00) |

---

## Reçeteler

**Tek bir gün** — date-only `to_date` kolaylığı 23 Mayıs'ın tamamını kapsar:

```json
{ "assistant_id": 7, "from_date": "2026-05-23", "to_date": "2026-05-23" }
```

**Bir takvim ayı** — `to_date` hariç olduğundan, Haziran'a taşmadan 1–31 Mayıs:

```json
{ "assistant_id": 7, "from_date": "2026-05-01", "to_date": "2026-06-01" }
```

**Türkiye mesai saatleri:**

```json
{
  "assistant_id": 7,
  "from_date": "2026-05-23T09:00:00+03:00",
  "to_date": "2026-05-23T17:00:00+03:00"
}
```

**Belirli bir andan bu yana her şey** — "şu ana kadar" için `to_date`'i boş bırakın:

```json
{ "assistant_id": 7, "from_date": "2026-05-23T11:00:00.000Z" }
```

**Aralıkları boşluksuz zincirleme** — aralıklar yarı açık olduğundan, bir aralığın `to_date`'i bir sonrakinin `from_date`'i olabilir; hiçbir kayıt iki kez sayılmaz, hiçbiri atlanmaz:

```json
{ "from_date": "2026-05-23T00:00:00Z", "to_date": "2026-05-24T00:00:00Z" }
{ "from_date": "2026-05-24T00:00:00Z", "to_date": "2026-05-25T00:00:00Z" }
```

### Sık yapılan hatalar

| Gönderdiğiniz | Sonuç |
|---|---|
| `"2026-05-23 15:30:00"` (boşluk) | `400 INVALID_DATE_FORMAT` — `T` kullanın |
| `"2026-05-23T15:30:00"` (saat dilimi yok) | `400 MISSING_TIMEZONE` — `Z` veya `+03:00` ekleyin |
| `from_date`, `to_date`'den sonra | `400 DATE_RANGE_INVALID` |
| `"23-05-2026"` veya `"05/23/2026"` | `400 INVALID_DATE_FORMAT` — yalnızca ISO 8601 |
