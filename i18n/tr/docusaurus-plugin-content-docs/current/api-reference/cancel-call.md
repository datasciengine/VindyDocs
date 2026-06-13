---
title: Tek Bir Çağrıyı İptal Et
sidebar_label: Tek Bir Çağrıyı İptal Et
sidebar_position: 6
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `POST /v1/calls/:callId/cancel`

**Bekleyen** (henüz aranmamış) tek bir çağrıyı iptal eder. Yalnızca kendi şirketinizin çağrılarını iptal edebilirsiniz.

Bir çağrı yalnızca hâlâ bekleme durumundayken iptal edilebilir. Halihazırda aranmakta olan veya bitmiş çağrılar iptal edilemez.

---

## İstek

```http
POST https://api-vindy.vinter.me/v1/calls/12345/cancel
Authorization: Bearer <api-key>
```

İstek gövdesi yoktur.

## Yol parametreleri

| Parametre | Tür | Açıklama |
|---|---|---|
| `callId` | int | Çağrının sayısal kimliği. |

## Yanıt (200 OK)

```json
{ "id": 12345 }
```

| Alan | Tür | Açıklama |
|---|---|---|
| `id` | int | İptal edilen çağrının kimliği. |

## Hatalar

| Durum | Kod | Açıklama |
|---|---|---|
| `400` | `VALIDATION_FAILED` | `callId` pozitif bir tam sayı değil. |
| `400` | `ERR_CALL_NOT_FOUND` | Çağrı bulunamadı veya sizin şirketinize ait değil. |
| `400` | `ERR_CALL_ALREADY_FINAL` | Çağrı zaten sonlanmış (tamamlandı, başarısız oldu veya iptal edildi). |
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` | Kimlik doğrulama hataları. |
| `409` | `ERR_CALL_PROCESSING_CANCEL` | Çağrı şu anda aranıyor ve artık iptal edilemez. |

:::note İptalin artık mümkün olmadığı durum
Bir çağrı, bekleme durumundan aranma durumuna hızla geçer. `409 ERR_CALL_PROCESSING_CANCEL` alırsanız, çağrı başlamış ve API üzerinden durdurulamaz hâle gelmiş demektir. Çağrı sonlandığında sonucunu [`POST /v1/calls/list`](list-calls/index.md), [`GET /v1/calls/:callId`](get-call.md) veya bir [webhook olayı](webhooks.md) ile görürsünüz.
:::

:::tip Bütün bir batch'i iptal etme
Aynı anda çok sayıda bekleyen çağrıyı — örneğin bir toplu aramadaki kalan tüm çağrıları — iptal etmek için, her çağrıyı tek tek iptal etmek yerine bulk isteğinizden gelen `batch_call_id` ile [`POST /v1/calls/batches/:batchId/cancel`](cancel-batch.md) endpoint'ini kullanın.
:::

## Örnekler

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
curl -X POST https://api-vindy.vinter.me/v1/calls/12345/cancel \
  -H "Authorization: Bearer $VINDY_API_KEY"
# → { "id": 12345 }
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
async function cancelCall(callId) {
  const response = await fetch(
    `https://api-vindy.vinter.me/v1/calls/${callId}/cancel`,
    {
      method: "POST",
      headers: { Authorization: `Bearer ${process.env.VINDY_API_KEY}` },
    },
  );

  if (!response.ok) {
    const error = await response.json();
    if (error.code === "ERR_CALL_PROCESSING_CANCEL") {
      return false; // çok geç — çağrı zaten aranıyor
    }
    throw new Error(`${error.code}: ${error.message}`);
  }

  const { id } = await response.json();
  return id === callId;
}

await cancelCall(12345);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

def cancel_call(call_id):
    response = requests.post(
        f"https://api-vindy.vinter.me/v1/calls/{call_id}/cancel",
        headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
    )

    if not response.ok:
        error = response.json()
        if error.get("code") == "ERR_CALL_PROCESSING_CANCEL":
            return False  # çok geç — çağrı zaten aranıyor
        raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

    return response.json()["id"] == call_id

cancel_call(12345)
```

</TabItem>
</Tabs>
