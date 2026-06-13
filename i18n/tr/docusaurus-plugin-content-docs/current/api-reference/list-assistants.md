---
title: Asistanları Listele
sidebar_label: Asistanları Listele
sidebar_position: 1
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `GET /v1/assistants`

Şirketinizin asistanlarını ve squad'larını tek bir birleşik liste hâlinde döndürür. Her öğe, türünü belirten bir `type` alanı taşır (`"assistant"` veya `"squad"`). Asistanlara bağlı structured output şemaları her öğede yer alır. Squad'lar için structured output'lar, squad'a üye asistanların şemalarının yinelenenlerden arındırılmış birleşimidir.

---

## İstek

```http
GET https://api-vindy.vinter.me/v1/assistants
Authorization: Bearer <api-key>
```

Sorgu parametresi yoktur.

## Yanıt (200 OK)

```json
{
  "data": [
    {
      "type": "assistant",
      "assistant_id": 7,
      "assistant_name": "Customer Support",
      "assistant_language": "tr",
      "assistant_created_at": "2026-05-01T10:30:00.000Z",
      "structured_outputs": [
        {
          "id": "9b1c7e2a-4d3f-4a8b-bc12-5e6f7a8b9c01",
          "name": "Support Ticket",
          "schema": {
            "type": "object",
            "properties": {
              "customer_name": {
                "type": "string"
              },
              "issue_type": {
                "type": "string"
              },
              "resolved": {
                "type": "boolean"
              }
            }
          }
        }
      ]
    },
    {
      "type": "squad",
      "squad_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "squad_name": "Sales Team",
      "squad_created_at": "2026-05-15T14:20:00.000Z",
      "squad_assistants": [
        { "assistant_id": 7, "assistant_name": "Customer Support" },
        { "assistant_id": 8, "assistant_name": "Sales Assistant" }
      ],
      "structured_outputs": [
        {
          "id": "9b1c7e2a-4d3f-4a8b-bc12-5e6f7a8b9c01",
          "name": "Support Ticket",
          "schema": {
            "type": "object",
            "properties": {
              "customer_name": {
                "type": "string"
              }
            }
          }
        }
      ]
    }
  ],
  "total": 2
}
```

## Yanıt alanları

**Üst düzey**

| Alan | Tür | Açıklama |
|---|---|---|
| `data` | array | Asistan ve/veya squad öğelerinden oluşan karışık liste. |
| `total` | int | `data` dizisinin uzunluğu. |

**Asistan öğesi (`type: "assistant"`)**

| Alan | Tür | Açıklama |
|---|---|---|
| `type` | `"assistant"` | Tür ayırt edici (discriminator). |
| `assistant_id` | int | Kalıcı asistan kimliği. [`POST /v1/calls/list`](list-calls/index.md) isteğinde kullanılır. |
| `assistant_name` | string | Görünen ad. |
| `assistant_language` | string | Dil kodu (örneğin `tr`, `en`). |
| `assistant_created_at` | ISO string | Oluşturulma zamanı (UTC). |
| `structured_outputs` | array | Bu asistana bağlı structured output şemaları. |

**Squad öğesi (`type: "squad"`)**

| Alan | Tür | Açıklama |
|---|---|---|
| `type` | `"squad"` | Tür ayırt edici (discriminator). |
| `squad_id` | UUID | Kalıcı squad kimliği. [`POST /v1/calls/list`](list-calls/index.md) isteğinde `squad_id` olarak kullanılır. |
| `squad_name` | string \| null | Görünen ad. |
| `squad_created_at` | ISO string | Oluşturulma zamanı (UTC). |
| `squad_assistants` | array | Üye asistanlar (kısa meta veri: `assistant_id`, `assistant_name`). |
| `structured_outputs` | array | Üye asistanların structured output'larının yinelenenlerden arındırılmış birleşimi. |

**StructuredOutput nesnesi (her iki öğe türünde de aynı yapı)**

| Alan | Tür | Açıklama |
|---|---|---|
| `id` | string (UUID) | Structured output'un kalıcı kimliği. [`POST /v1/calls/list`](list-calls/index.md) yanıtındaki `call_structured_data` nesnesinde anahtar olarak kullanılan değerle aynıdır; böylece bir çağrının çıkarılan verisini şemasıyla eşleştirebilirsiniz. |
| `name` | string | Görünen ad. |
| `schema` | object | JSON Schema. Alan düzeyindeki `description` anahtarları ve `required` dizisi yanıttan çıkarılır. |

## Hatalar

| Durum | Kod |
|---|---|
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` |
| `500` | `HTTP_500` |

## Notlar

- `data` dizisi şu sırayla döner: önce asistanlar (`created_at` artan), ardından squad'lar (`created_at` artan).
- Aynı structured output, birden fazla asistanın ve squad'ın altında görünebilir; aynı `id` değerinin farklı öğelerde yer alması beklenen bir davranıştır.
- Bir structured output'un `id` değeri, [Çağrıları Listele](list-calls/index.md) yanıtındaki `call_structured_data` içinde verisinin göründüğü anahtarla aynıdır; çağrının çıkarılan verisini buradaki şemayla bu sayede eşleştirebilirsiniz.
- Squad'lar birer asistan grubudur; bir squad üzerinden yapılan çağrılar [`POST /v1/calls/list`](list-calls/index.md) endpoint'inde `squad_id` ile filtrelenebilir.
- Bu endpoint sayfalama yapmaz. En fazla 1000 asistan ve 1000 squad döndürür; pratikte şirketlerin çok daha azı olur. Bu sınırı aşmayı bekliyorsanız Vindy ekibiyle iletişime geçin.

## Örnekler

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
curl https://api-vindy.vinter.me/v1/assistants \
  -H "Authorization: Bearer $VINDY_API_KEY"
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
const response = await fetch("https://api-vindy.vinter.me/v1/assistants", {
  headers: { Authorization: `Bearer ${process.env.VINDY_API_KEY}` },
});

if (!response.ok) {
  const error = await response.json();
  throw new Error(`${error.code}: ${error.message}`);
}

const { data, total } = await response.json();
console.log(`${total} öğe`);

for (const item of data) {
  if (item.type === "assistant") {
    console.log(`Asistan #${item.assistant_id}: ${item.assistant_name}`);
  } else {
    console.log(`Squad ${item.squad_id}: ${item.squad_name}`);
  }
}
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

response = requests.get(
    "https://api-vindy.vinter.me/v1/assistants",
    headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
)
if not response.ok:
    error = response.json()
    raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

body = response.json()
print(f"{body['total']} öğe")

for item in body["data"]:
    if item["type"] == "assistant":
        print(f"Asistan #{item['assistant_id']}: {item['assistant_name']}")
    else:
        print(f"Squad {item['squad_id']}: {item['squad_name']}")
```

</TabItem>
</Tabs>
