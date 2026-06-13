---
title: Toplu Çağrıyı İptal Et
sidebar_label: Toplu Çağrıyı İptal Et
sidebar_position: 7
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `POST /v1/calls/batches/:batchId/cancel`

[`POST /v1/calls/bulk`](bulk-create-calls.md) ile oluşturulan bir batch'teki tüm **bekleyen** çağrıları iptal eder. Yalnızca henüz aranmamış çağrılar iptal edilir; halihazırda aranmakta olan veya bitmiş çağrılara dokunulmaz.

`batchId`, bulk yanıtında dönen `batch_call_id` değeridir.

---

## İstek

```http
POST https://api-vindy.vinter.me/v1/calls/batches/842/cancel
Authorization: Bearer <api-key>
```

İstek gövdesi yoktur.

## Yol parametreleri

| Parametre | Tür | Açıklama |
|---|---|---|
| `batchId` | int | Batch'in sayısal kimliği — [`POST /v1/calls/bulk`](bulk-create-calls.md) yanıtındaki `batch_call_id`. |

## Yanıt (200 OK)

```json
{ "batch_call_id": 842, "cancelled_pending_count": 37 }
```

| Alan | Tür | Açıklama |
|---|---|---|
| `batch_call_id` | int | İptal edilen batch'in kimliği (istekte gönderdiğiniz `batchId`). |
| `cancelled_pending_count` | int | Bu istekle iptal edilen bekleyen çağrı sayısı. |

## Hatalar

| Durum | Kod | Açıklama |
|---|---|---|
| `400` | `VALIDATION_FAILED` | `batchId` pozitif bir tam sayı değil. |
| `400` | `ERR_BATCH_NOT_FOUND` | Batch bulunamadı veya sizin şirketinize ait değil. |
| `400` | `ERR_BATCH_ALREADY_FINAL` | Batch artık çalışmıyor (zaten bitmiş veya iptal edilmiş). |
| `400` | `ERR_BATCH_NO_PENDING_CALLS` | Batch hâlâ aktif, ancak iptal edilecek bekleyen çağrı kalmamış. |
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` | Kimlik doğrulama hataları. |

:::note Yalnızca bekleyen çağrılar etkilenir
Bu endpoint, henüz başlamamış çağrıları durdurur. Halihazırda devam eden çağrılar tamamlanana kadar sürer, bitmiş çağrılar değişmez. Dönen `cancelled_pending_count`, tam olarak kaç çağrının durdurulduğunu belirtir.
:::

## Örnekler

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
curl -X POST https://api-vindy.vinter.me/v1/calls/batches/842/cancel \
  -H "Authorization: Bearer $VINDY_API_KEY"
# → { "batch_call_id": 842, "cancelled_pending_count": 37 }
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
async function cancelBatch(batchId) {
  const response = await fetch(
    `https://api-vindy.vinter.me/v1/calls/batches/${batchId}/cancel`,
    {
      method: "POST",
      headers: { Authorization: `Bearer ${process.env.VINDY_API_KEY}` },
    },
  );

  if (!response.ok) {
    const error = await response.json();
    // ERR_BATCH_ALREADY_FINAL / ERR_BATCH_NO_PENDING_CALLS — iptal edilecek bir şey yok
    throw new Error(`${error.code}: ${error.message}`);
  }

  const { cancelled_pending_count } = await response.json();
  console.log(`${cancelled_pending_count} bekleyen çağrı iptal edildi`);
  return cancelled_pending_count;
}

await cancelBatch(842);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

def cancel_batch(batch_call_id):
    response = requests.post(
        f"https://api-vindy.vinter.me/v1/calls/batches/{batch_call_id}/cancel",
        headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
    )

    if not response.ok:
        error = response.json()
        # ERR_BATCH_ALREADY_FINAL / ERR_BATCH_NO_PENDING_CALLS — iptal edilecek bir şey yok
        raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

    count = response.json()["cancelled_pending_count"]
    print(f"{count} bekleyen çağrı iptal edildi")
    return count

cancel_batch(842)
```

</TabItem>
</Tabs>
