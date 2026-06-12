---
title: Webhook'lar
sidebar_label: Webhooks
sidebar_position: 9
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Webhook'lar (Olay Teslimatı)

Vindy, hesabınız için tanımlı webhook URL'sine **HTTP POST** ile olaylar gönderir; böylece sürekli [`POST /v1/calls/list`](list-calls/index.md) ile sorgulamak yerine neredeyse anlık olarak tepki verebilirsiniz. **İki olay tipi** vardır:

| `event_type` | Ne zaman tetiklenir | `data` nedir |
|---|---|---|
| [`call-ended`](#call-ended) | Bir çağrı **sona erdiğinde** (tamamlandı, başarısız oldu veya iptal edildi). | Tam çağrı nesnesi. |
| [`batch-ended`](#batch-ended) | Bir **toplu arama** bittiğinde — içindeki her çağrı sonlanmış bir duruma ulaştığında. | Durum bazında dökümü olan bir toplu arama özeti. |

İkisi de aynı teslimat semantiğini paylaşır (yeniden denemeler, en az bir kez teslimat, header doğrulaması) — bkz. [Davranış](#behavior).

---

## Kurulum

:::info Webhook kurulumu henüz self-servis değil
Webhook endpoint'leri (URL'niz ve varsa özel header'larınız) şu anda **Vindy ekibi tarafından** tanımlanır. Panelden self-servis yönetim yol haritasındadır. Webhook'ları etkinleştirmek için Vindy ekibiyle iletişime geçin ve şunları iletin:

- **URL** — `https://` olmalıdır (düz `http` kabul edilmez).
- **Özel header'lar (opsiyonel)** — isteği kendi tarafınızda doğrulamak için kullanılır. Örneğin `{"x-api-key": "<sizin-secret>"}` veya `{"Authorization": "Bearer <sizin-token>"}`. Vindy bu header'ları her teslimatta aynen gönderir.
:::

## `call-ended` olayı {#call-ended}

Vindy, JSON gövdeli bir HTTP `POST` gönderir. Gövde, `data`'yı saran bir **üst düzey nesnedir** (`event_type`, `delivery_id`, `call_id`). `data`, **tam çağrı nesnesidir** — [`GET /v1/calls/:callId`](get-call.md) ve [`POST /v1/calls/list`](list-calls/index.md) içindeki her öğeyle birebir aynı yapı.

```http
POST <sizin-webhook-url>
Content-Type: application/json
User-Agent: Vindy-Webhooks/1.0
<sizin özel header'larınız, örn. x-api-key: ...>
```

```json
{
  "event_type": "call-ended",
  "delivery_id": "0190aa00-1c5a-7000-8000-abc123def456",
  "call_id": 12345,
  "data": {
    "call_id": 12345,
    "call_status": "completed",
    "call_assistant_id": 7,
    "call_squad_id": null,
    "call_phone_number": "+905551112233",
    "call_bound_type": "outbound",
    "call_started_at": "2026-06-08T10:30:00.000Z",
    "call_ended_at": "2026-06-08T10:31:27.000Z",
    "call_created_at": "2026-06-08T10:29:55.000Z",
    "call_duration_seconds": 87,
    "call_end_reason": "customer-ended-call",
    "call_transcript": "AI: Merhaba, ben yapay zeka asistanı Vindy. Müşteri memnuniyeti anketimiz kapsamında size birkaç kısa soru sormak istiyorum — şu an uygun musunuz?\nUser: Evet, müsaitim.\nAI: Teşekkürler. Öncelikle yaşınızı öğrenebilir miyim?\nUser: Otuz iki.\n",
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
      "url": "https://your-bucket.s3.eu-central-1.amazonaws.com/call-records/...wav?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Expires=86400&X-Amz-Signature=...",
      "expires_at": "2026-06-09T10:31:30.000Z"
    }
  }
}
```

`data.call_transcript` tek bir metin dizesidir; içindeki her konuşma sırası bir satır sonu (`\n`) ile ayrılır. JSON satır sonlarını kaçışlı yazdığı için yukarıdaki değer tek satırda görünür. Gerçek satır sonlarıyla görüntülendiğinde yukarıdaki transcript şöyledir:

```text
AI: Merhaba, ben yapay zeka asistanı Vindy. Müşteri memnuniyeti anketimiz kapsamında size birkaç kısa soru sormak istiyorum — şu an uygun musunuz?
User: Evet, müsaitim.
AI: Teşekkürler. Öncelikle yaşınızı öğrenebilir miyim?
User: Otuz iki.
```

### Üst düzey alanlar

| Alan | Tür | Açıklama |
|---|---|---|
| `event_type` | string | Bu olay için `call-ended`. |
| `delivery_id` | string (UUID) | Bu teslimatın kalıcı kimliği. Aynı olayın her yeniden deneme adımında değişmez; bu nedenle tekrarları `call_id` ile olduğu kadar bu değerle de ayıklayabilirsiniz. |
| `call_id` | int | Çağrının kalıcı kimliği — API genelinde kullanılan aynı kimlik (ör. [`GET /v1/calls/:callId`](get-call.md)). `data`'yı ayrıştırmadan tekrarları ayıklayıp olayı yönlendirebilmeniz için üst seviyede de yer alır. |
| `data` | object | Tam çağrı nesnesi — tüm alanlar aşağıda. |

### `data` — çağrı nesnesi

`data`, [`GET /v1/calls/:callId`](get-call.md)'in döndürdüğü nesnenin aynısıdır:

| Alan | Tür | Açıklama |
|---|---|---|
| `call_id` | int | Kalıcı çağrı kimliği (üst düzeydeki `call_id` ile aynı değer). |
| `call_status` | string | `completed` \| `failed` \| `cancelled` — sonuç, panelde göründüğü haliyle. |
| `call_assistant_id` | int \| null | Çağrıyı yürüten asistan (squad çağrılarında `null`). |
| `call_squad_id` | UUID \| null | Çağrı bir squad üzerinden yapıldıysa squad kimliği (aksi halde `null`). |
| `call_phone_number` | string | Aranan veya arayan telefon numarası (mümkün olduğunda E.164 formatında). |
| `call_bound_type` | string \| null | `inbound` \| `outbound` \| `null`. |
| `call_started_at` | ISO 8601 (UTC) \| null | Çağrının fiilen başladığı an — **UTC** cinsinden, `Z` ekli ve milisaniye hassasiyetli bir ISO-8601 zaman damgası (örn. `2026-05-15T10:30:00.000Z`). Görüntülemek için kendi yerel saat diliminize çevirin. Çağrı hiç bağlanmadıysa `null`. |
| `call_ended_at` | ISO 8601 (UTC) \| null | Çağrının sona erdiği an, aynı ISO-8601 UTC formatında. Çağrı hiç bağlanmadıysa `null`. |
| `call_created_at` | ISO 8601 (UTC) | Çağrı kaydının sistemimizde oluşturulduğu an, aynı ISO-8601 UTC formatında. |
| `call_duration_seconds` | int \| null | Saniye cinsinden çağrı süresi. |
| `call_end_reason` | string \| null | Çağrının sona erme teknik nedeni — geniş bir enum, bkz. [Bitiş nedenleri](list-calls/index.md#end-reasons). Bilinmeyen değerlerde hata vermeyin. |
| `call_transcript` | string \| null | Düz metin transcript. Her konuşma sırası `AI:` (asistan) veya `User:` (arayan) ile başlar, sıralar `\n` ile ayrılır. Çok kısa veya başarısız çağrılarda boş ya da `null` olabilir. |
| `call_structured_data` | object \| null | AI tarafından çıkarılan veri; çağrıya uygulanan her yapısal çıktının `id`'siyle ([`GET /v1/assistants`](list-assistants.md)'ten) anahtarlanır; her değer o çıktının `name`'ini ve çıkarılan `result`'ını içerir. Üretilmediyse `null` — bkz. [Yapısal veri şekilleri](list-calls/index.md#yapısal-veri-şekilleri). |
| `call_metadata` | object \| null | [`POST /v1/calls/bulk`](bulk-create-calls.md) ile gönderdiğiniz opak metadata; korelasyon için aynen geri döner. Çağrı metadata ile oluşturulmadıysa `null`. |
| `call_recording` | object | Kayıt durumu + URL — alanlar aşağıda. |

**`data.call_recording`**

| Alan | Tür | Açıklama |
|---|---|---|
| `available` | bool | Bu çağrı için indirilebilir bir ses kaydının mevcut olup olmadığı. |
| `url` | string \| yok | 24 saatlik presigned indirme URL'si. **Yalnızca** `available: true` iken bulunur. |
| `expires_at` | ISO string \| yok | URL'nin geçerliliğini yitireceği an (UTC). **Yalnızca** `available: true` iken bulunur. |

:::warning Ses kaydı URL'si gönderim anında üretilir
`data.call_recording.url`, **webhook'un gönderildiği andan itibaren** 24 saat geçerlidir. Olayı geç işlerseniz — ya da bir yeniden denemede geldiyse — URL'nin süresi dolmuş olabilir; güncelini [`GET /v1/calls/:callId`](get-call.md) ile alın. `available` `false` ise çağrı için kalıcı bir kayıt yoktur — [ne anlama geldiğine](list-calls/index.md#recording-not-available) bakın.
:::

## `batch-ended` olayı {#batch-ended}

[`POST /v1/calls/bulk`](bulk-create-calls.md) ile oluşturulan bir toplu arama `completed` durumuna ulaştığında, yani **içindeki her çağrı sonlanmış bir duruma ulaştığında**, **bir kez** tetiklenir. Bir toplu aramadaki tüm çağrıların bittiğini bu sayede anlarsınız; sonuç için `counts` dökümünü kullanın, ardından çağrıları [`POST /v1/calls/batches/:batchId/calls`](get-batch-calls.md) ile çekin.

:::caution İptalde gönderilmez
Bir toplu arama iptal edildiğinde `batch-ended` olayı **gönderilmez**. Bir toplu aramayı iptal etmek, iptal edilen her çağrı için zaten ayrı bir `call-ended` üretir; bu nedenle çağrı bazındaki sonuçlar yine bu olaylarla size ulaşır.
:::

Üst düzey nesne, `call-ended`'den farklıdır: üst seviyede `call_id` **değil** `batch_call_id` taşır ve `data`, bir çağrı nesnesi değil bir **toplu arama özetidir**.

```json
{
  "event_type": "batch-ended",
  "delivery_id": "0a61f9bd-...",
  "batch_call_id": 842,
  "data": {
    "batch_call_id": 842,
    "status": "completed",
    "total_count": 200,
    "counts": {
      "completed": 180,
      "failed": 12,
      "cancelled": 8,
      "pending": 0,
      "processing": 0
    },
    "created_at": "2026-06-09T23:39:20.298Z"
  }
}
```

### Üst düzey alanlar

| Alan | Tür | Açıklama |
|---|---|---|
| `event_type` | string | Bu olay için `batch-ended`. |
| `delivery_id` | string (UUID) | Bu teslimatın kalıcı kimliği — her yeniden deneme adımında değişmez. Tekrarları bu değerle veya `batch_call_id` ile ayıklayın. |
| `batch_call_id` | int | Toplu aramanın kimliği — kolaylık için üst seviyede de tekrarlanır. Tekrarları ayıklamak ve toplu aramanın çağrılarını [`POST /v1/calls/batches/:batchId/calls`](get-batch-calls.md) ile çekmek için kullanın. |
| `data` | object | Toplu arama özeti — alanlar aşağıda. |

### `data` — toplu arama özeti

| Alan | Tür | Açıklama |
|---|---|---|
| `batch_call_id` | int | Toplu aramanın kimliği (üst düzeydeki `batch_call_id` ile aynı değer). |
| `status` | string | Toplu aramanın nihai durumu — bu olay için `completed`. |
| `total_count` | int | Toplu aramadaki toplam çağrı sayısı. |
| `counts` | object | Toplu aramanın çağrılarının durum bazında dökümü — toplamı `total_count`'a eşittir. |
| `created_at` | ISO string | Toplu aramanın oluşturulduğu an. |

**`data.counts`**

| Alan | Tür | Açıklama |
|---|---|---|
| `completed` | int | Başarıyla tamamlanan çağrılar. |
| `failed` | int | Başarısız biten çağrılar. |
| `cancelled` | int | İptal edilen çağrılar. |
| `pending` | int | Henüz başlamamış çağrılar. Tamamlanmış bir toplu aramada `0`. |
| `processing` | int | Hâlâ devam eden çağrılar. Tamamlanmış bir toplu aramada `0`. |

:::note `call-ended` ile aynı teslimat semantiği
`batch-ended`, tıpkı `call-ended` gibi teslim edilir — ~15 saniye içinde `2xx` dönün, artan beklemelerle yeniden denenir, en az bir kez teslim edilir (`delivery_id` veya `batch_call_id` ile tekrarları ayıklayın), sıra garantisi yoktur ve aynı özel header'lar/doğrulama kullanılır. Bkz. [Davranış](#behavior).
:::

## Davranış {#behavior}

- **`2xx` dönün** — yaklaşık 15 saniye içinde. `2xx` dışı bir yanıt veya zaman aşımı, başarısız teslimat olarak değerlendirilir ve Vindy yeniden dener.
- **Yeniden denemeler** — başarısız teslimatlar, artan beklemelerle (yaklaşık `30sn → 2dk → 10dk → 1sa`) ve toplamda yaklaşık 5 denemeyle (≈1 saat içinde) tekrarlanır; ardından teslimat başarısız olarak işaretlenir.
- **En az bir kez teslimat (at-least-once)** — olumsuz ağ koşullarında aynı olay birden çok kez gelebilir. **Tekrarları ayıklayın:** `call-ended` için `call_id` (veya `delivery_id`), `batch-ended` için `batch_call_id` (veya `delivery_id`) üzerinden.
- **Sıra garantisi yok** — olaylar, çağrıların sona erme sırasından farklı bir sırada gelebilir.
- **Ses kaydı bağlantısının güncelliği** — `data.call_recording.url`, gönderim anında üretilmiş 24 saatlik bir bağlantıdır. Olayı geç işlerseniz bağlantının süresi dolmuş olabilir; güncelini [`GET /v1/calls/:callId`](get-call.md) ile alın.
- **PII** — payload telefon numarası ve transcript içerebilir; bu nedenle endpoint'iniz `https` olmalıdır.

:::tip Hızlı onaylayın, sonra işleyin
Olayı güvenli biçimde kaydeder kaydetmez `2xx` dönün; ardından ağır işleri (ses kaydı indirme, sistemlerinizi güncelleme) eşzamansız (asenkron) olarak yapın. Bu, ~15 saniyelik pencerede kalmanızı sağlar ve gereksiz yeniden denemeleri önler.
:::

## Doğruluğun denetlenmesi

Bir isteğin Vindy'den geldiğini, **tanımladığınız özel header'ı denetleyerek** doğrularsınız — örneğin beklenen `x-api-key` değerinin geldiğini teyit ederek. Vindy bu sürümde HMAC imzası göndermez; kimlik doğrulama tamamen kurulum sırasında tanımladığınız header'lar üzerinden yapılır.

## Bir teslimatı işleme

Sağlam bir işleyici, hızlıca onay verir, `call_id` ile tekrarları ayıklar ve gerektiğinde güncel bir ses kaydı bağlantısı çeker.

<Tabs groupId="lang">
<TabItem value="node" label="Node.js (Express)">

```javascript
import express from "express";

const app = express();
const seen = new Set(); // üretimde bunu bir DB / unique constraint ile destekleyin

app.post("/vindy/webhook", express.json(), async (req, res) => {
  // 1. İsteğin Vindy'den geldiğini doğrulayın (tanımladığınız özel header)
  if (req.get("x-api-key") !== process.env.VINDY_WEBHOOK_SECRET) {
    return res.sendStatus(401);
  }

  const { call_id, data } = req.body;

  // 2. Tekrarları ayıklayın (en az bir kez teslimat)
  if (seen.has(call_id)) return res.sendStatus(200);
  seen.add(call_id);

  // 3. Hızlı onaylayın, sonra eşzamansız işleyin
  res.sendStatus(200);

  // 4. Ses kaydı bağlantısının süresi dolmuş olabilir — gerekirse güncel çağrıyı çekin
  void processCall(call_id, data);
});

app.listen(3000);
```

</TabItem>
<TabItem value="python" label="Python (Flask)">

```python
import os
from flask import Flask, request, abort

app = Flask(__name__)
seen = set()  # üretimde bunu bir DB / unique constraint ile destekleyin

@app.post("/vindy/webhook")
def vindy_webhook():
    # 1. İsteğin Vindy'den geldiğini doğrulayın (tanımladığınız özel header)
    if request.headers.get("x-api-key") != os.environ["VINDY_WEBHOOK_SECRET"]:
        abort(401)

    body = request.get_json()
    call_id = body["call_id"]

    # 2. Tekrarları ayıklayın (en az bir kez teslimat)
    if call_id in seen:
        return "", 200
    seen.add(call_id)

    # 3. Eşzamansız işleme için kuyruğa alın, sonra hızlı onaylayın
    enqueue_processing(call_id, body["data"])
    return "", 200
```

</TabItem>
</Tabs>

:::note İlgili
Webhook'lar [`POST /v1/calls/list`](list-calls/index.md) endpoint'ini tamamlar, ancak onun yerini almaz. Sorgulamaya (polling) dayalı bir mutabakat deseni için [Artımlı Senkronizasyon kılavuzuna](../guides/incremental-sync.md) bakabilirsiniz.
:::
