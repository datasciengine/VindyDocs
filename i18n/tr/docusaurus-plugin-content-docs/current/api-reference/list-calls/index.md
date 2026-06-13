---
title: Çağrıları Listele
sidebar_label: Çağrıları Listele
sidebar_position: 2
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `POST /v1/calls/list`

Bir asistanınıza veya squad'ınıza ait çağrıları döndürür; her çağrı kendi dökümü (transcript), structured output'larınızın çıkardığı veriler, eklediğiniz metadata ve hazır olduğunda bir ses kaydı bağlantısıyla birlikte gelir. Sonuçlar opak bir cursor ile sayfa sayfa gelir ve istenirse bir tarih aralığına daraltılabilir.

:::info Yarım veri döndürülmez
Yalnızca **size gösterilmeye hazır** çağrılar döndürülür. Bir çağrının hazır sayılması için:

- Çağrının sona ermiş olması (durum: `completed`, `failed` veya `cancelled`) ve
- Varsa, ses kaydının aktarımının tamamlanmış olması (başarıyla ya da kalıcı olarak başarısız biçimde)

gerekir. Hâlâ devam eden çağrılar (örneğin ses kaydı aktarımı sürenler) bu listede **hiçbir zaman** yer almaz. Bu davranış, senkronizasyon mantığınızın idempotent çalışmasını sağlar.
:::

Bir çağrı **sona erdikten kısa süre sonra** müsait hâle gelir — ses kaydı kalıcı depolamaya aktarıldığında. Bu genellikle birkaç saniye, uzun kayıtlarda zaman zaman birkaç dakika sürer. Bir çağrı hazır olana dek ne bu listede ne de `call-ended` webhook'unda görünür; bu yüzden az önce biten bir çağrı, hemen ardından attığınız istekte henüz yer almayabilir.

:::tip Pull ve push aynı sinyali paylaşır
Bu endpoint, [`call-ended` webhook'unun](../webhooks.md) **pull** karşılığıdır: bir çağrı, hazır hâle geldiği anda hem burada görünür hem de o webhook'u tetikler. Anlık teslim için webhook'u; istediğiniz anda çekmek veya kaçırmış olabileceklerinizi tamamlamak için bu endpoint'i kullanın.
:::

---

## İstek

```http
POST https://api-vindy.vinter.me/v1/calls/list
Authorization: Bearer <api-key>
Content-Type: application/json

{
  "assistant_id": 7,
  "from_date": "2026-05-01",
  "to_date": "2026-06-01",
  "limit": 100,
  "cursor": null
}
```

## Gövde parametreleri

| Alan | Tür | Zorunlu | Varsayılan | Açıklama |
|---|---|---|---|---|
| `assistant_id` | int | **biri zorunlu** | — | Asistan kimliğine göre filtreler. `squad_id` ile birlikte kullanılamaz. |
| `squad_id` | UUID | **biri zorunlu** | — | Squad kimliğine göre filtreler. `assistant_id` ile birlikte kullanılamaz. |
| `from_date` | ISO 8601 string | hayır | — | Bu andan itibaren erişilebilir hâle gelen çağrılar. Bkz. [Filtreleme ve Sayfalama](filtering-pagination.md). |
| `to_date` | ISO 8601 string | hayır | — | Bu andan kesinlikle önce erişilebilir hâle gelen çağrılar. Bkz. [Filtreleme ve Sayfalama](filtering-pagination.md). |
| `limit` | int | hayır | `100` | Bu sayfadaki en fazla kayıt sayısı. Aralık: 1–500. |
| `cursor` | string | hayır | — | Önceki yanıttaki `next_cursor` değerinden alınan opak cursor. İlk istekte gönderilmez. |

**Filtre seçimi — `assistant_id` mı `squad_id` mı?** Her çağrı ya bir asistana ya da bir squad'a aittir; bu yüzden bu ikisinden tam olarak birini gönderirsiniz — ikisini birden değil, hiçbirini de değil. Tek bir asistanın yürüttüğü çağrıları görmek için `assistant_id`, bir squad üzerinden geçen tüm çağrıları (squad'a bağlı tüm asistanlar dâhil) görmek için `squad_id` kullanın. Her iki kimliği de [`GET /v1/assistants`](../list-assistants.md) yanıtından alırsınız.

**Doğrulama kuralları:**

- `assistant_id` veya `squad_id` parametrelerinden yalnızca biri zorunludur; ikisinin birlikte gönderilmesi 400 döndürür.
- `from_date >= to_date` durumu 400 (`DATE_RANGE_INVALID`) döndürür.
- Tarih davranışı ve kabul edilen biçimler için [Filtreleme ve Sayfalama](filtering-pagination.md) sayfasına bakabilirsiniz.

## Sayfalama ve filtreleme

Sonucu iki bağımsız mekanizma şekillendirir ve bu ikisi sorunsuz biçimde birlikte çalışır:

- **Tarih aralığı** (`from_date` / `to_date`) *hangi* çağrıların kapsama gireceğini belirler. İkisi de opsiyoneldir: tümünü taramak için boş bırakın ya da bir pencere tanımlamak için birini veya ikisini verin.
- **Cursor** (`cursor` / `limit`) bu kapsamın *içinde* sayfa sayfa ilerler.

İkisini ayrı ayrı da, birlikte de kullanabilirsiniz. Tarih aralığı ve cursor olmadan, tüm çağrılarınız arasında **en eskiden en yeniye** gezinirsiniz: ilk istek en eski `limit` kadar çağrıyı (varsayılan 100) döndürür, geriye kayıt kalmayana dek devam edersiniz. Bir tarih aralığı eklediğinizde de aynı şekilde, yalnızca o pencere içinde gezinirsiniz. Her durumda kural aynıdır: filtrelerinizi ilk istekte gönderin; sonraki her istekte aldığınız `next_cursor` değerini — **değiştirmeden** — geri gönderin ve `assistant_id`/`squad_id`, `from_date`, `to_date`, `limit` değerlerini olduğu gibi koruyun. Cursor, konumunuzu *o belirli sorgunun içinde* kodlar; bu yüzden gezinme sırasında bir filtreyi değiştirmek anlamsız sonuçlar üretir. `has_more` `false` olduğunda (bu noktada `next_cursor` da `null` olur) iş tamamlanmıştır.

Parametrelerin tam referansı, kabul edilen tarih biçimleri ve hazır reçeteler **[Filtreleme ve Sayfalama](filtering-pagination.md)** sayfasındadır.

## Yanıt (200 OK)

```json
{
  "data": [
    {
      "call_id": 12345,
      "call_status": "completed",
      "call_assistant_id": 7,
      "call_squad_id": null,
      "call_phone_number": "+905551112233",
      "call_bound_type": "outbound",
      "call_started_at": "2026-05-15T10:30:00.000Z",
      "call_ended_at": "2026-05-15T10:31:27.000Z",
      "call_created_at": "2026-05-15T10:29:55.000Z",
      "call_duration_seconds": 87,
      "call_end_reason": "customer-ended-call",
      "call_transcript": "AI: Merhaba, ben yapay zeka asistanı Vindy. Müşteri memnuniyeti anketimiz kapsamında size birkaç kısa soru sormak istiyorum — şu an uygun musunuz?\nUser: Evet, müsaitim.\nAI: Teşekkürler. Öncelikle yaşınızı öğrenebilir miyim?\nUser: Otuz iki.\nAI: Aldığınız hizmetten genel memnuniyetinizi 1 ile 5 arasında nasıl puanlarsınız?\nUser: 4 diyebilirim.\nAI: Peki talebinizin ne kadar hızlı çözüldüğünü 1 ile 5 arasında nasıl değerlendirirsiniz?\nUser: 5.\nAI: Hizmetimizi bir arkadaşınıza veya iş arkadaşınıza tavsiye eder misiniz?\nUser: Evet, ederim.\nAI: Çok teşekkür ederiz, görüşleriniz bizim için çok değerli. İyi günler dilerim!\nUser: Rica ederim, size de.\n",
      "call_structured_data": {
        "9b1c7e2a-4d3f-4a8b-bc12-5e6f7a8b9c01": {
          "name": "Memnuniyet Anketi",
          "result": {
            "age": 32,
            "overall_satisfaction": 4,
            "support_speed": 5,
            "would_recommend": true
          }
        }
      },
      "call_metadata": { "crm_contact_id": "CNT-90412" },
      "call_recording": {
        "available": true,
        "url": "https://...",
        "expires_at": "2026-06-04T12:34:56.789Z"
      }
    },
    {
      "call_id": 12346,
      "call_status": "completed",
      "call_assistant_id": 7,
      "call_squad_id": null,
      "call_phone_number": "+905554445566",
      "call_bound_type": "outbound",
      "call_started_at": "2026-05-15T11:02:10.000Z",
      "call_ended_at": "2026-05-15T11:02:20.000Z",
      "call_created_at": "2026-05-15T11:01:58.000Z",
      "call_duration_seconds": 10,
      "call_end_reason": "customer-ended-call",
      "call_transcript": "AI: Merhaba, ben Vindy; müşteri memnuniyeti anketi için arıyorum. Şu an uygun musunuz?\nUser: Pardon, yanlış numara.\n",
      "call_structured_data": null,
      "call_metadata": null,
      "call_recording": {
        "available": false
      }
    }
  ],
  "pagination": {
    "next_cursor": "eyJ0IjoiMjAyNi0wNS0xNVQxMDozMToyOC4wMDBaIiwiaSI6MTIzNDZ9",
    "has_more": true,
    "limit": 100
  }
}
```

`call_transcript` tek bir metin dizesidir; içindeki her konuşma sırası bir satır sonu (`\n`) ile ayrılır. JSON satır sonlarını kaçışlı yazdığı için yukarıdaki değer tek satırda görünür. Gerçek satır sonlarıyla görüntülendiğinde ilk çağrının transcript'i şöyledir:

```text
AI: Merhaba, ben yapay zeka asistanı Vindy. Müşteri memnuniyeti anketimiz kapsamında size birkaç kısa soru sormak istiyorum — şu an uygun musunuz?
User: Evet, müsaitim.
AI: Teşekkürler. Öncelikle yaşınızı öğrenebilir miyim?
User: Otuz iki.
AI: Aldığınız hizmetten genel memnuniyetinizi 1 ile 5 arasında nasıl puanlarsınız?
User: 4 diyebilirim.
AI: Peki talebinizin ne kadar hızlı çözüldüğünü 1 ile 5 arasında nasıl değerlendirirsiniz?
User: 5.
AI: Hizmetimizi bir arkadaşınıza veya iş arkadaşınıza tavsiye eder misiniz?
User: Evet, ederim.
AI: Çok teşekkür ederiz, görüşleriniz bizim için çok değerli. İyi günler dilerim!
User: Rica ederim, size de.
```

:::note Başarısız ve iptal edilen çağrılar da listede döner
Liste yalnızca başarılı çağrıları değil; `completed`, `failed` ve `cancelled` çağrıları döndürür. Hiç bağlanmamış bir çağrının (örneğin `cancelled` bir çağrı) konuşması veya ses kaydı olmaz; bu yüzden zaman temelli alanları `null` olur ve `call_recording.available` `false` döner. Kodunuz bu `null` değerlere dayanıklı olmalıdır:

```json
{
  "call_id": 12347,
  "call_status": "cancelled",
  "call_assistant_id": 7,
  "call_squad_id": null,
  "call_phone_number": "+905557778899",
  "call_bound_type": "outbound",
  "call_started_at": null,
  "call_ended_at": null,
  "call_created_at": "2026-05-15T11:05:00.000Z",
  "call_duration_seconds": null,
  "call_end_reason": null,
  "call_transcript": null,
  "call_structured_data": null,
  "call_metadata": { "crm_contact_id": "CNT-90418" },
  "call_recording": { "available": false }
}
```
:::

## Yanıt alanları

**Üst düzey**

| Alan | Tür | Açıklama |
|---|---|---|
| `data` | array | Bu sayfadaki çağrılar. |
| `pagination` | object | Standart [sayfalama nesnesi](filtering-pagination.md#paginated). |

**Call nesnesi**

| Alan | Tür | Açıklama |
|---|---|---|
| `call_id` | int | Çağrının sistemimizdeki kalıcı ve benzersiz kimliği. Bir endpoint `:callId` aldığı her yerde kullanılır — örneğin bu çağrıyı getirmek için [`GET /v1/calls/:callId`](../get-call.md), güncel bir kayıt bağlantısı için [`GET /v1/calls/:callId/recording-url`](../get-recording-url.md) veya çağrıyı iptal etmek için [`POST /v1/calls/:callId/cancel`](../cancel-call.md) — ayrıca çağrıyı [`call-ended` webhook](../webhooks.md) içeriğiyle eşleştirmek için. |
| `call_status` | string | `completed` \| `failed` \| `cancelled` |
| `call_assistant_id` | int \| null | Çağrıyı yöneten asistanın kimliği (squad çağrılarında null). |
| `call_squad_id` | UUID \| null | Çağrı bir squad üzerinden yapıldıysa squad kimliği. |
| `call_phone_number` | string | Aranan veya arayan numara (mevcut olduğunda E.164 biçiminde). |
| `call_bound_type` | string \| null | `inbound` \| `outbound` \| `null` |
| `call_started_at` | ISO 8601 (UTC) \| null | Çağrının fiilen başladığı an — **UTC** cinsinden, `Z` ekli ve milisaniye hassasiyetli bir ISO-8601 zaman damgası (örn. `2026-05-15T10:30:00.000Z`). Görüntülemek için kendi yerel saat diliminize çevirin. Çağrı hiç bağlanmadıysa `null`. |
| `call_ended_at` | ISO 8601 (UTC) \| null | Çağrının sona erdiği an, aynı ISO-8601 UTC formatında. Çağrı hiç bağlanmadıysa `null`. |
| `call_created_at` | ISO 8601 (UTC) | Çağrı kaydının sistemimizde oluşturulduğu an, aynı ISO-8601 UTC formatında. |
| `call_duration_seconds` | int \| null | Çağrı süresi (saniye). |
| `call_end_reason` | string \| null | Çağrının nasıl sona erdiğini belirten teknik kod. Bkz. [Bitiş nedenleri](#end-reasons). |
| `call_transcript` | string \| null | Görüşmenin düz metin dökümü. Her konuşma sırası `AI:` (asistan) veya `User:` (arayan) ön ekiyle başlar; sıralar `\n` ile ayrılır. Çok kısa veya başarısız çağrılarda boş ya da null olabilir. |
| `call_structured_data` | object \| null | Yapay zekânın çıkardığı veri; çağrıya uygulanan her structured output için anahtarlanmış bir nesnedir. Her anahtar, [`GET /v1/assistants`](../list-assistants.md) yanıtında dönen structured output `id` değeridir; karşılığındaki değer ise ilgili çıktının `name` değerini ve çıkarılan `result` verisini içerir. Yapısal veri üretilmediyse `null` döner. |
| `call_metadata` | object \| null | Çağrıyı [`POST /v1/calls/bulk`](../bulk-create-calls.md) ile oluştururken eklediğiniz metadata; korelasyon için size aynen geri döner. Çağrı metadata olmadan oluşturulduysa `null`. Kurallar için bkz. [Metadata](../bulk-create-calls.md#metadata). |
| `call_recording` | object | Ses kaydı durumu ve bağlantısı (aşağıda). |

**`call_recording` nesnesi**

| Alan | Tür | Açıklama |
|---|---|---|
| `available` | bool | Bu çağrı için indirilebilir bir ses kaydının mevcut olup olmadığı. |
| `url` | string \| yok | 24 saat geçerli, imzalı (presigned) bağlantı. Yalnızca `available: true` olduğunda bulunur. |
| `expires_at` | ISO string \| yok | Bağlantının geçerliliğini yitireceği an (UTC). Yalnızca `available: true` olduğunda bulunur. |

### `call_recording.available: false` ne anlama gelir? {#recording-not-available}

Bu durum **kalıcıdır (terminal)**; yeniden denemek sonucu değiştirmez. Bir çağrı bu listede görünüyorsa, ses kaydı durumu kesinleşmiş demektir. Bu endpoint, ses kaydı aktarımı hâlâ süren çağrıları zaten döndürmez. `available: false` aşağıdaki durumlardan birini ifade eder:

- Çağrı için herhangi bir ses kaydı üretilmemiştir (örneğin hiç ses alınamadan biten çok kısa veya başarısız bir çağrı).
- Ses kaydının kalıcı depolamaya aktarımı **kalıcı olarak başarısız olmuştur**.

Kesin nedeni öğrenmek için [`GET /v1/calls/:callId/recording-url`](../get-recording-url.md) endpoint'i çağırın; yanıttaki `recording_status` alanı durumu net biçimde belirtir. Kaydın var olması gerektiğini düşünüyorsanız Vindy ekibiyle iletişime geçin.

:::note Panelle olası farklılık
Vindy yönetim paneli, ses kayıtlarını başka kaynaklardan (örneğin geçici bir sağlayıcı bağlantısından) gösterebilir. API ise güvenlik gereği yalnızca kalıcı depolamaya yazılmış kayıtları sunar. Bir kaydın panelde görünüp API üzerinden görünmemesi beklenen bir durumdur; **müşteri tarafı için bağlayıcı olan, API yanıtıdır**.
:::

### Yapısal veri şekilleri

`call_structured_data`, structured output'unuzun çıkardığı veriyi olduğu gibi döndürür; bu nedenle bir `result` skaler değerler, iç içe nesneler ve diziler (nesne dizileri dahil) içerebilir. Örneğin bir *Order Summary* çıktısı şöyle dönebilir:

```json
{
  "call_structured_data": {
    "c7a1e4d2-8b6f-4c3a-9d10-2f5e7a8b9c34": {
      "name": "Order Summary",
      "result": {
        "customer_name": "Jane Doe",
        "callback_requested": false,
        "orders": [
          { "product": "Wireless Keyboard", "quantity": 2, "in_stock": true },
          { "product": "USB-C Cable", "quantity": 5, "in_stock": false }
        ],
        "shipping": {
          "city": "Istanbul",
          "methods": ["standard", "express"]
        }
      }
    }
  }
}
```

`result`'un şekli, ilgili structured output için tanımladığınız şemayı birebir yansıtır ([`GET /v1/assistants`](../list-assistants.md) yanıtında döner); böylece alan alan ayrıştırabilirsiniz.

## Çağrı bitiş nedenleri {#end-reasons}

`call_end_reason` alanı, çağrının nasıl sona erdiğini belirten teknik bir koddur. **Geniş bir enum'dur**; en sık karşılaşılan değerler şunlardır:

| Değer | Açıklama |
|---|---|
| `customer-ended-call` | Müşteri (son kullanıcı) görüşmeyi sonlandırdı |
| `assistant-ended-call` | Asistan görüşmeyi sonlandırdı (örneğin konuşma doğal biçimde tamamlandı) |
| `customer-did-not-answer` | Giden çağrı: müşteri yanıt vermedi |
| `customer-busy` | Giden çağrı: telefon çaldı ancak müşteri yanıtlamadan çağrıyı reddetti (hat meşgul veya çağrı reddedildi) |
| `phone-call-provider-closed-websocket` | Telefon sağlayıcısı bağlantıyı düşürdü |
| `exceeded-max-duration` | Azami çağrı süresine ulaşıldı |
| `silence-timed-out` | Uzun sessizlik nedeniyle zaman aşımı |
| `pipeline-error-*` | Çağrı hattında bir hata (sağlayıcı, model vb.) |
Yeni sağlayıcılar ve bileşenler eklendikçe liste genişlediğinden, başka değerler de görülebilir. Bilinen değerlerden oluşan sabit bir liste tutuyorsanız, **tanımadığınız bir nedenle karşılaştığınızda hata fırlatmamalısınız**; değeri log'layıp işleme devam edin.

## Hatalar

| Durum | Kod | Açıklama |
|---|---|---|
| `400` | `VALIDATION_FAILED` | `assistant_id` ve `squad_id` ya hiç verilmemiş ya da ikisi birlikte verilmiş; `limit` aralık dışı vb. |
| `400` | `DATE_RANGE_INVALID` | `from_date >= to_date` |
| `400` | `INVALID_DATE_FORMAT` | Tarih, kabul edilen biçimlere uymuyor |
| `400` | `MISSING_TIMEZONE` | Tarih-saat değerinde `Z` veya offset eksik |
| `400` | `INVALID_CURSOR` | Cursor çözümlenemedi |
| `400` | `MALFORMED_CURSOR` | Cursor içeriği beklenen yapıda değil |
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` | Kimlik doğrulama hataları |
| `500` | `HTTP_500` | Sunucu hatası — `extensions.code` içinde yer alır; bildirirken `requestId` değerini belirtin |

## Örnekler

### Tüm sayfalarda gezinme

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
# İlk istek (cursor yok)
curl -X POST https://api-vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id":7,"limit":100}'

# Yanıt: { "data": [100 çağrı], "pagination": { "next_cursor": "X", "has_more": true } }

# Sonraki istek (next_cursor değerini kullanın)
curl -X POST https://api-vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id":7,"limit":100,"cursor":"X"}'

# has_more: false döndüğünde durun
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
async function listAllCalls(assistantId) {
  const calls = [];
  let cursor = undefined;

  do {
    const response = await fetch("https://api-vindy.vinter.me/v1/calls/list", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.VINDY_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ assistant_id: assistantId, limit: 100, cursor }),
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(`${error.code}: ${error.message}`);
    }

    const body = await response.json();
    calls.push(...body.data);
    cursor = body.pagination.next_cursor;
  } while (cursor);

  return calls;
}

const calls = await listAllCalls(7);
console.log(`${calls.length} çağrı`);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

def list_all_calls(assistant_id):
    calls = []
    cursor = None

    while True:
        payload = {"assistant_id": assistant_id, "limit": 100}
        if cursor:
            payload["cursor"] = cursor

        response = requests.post(
            "https://api-vindy.vinter.me/v1/calls/list",
            headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
            json=payload,
        )
        if not response.ok:
            error = response.json()
            raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

        body = response.json()
        calls.extend(body["data"])
        cursor = body["pagination"]["next_cursor"]
        if not cursor:
            break

    return calls

calls = list_all_calls(7)
print(f"{len(calls)} çağrı")
```

</TabItem>
</Tabs>

### Squad'a göre filtreleme

```bash
curl -X POST https://api-vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"squad_id":"f47ac10b-58cc-4372-a567-0e02b2c3d479","limit":50}'
```

### Tarih aralığı — tek gün

```bash
curl -X POST https://api-vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "assistant_id": 7,
    "from_date": "2026-05-23",
    "to_date": "2026-05-23"
  }'
```

Yalnızca tarih içeren `to_date`, ertesi UTC gününün başlangıcına genişler; böylece **23 Mayıs gününün tamamı** kapsama dâhil olur. Bkz. [tarih kolaylığı](filtering-pagination.md#range-semantics).

### Tarih aralığı — Türkiye mesai saatleri

```bash
curl -X POST https://api-vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "assistant_id": 7,
    "from_date": "2026-05-23T09:00:00+03:00",
    "to_date": "2026-05-23T17:00:00+03:00"
  }'
```

Düzenli senkronizasyon örnekleri için [artımlı senkronizasyon rehberine](../../guides/incremental-sync.md) bakabilirsiniz.
