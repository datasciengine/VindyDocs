---
title: Yanıt Formatı
sidebar_label: Yanıt Formatı
sidebar_position: 1
---

# Yanıt Formatı

Tüm Vindy API yanıtları JSON'dur (`application/json; charset=utf-8`) ve küçük, öngörülebilir bir yapı kümesine uyar. Bir kez öğrendiğinizde her endpoint tanıdık gelir.

Çoğu liste yanıtı bir sayfalama nesnesiyle sarmalanır — bkz. [Filtreleme ve Sayfalama](../api-reference/list-calls/filtering-pagination.md#paginated). Tek istisna, sayfalama nesnesi yerine `{ data, total }` döndüren [`GET /v1/assistants`](../api-reference/list-assistants.md) endpoint'idir.

---

## Hata formatı {#error-envelope}

Tüm hata yanıtları aynı yapıyı paylaşır:

```json
{
  "statusCode": 401,
  "timestamp": "2026-06-03T12:34:56.789Z",
  "path": "/v1/assistants",
  "requestId": "01902f6e-7c5a-7000-8000-abc123",
  "code": "INVALID_API_KEY",
  "message": "API key is invalid, expired, or has been revoked.",
  "extensions": {
    "code": "INVALID_API_KEY",
    "statusCode": 401,
    "timestamp": "2026-06-03T12:34:56.789Z",
    "path": "/v1/assistants",
    "requestId": "01902f6e-7c5a-7000-8000-abc123"
  }
}
```

| Alan | Tür | Açıklama |
|---|---|---|
| `statusCode` | int | HTTP durum kodu. |
| `timestamp` | ISO string | Hatanın oluştuğu UTC zamanı. |
| `path` | string | İsteğin yolu. |
| `requestId` | string | Bu isteğe özgü UUID; hata bildirimlerinizde belirtin. |
| `code` | string | Makine-okunabilir hata kodu — bkz. [Hata Kodları kataloğu](../errors.md). |
| `message` | string | İnsan-okunabilir açıklama. |
| `extensions` | object | Her hatada bulunur. `code`, `statusCode`, `timestamp`, `path` ve `requestId` alanlarını yansıtır; bazı hatalarda ek ayrıntı taşır (aşağıya bakın). |

`extensions` nesnesi her hatada bulunur. Yukarıdaki `code`, `statusCode`, `timestamp`, `path` ve `requestId` alanlarını yansıtır; bazı hatalarda ise ek makine-okunabilir ayrıntı taşır — örneğin recording-url hatalarında (404/409) `extensions.recording_status`, tarih biçimi ve toplu çağrı hatalarında `extensions.details`. Beklenmeyen sunucu hatalarında (`500`) kod yalnızca burada yer alır: `extensions.code: "HTTP_500"` (bu, üst düzey `code` içermeyen tek durumdur).

Bir opsiyonel alan daha gelebilir:

- `validation_errors` — doğrulamanın tam olarak hangi noktada başarısız olduğunu listeleyen bir dizi (bazı 400 hatalarında). Bu alan `extensions` içinde de yer alır.

---

## Request ID başlığı {#request-id}

Her yanıt bir `X-Request-Id` başlığı (UUID) taşır. Bu değeri hata bildirimlerinizde belirtmeniz, sorunun çözümünü hızlandırır.

Kendi `X-Request-Id` başlığınızı gönderirseniz API bu değeri olduğu gibi korur; bu da istekleri kendi log'larınızla ilişkilendirmenizi kolaylaştırır.
