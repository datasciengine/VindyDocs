---
title: Tek Bir Çağrıyı Getir
sidebar_label: Tek Bir Çağrıyı Getir
sidebar_position: 5
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `GET /v1/calls/:callId`

Tek bir çağrıyı kalıcı `call_id` değeriyle döndürür. Yanıt, [`POST /v1/calls/list`](list-calls/index.md) içindeki bir çağrı nesnesiyle **birebir aynıdır** — transcript, yapısal veri, metadata ve (hazırsa) güncel 24 saatlik bir ses kaydı bağlantısı dahil.

Elinizde bir `call_id` olduğunda — [Çağrıları Listele](list-calls/index.md), bir [webhook](webhooks.md) ya da kendi kayıtlarınızdan — çağrıyı talep anında çekmek için kullanın. Webhook'tan sonra tam nesne zaten elinizdedir; bu endpoint'i sonradan çağırmanın asıl nedeni, süresi dolmuş bir ses kaydı bağlantısını tazelemektir.

:::info Çağrıları Listele ile aynı görünürlük kuralı
Yalnızca **müşteriye gösterilebilir** çağrılar (durum `completed`, `failed` veya `cancelled`) döner. Devam eden / yarım çağrılar `404` döndürür — bu, [`POST /v1/calls/list`](list-calls/index.md) ile tamamen aynı görünürlük kuralıdır.
:::

---

## İstek

```http
GET https://api-vindy.vinter.me/v1/calls/12345
Authorization: Bearer <api-key>
```

## Yol parametreleri

| Parametre | Tür | Açıklama |
|---|---|---|
| `callId` | int | Çağrının sayısal kimliği ([`POST /v1/calls/list`](list-calls/index.md) yanıtından veya bir [webhook olayından](webhooks.md) alınır). |

## Yanıt (200 OK)

[`POST /v1/calls/list`](list-calls/index.md) yanıtındaki `data[]` öğesiyle aynı yapı:

```json
{
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
    "url": "https://...?X-Amz-...",
    "expires_at": "2026-06-09T10:31:30.000Z"
  }
}
```

## Yanıt alanları

Çağrı nesnesi, bir [Çağrıları Listele](list-calls/index.md#yanıt-alanları) öğesiyle **aynı alanlara** sahiptir. Burada özellikle belirtilmesi gereken alan:

| Alan | Tür | Açıklama |
|---|---|---|
| `call_metadata` | object \| null | [`POST /v1/calls/bulk`](bulk-create-calls.md) ile gönderdiğiniz metadata; aynen geri döner. Çağrı metadata ile oluşturulmadıysa `null` olur. |

Diğer tüm alanlar — `call_status`, `call_transcript`, `call_structured_data`, `call_recording`, `call_end_reason` enum'u ve `call_recording.available: false` ne anlama geldiği — için tam [Çağrıları Listele alan referansına](list-calls/index.md#yanıt-alanları) bakabilirsiniz.

:::tip Güncel ses kaydı bağlantısı
Buradaki `call_recording.url`, **istek anında** üretilir ve 24 saat geçerlidir. Bir webhook payload'undaki ses kaydı bağlantısının süresi siz işlerken dolmuşsa, güncelini almak için bu endpoint'i çağırın.
:::

## Hatalar

| Durum | Kod | Açıklama |
|---|---|---|
| `400` | `VALIDATION_FAILED` | `callId` pozitif bir tam sayı değil. |
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` | Kimlik doğrulama hataları. |
| `404` | `RESOURCE_NOT_FOUND` | Çağrı bulunamadı, henüz erişilebilir değil (hâlâ devam ediyor) veya başka bir şirkete ait. |

:::note Varlık bilgisi sızdırılmaz
Başka bir şirkete ait bir `call_id`, var olmayan bir kimlikle aynı `404 RESOURCE_NOT_FOUND` yanıtını döndürür — bkz. [Çoklu kiracılık](../concepts/multi-tenancy.md).
:::

## Örnekler

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
curl -H "Authorization: Bearer $VINDY_API_KEY" \
  https://api-vindy.vinter.me/v1/calls/12345
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
async function getCall(callId) {
  const response = await fetch(
    `https://api-vindy.vinter.me/v1/calls/${callId}`,
    { headers: { Authorization: `Bearer ${process.env.VINDY_API_KEY}` } },
  );

  if (response.status === 404) {
    return null; // bulunamadı, henüz hazır değil veya sizin şirketinizde değil
  }
  if (!response.ok) {
    const error = await response.json();
    throw new Error(`${error.code}: ${error.message}`);
  }

  return response.json();
}

const call = await getCall(12345);
console.log(call?.call_status, call?.call_metadata);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

def get_call(call_id):
    response = requests.get(
        f"https://api-vindy.vinter.me/v1/calls/{call_id}",
        headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
    )

    if response.status_code == 404:
        return None  # bulunamadı, henüz hazır değil veya sizin şirketinizde değil
    if not response.ok:
        error = response.json()
        raise RuntimeError(f"{error.get('code')}: {error.get('message')}")
    return response.json()

call = get_call(12345)
if call:
    print(call["call_status"], call.get("call_metadata"))
```

</TabItem>
</Tabs>
