---
title: Multi-tenancy
sidebar_label: Multi-tenancy
sidebar_position: 3
---

# Multi-tenancy

**"Başka bir şirketin verisini görebilir miyim?" sorusunun kısa yanıtı: Hayır. Aynı şekilde başkaları da sizin verinizi göremez.**

Her API anahtarı yalnızca tek bir şirkete bağlıdır. Tüm endpoint'ler otomatik olarak yalnızca o şirketin verisini döndürür; yani yalnızca kendi verinizi görürsünüz:

- Her veritabanı sorgusu, yalnızca sizin şirketinizi kapsayacak biçimde sınırlandırılır.
- Başka bir şirkete ait `call_id` değerini kullandığınızda 404 (`RESOURCE_NOT_FOUND`) yanıtı alırsınız; kaydın var olup olmadığı bilgisi dışarıya sızdırılmaz. Böyle bir çağrının var olup olmadığını dahi anlayamazsınız.
- Bu kontrat, **garanti edilen bir davranıştır** ve test edilmiştir.

---

## Bunun pratikteki anlamı

| Senaryo | Sonuç |
|---|---|
| Kendinize ait bir çağrıyı sorgularsınız | Çağrı verisiyle birlikte `200` |
| Var olmayan bir çağrı kimliğini sorgularsınız | `404 RESOURCE_NOT_FOUND` |
| Başka bir şirkete ait bir çağrı kimliğini sorgularsınız | `404 RESOURCE_NOT_FOUND` — "var olmayan" durumdan ayırt edilemez |

İletmeniz gereken bir tenant parametresi ya da yapmanız gereken bir yapılandırma yoktur. Kapsamı, anahtarın kendisi belirler.

```bash
# Size ait olmayan bir çağrı, var olmayan bir çağrıyla tıpatıp aynı şekilde davranır:
curl -H "Authorization: Bearer $VINDY_API_KEY" \
  https://api.vindy.vinter.me/v1/calls/99999999/recording-url
```

```json
{
  "statusCode": 404,
  "timestamp": "2026-06-03T12:34:56.789Z",
  "path": "/v1/calls/99999999/recording-url",
  "requestId": "01902f6e-...",
  "code": "RESOURCE_NOT_FOUND",
  "message": "Call not found.",
  "extensions": {
    "code": "RESOURCE_NOT_FOUND",
    "statusCode": 404,
    "timestamp": "2026-06-03T12:34:56.789Z",
    "path": "/v1/calls/99999999/recording-url",
    "requestId": "01902f6e-..."
  }
}
```
