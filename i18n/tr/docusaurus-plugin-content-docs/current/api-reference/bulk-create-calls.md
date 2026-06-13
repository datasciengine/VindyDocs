---
title: Toplu Çağrı Oluştur
sidebar_label: Toplu Çağrı Oluştur
sidebar_position: 3
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `POST /v1/calls/bulk`

Verdiğiniz telefon numaralarına, bir asistan **veya** bir squad kullanarak toplu giden çağrı oluşturur (tek istekte 1–200 numara).

Her çağrıya isteğe bağlı bir `metadata` nesnesi ekleyebilirsiniz: Vindy bu veriyi işlemez ve [`POST /v1/calls/list`](list-calls/index.md), [`GET /v1/calls/:callId`](get-call.md) ile [webhook olaylarındaki](webhooks.md) her çağrı nesnesinde **olduğu gibi geri döndürür**. Böylece bir çağrıyı kendi sisteminizdeki bir kayıtla (CRM kişisi, sipariş, destek kaydı) ilişkilendirebilirsiniz. Ayrıntılar ve limitler için bkz. [Metadata](#metadata).

Çağrıların yapılacağı numara, hesabınızda tanımlı uygun bir numaradan **otomatik olarak seçilir** — istekte belirtmenize gerek yoktur.

:::info Aynı anda tek batch
Hesabınızda aynı anda yalnızca **bir toplu arama** devam edebilir. Bir batch çalışırken yeni bir istek göndermek `409 BATCH_IN_PROGRESS` döndürür.
:::

---

## İstek

```http
POST https://api-vindy.vinter.me/v1/calls/bulk
Authorization: Bearer <api-key>
Content-Type: application/json

{
  "assistant_id": 7,
  "calls": [
    { "phone_number": "+905551112233", "metadata": { "crm_contact_id": "CNT-90412" } },
    { "phone_number": "+905554445566", "metadata": { "crm_contact_id": "CNT-90413" } }
  ]
}
```

Squad ile aramak için `assistant_id` yerine `squad_id` gönderebilirsiniz.

## Gövde parametreleri

| Alan | Tür | Zorunlu | Açıklama |
|---|---|---|---|
| `assistant_id` | int | **biri zorunlu** | Çağrıları yapacak asistan. `squad_id` ile birlikte kullanılamaz. |
| `squad_id` | UUID | **biri zorunlu** | Çağrıları yapacak squad. `assistant_id` ile birlikte kullanılamaz. |
| `calls` | array | evet | Aranacak hedefler (1–200). |
| `calls[].phone_number` | string | evet | Aranacak Türkiye numarası. Bkz. aşağıdaki [Telefon numaraları](#phone-numbers). |
| `calls[].metadata` | object | hayır | İsteğe bağlı anahtar-değer nesnesi (bkz. [Metadata](#metadata) limitleri). Aynen geri döner. |

:::caution
`assistant_id` ve `squad_id` parametrelerinden **yalnızca birini** gönderin. İkisini birden ya da hiçbirini göndermek bir doğrulama hatasıdır (`VALIDATION_FAILED`).
:::

### Telefon numaraları {#phone-numbers}

Yalnızca **Türkiye** numaraları desteklenir. Aşağıdaki biçimlerin tümü kabul edilir ve `+905551112233` biçimine normalize edilir:

| Gönderdiğiniz | Normalize edilir |
|---|---|
| `+905551112233` | `+905551112233` |
| `905551112233` | `+905551112233` |
| `05551112233` | `+905551112233` |
| `5551112233` | `+905551112233` |

Yaygın ayraçlar — boşluk, tire ve parantez — tolere edilip temizlenir; bu nedenle `+90 555 111 22 33` ve `0555-111-2233` gibi numaralar da kabul edilir.

`+90` dışında bir ülke kodu (`+1`, `+44`, …) ile gönderilen numaralar **reddedilir** — bu sürümde uluslararası arama desteklenmez. Kabul edilen numaralar normalize edilmiş `+90…` biçiminde saklanır ve aranır; bu değeri daha sonra liste, tekil çağrı ve webhook yanıtlarında `call_phone_number` olarak görürsünüz.

### Metadata {#metadata}

`metadata`, tamamen **size ait** olan serbest biçimli bir anahtar-değer nesnesidir. Vindy onu opak bir veri olarak ele alır: içeriğini **asla okumaz, ayrıştırmaz, doğrulamaz veya ona göre bir işlem yapmaz** ve bir çağrının nasıl başlatıldığına, yönlendirildiğine veya işlendiğine **hiçbir etkisi yoktur**. Onu yalnızca saklar ve o çağrının her görünümünde size olduğu gibi geri döndürürüz — [`POST /v1/calls/list`](list-calls/index.md), [`GET /v1/calls/:callId`](get-call.md) ve [webhook olaylarında](webhooks.md).

Tek görevi **sizin tarafınızda eşleştirmedir**. Bir çağrıyı kendi verinizle ilişkilendirmek için sistemlerinizin ihtiyaç duyduğu kimlikleri ekleyin — bir CRM kişi kimliği, sipariş numarası, kampanya etiketi, kendi istek kimliğiniz vb. Sonuç geri geldiğinde aynı anahtarları `call_metadata` içinden okur ve sonucu doğrudan kendi CRM'inize, veritabanınıza veya iş akışınıza yönlendirirsiniz — ayrıca bir telefon-numarası–kayıt eşleme tablosu tutmanıza gerek kalmaz.

Tek kısıtlama yapısaldır; böylece veriyi güvenilir biçimde saklayıp geri döndürebiliriz:

| Limit | Değer |
|---|---|
| En fazla anahtar | 50 |
| En fazla anahtar uzunluğu | 40 |
| En fazla değer uzunluğu | 500 |
| Değer tipleri | `string`, `number`, `boolean` |
| İç içe nesne / dizi | İzin verilmez |

:::caution Kendi anahtarlarınız için kullanın — ve PII koymayın
Vindy `metadata`'yı asla yorumlamadığı için, burası **sizin** eşleştirme anahtarlarınızın doğru yeridir (örneğin `crm_contact_id`, `orderId`, `campaign`). Kişisel veriler (ad, telefon, kimlik no) için **uygun değildir** — onları kendi sistemlerinizde saklayın ve buradan yalnızca anahtarla referans verin.
:::

## Yanıt (201 Created)

```json
{
  "batch_call_id": 842,
  "accepted": 200
}
```

| Alan | Tür | Açıklama |
|---|---|---|
| `batch_call_id` | int | Oluşturulan toplu aramanın kimliği. Batch'i daha sonra [`POST /v1/calls/batches/:batchId/cancel`](cancel-batch.md) ile iptal etmek için **saklayın**. |
| `accepted` | int | Kuyruğa alınan çağrı sayısı. |

:::info Sonuçları eşleştirme
Tek tek çağrı kimlikleri burada **dönmez** (büyük batch'lerde gereksiz büyük yanıtları önlemek için). Sonuçlar, çağrılar tamamlandıkça [`POST /v1/calls/list`](list-calls/index.md) ve [webhook olaylarıyla](webhooks.md) gelir; her ikisi de gönderdiğiniz `metadata` değerini her çağrı nesnesinde aynen döndürür. Kendi tarafınızda `metadata` anahtarlarınızı saklayıp gelen `call_id` değerleriyle eşleştirin.
:::

Çağrılar kuyruğa alınır ve arka planda yürütülür. Sonuçlar (transcript, ses kaydı, yapısal veri) her çağrı tamamlandıkça erişilebilir hâle gelir.

:::tip Toplu aramanın ilerleyişini takip edin
Bu toplu aramanın çağrılarını tamamlandıkça sayfalamak için [`POST /v1/calls/batches/:batchId/calls`](get-batch-calls.md) endpoint'ini kullanın. **Tüm** toplu aramanın ne zaman bittiğini — durum bazında bir dökümle birlikte — öğrenmek için [`batch-ended` webhook'unu](webhooks.md#batch-ended) dinleyin.
:::

## Hatalar

| Durum | Kod | Açıklama |
|---|---|---|
| `400` | `VALIDATION_FAILED` | `assistant_id` / `squad_id` ikisi birden ya da hiçbiri gönderilmiş, `calls` boş veya 200'den fazla vb. |
| `400` | `INVALID_PHONE_NUMBER` | Bir `calls[i].phone_number` geçerli bir Türkiye numarası değil. Hatalı indeks `extensions.details.index` içindedir. |
| `400` | `INVALID_METADATA` | Bir çağrının metadata'sı limitleri aşıyor veya geçersiz bir değer tipi kullanıyor. Hatalı indeks `extensions.details.index` içindedir. |
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` | Kimlik doğrulama hataları. |
| `404` | `ASSISTANT_NOT_FOUND` | Asistan bulunamadı, sizin şirketinize ait değil veya arama için uygun değil. |
| `404` | `SQUAD_NOT_FOUND` | Squad bulunamadı, sizin şirketinize ait değil veya arama için uygun değil. |
| `409` | `NO_OUTBOUND_PHONE_NUMBER` | Hesabınızda giden arama yapabilecek bir numara tanımlı değil. Vindy ekibiyle iletişime geçin. |
| `409` | `BATCH_IN_PROGRESS` | Devam eden bir toplu arama var. Bitmesini bekleyip tekrar deneyin. |

:::caution Atomik istek
İstekteki **herhangi bir** numara veya metadata geçersizse **hiçbir çağrı oluşturulmaz** — tüm istek reddedilir. Hatalı kaydı düzeltip (bkz. `extensions.details.index`) yeniden gönderin.
:::

## Örnekler

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
curl -X POST https://api-vindy.vinter.me/v1/calls/bulk \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "assistant_id": 7,
    "calls": [
      { "phone_number": "+905551112233", "metadata": { "crm_contact_id": "CNT-90412" } },
      { "phone_number": "05554445566", "metadata": { "crm_contact_id": "CNT-90413" } }
    ]
  }'
# → { "batch_call_id": 842, "accepted": 2 }
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
async function createBulkCalls(assistantId, targets) {
  const response = await fetch("https://api-vindy.vinter.me/v1/calls/bulk", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.VINDY_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ assistant_id: assistantId, calls: targets }),
  });

  if (!response.ok) {
    const error = await response.json();
    // INVALID_PHONE_NUMBER / INVALID_METADATA, extensions.details.index taşır
    throw new Error(`${error.code}: ${error.message}`);
  }

  const { batch_call_id, accepted } = await response.json();
  console.log(`Batch ${batch_call_id}, ${accepted} çağrı kuyruğa alındı`);
  return batch_call_id; // batch'i sonra iptal etmek için saklayın
}

await createBulkCalls(7, [
  { phone_number: "+905551112233", metadata: { crm_contact_id: "CNT-90412" } },
  { phone_number: "05554445566", metadata: { crm_contact_id: "CNT-90413" } },
]);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

def create_bulk_calls(assistant_id, targets):
    response = requests.post(
        "https://api-vindy.vinter.me/v1/calls/bulk",
        headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
        json={"assistant_id": assistant_id, "calls": targets},
    )

    if not response.ok:
        error = response.json()
        # INVALID_PHONE_NUMBER / INVALID_METADATA, extensions.details.index taşır
        raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

    body = response.json()
    print(f"Batch {body['batch_call_id']}, {body['accepted']} çağrı kuyruğa alındı")
    return body["batch_call_id"]  # batch'i sonra iptal etmek için saklayın

create_bulk_calls(7, [
    {"phone_number": "+905551112233", "metadata": {"crm_contact_id": "CNT-90412"}},
    {"phone_number": "05554445566", "metadata": {"crm_contact_id": "CNT-90413"}},
])
```

</TabItem>
</Tabs>

### Squad ile arama

```bash
curl -X POST https://api-vindy.vinter.me/v1/calls/bulk \
  -H "Authorization: Bearer $VINDY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "squad_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "calls": [
      { "phone_number": "+905551112233", "metadata": { "orderId": "ORD-77" } }
    ]
  }'
```
