---
title: Artımlı Senkronizasyon
sidebar_label: Artımlı Senkronizasyon
sidebar_position: 1
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Artımlı Senkronizasyon

Kendi veritabanınızı, her seferinde tüm veriyi yeniden indirmeden Vindy çağrılarıyla güncel tutabilirsiniz.

---

## Tek seferlik aktarım ile sürekli senkronizasyon

- **Tek seferlik dışa aktarım** (geçmişe yönelik bir rapor) için: tüm sayfalarda [cursor](../api-reference/list-calls/filtering-pagination.md#cursors) ile gezinebilirsiniz ve işlemi tamamlayın.
- **Sürekli senkronizasyon** (canlı operasyon) için: `from_date` değerini son senkronizasyon zamanınıza eşitleyin ve yalnızca yeni hazır hâle gelen çağrıları alın.

Önerilen sorgulama (polling) sıklığı: **dakikada birden fazla olmamalıdır.** Daha sık sorgulama genellikle gereksizdir; çağrılar milisaniye aralıklarla değil, gruplar hâlinde hazır hâle gelir.

---

## Yaklaşım

1. Veritabanınızda **`call_id` üzerinde bir benzersizlik (unique) kısıtı** kullanabilirsiniz ve her çağrıyı upsert edin.
2. Kendi tarafınızda bir **`last_synced_at`** zamanı tutun — bunu **her senkronizasyon çalıştırmasını başlattığınız ana** (kendi UTC saatiniz) eşitleyin; çağrı nesnesindeki bir alana **değil** (aşağıdaki nota bakın).
3. Bir sonraki senkronizasyonda `from_date=<last_synced_at>` ile sorgu gönderin. `from_date`, **çağrının size ne zaman sunulduğunu** filtreler; bu yüzden bir önceki çalıştırmanızın başlangıç zamanına dayanmak, o andan sonra hazır hâle gelen her şeyi yakalar.
4. Her sayfayı upsert edin. Cursor'u oturum içinde kullanabilirsiniz ve işiniz bitince bırakın (uzun süre saklamanız önerilmez).

:::info Neden bir saat değeri, `call_created_at` değil?
Liste, **her çağrının size ne zaman sunulduğuna** göre sıralanır ve filtrelenir — bu, çağrı nesnesinde **yer almayan** sunucu tarafı bir andır. `call_created_at`, çağrının *kuyruğa alındığı* andır (daha erken ve sunulma anıyla ilgisiz); bunu watermark olarak kullanmak pencereyi sabitleyebilir ya da büyük örtüşmeleri yeniden taratır. `from_date`'i bir önceki çalıştırmayı **başlattığınız** ana dayamak sunulma anını doğru izler; `call_id` üzerinden upsert, küçük sınır örtüşmesini zararsız kılar. Sunucu saatiniz UTC ile tam senkron değilse güvenlik payı olarak bir-iki dakika çıkarın.
:::

Bu yaklaşımın güvenli olmasının nedenleri:

- `POST /v1/calls/list` hiçbir zaman yarı işlenmiş çağrı döndürmez; listede görünen bir çağrı kesinleşmiştir. Bkz. [yarım veri döndürülmez](../api-reference/list-calls/index.md).
- Cursor sayfaları tek bir gezinme içinde çağrıları tekrarlamaz — sonraki sayfa, önceki sayfanın çağrılarını **yeniden döndürmez**. `call_id` üzerinden upsert yapmanızın tek nedeni, ileride örtüşen bir `from_date` ile yeniden senkronize ettiğinizde (veya başarısız bir isteği tekrarladığınızda) kayıt çiftlenmesini önlemektir.
- `from_date` dâhil, `to_date` hariç olduğundan, ardışık zaman aralıkları ne çakışır ne de boşluk bırakır. Bkz. [aralık anlamı](../api-reference/list-calls/filtering-pagination.md#range-semantics).

---

## Uygulama

<Tabs groupId="lang">
<TabItem value="node" label="Node.js">

```javascript
async function syncCalls(assistantId, lastSyncedAt) {
  // BİR SONRAKİ çalıştırmanın watermark'ı: kendi UTC saatinizi fetch'ten *önce*
  // alın ki çalıştırma sürerken hazır olan çağrılar bir sonraki sefer yakalansın.
  // call_id üzerinden upsert, küçük sınır örtüşmesini zararsız kılar.
  const runStartedAt = new Date().toISOString();
  let cursor = undefined;

  do {
    const response = await fetch("https://api.vindy.vinter.me/v1/calls/list", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.VINDY_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        assistant_id: assistantId,
        from_date: lastSyncedAt, // sunulma watermark'ı — bir önceki çalıştırmanın başlangıç zamanı
        limit: 500,
        cursor,
      }),
    });
    if (!response.ok) {
      const error = await response.json();
      throw new Error(`${error.code}: ${error.message}`);
    }

    const body = await response.json();
    for (const call of body.data) {
      await upsertCall(call); // INSERT ... ON CONFLICT (call_id) DO UPDATE
    }
    cursor = body.pagination.next_cursor;
  } while (cursor);

  return runStartedAt; // bir sonraki çalıştırmanın lastSyncedAt değeri olarak kaydedin
}
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
from datetime import datetime, timezone

import requests

def sync_calls(assistant_id, last_synced_at):
    # BİR SONRAKİ çalıştırmanın watermark'ı: kendi UTC saatinizi fetch'ten *önce*
    # alın ki çalıştırma sürerken hazır olan çağrılar bir sonraki sefer yakalansın.
    # call_id üzerinden upsert, küçük sınır örtüşmesini zararsız kılar.
    run_started_at = datetime.now(timezone.utc).isoformat()
    cursor = None

    while True:
        payload = {
            "assistant_id": assistant_id,
            "from_date": last_synced_at,  # sunulma watermark'ı — bir önceki çalıştırmanın başlangıç zamanı
            "limit": 500,
        }
        if cursor:
            payload["cursor"] = cursor

        response = requests.post(
            "https://api.vindy.vinter.me/v1/calls/list",
            headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
            json=payload,
        )
        if not response.ok:
            error = response.json()
            raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

        body = response.json()
        for call in body["data"]:
            upsert_call(call)  # INSERT ... ON CONFLICT (call_id) DO UPDATE

        cursor = body["pagination"]["next_cursor"]
        if not cursor:
            break

    return run_started_at  # bir sonraki çalıştırmanın last_synced_at değeri olarak kaydedin
```

</TabItem>
</Tabs>

---

## Gün gün örnek

```bash
# 1. gün: tam dolum (backfill)
curl -X POST https://api.vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id":7,"limit":500}'
# has_more: false döndürene kadar tüm sayfalarda gezinin.
# Bu çalıştırmayı BAŞLATTIĞINIZ UTC zamanını last_synced_at olarak kaydedin (çağrıdaki bir alanı değil).

# 2. gün ve sonrası: artımlı
curl -X POST https://api.vindy.vinter.me/v1/calls/list \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"assistant_id":7,"from_date":"<last_synced_at>","limit":500}'
```
