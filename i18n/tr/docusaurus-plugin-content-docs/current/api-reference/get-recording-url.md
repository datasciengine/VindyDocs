---
title: Ses Kaydı Bağlantısı Al
sidebar_label: Ses Kaydı Bağlantısı Al
sidebar_position: 4
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# `GET /v1/calls/:callId/recording-url`

Belirli bir çağrının ses kaydı için 24 saat geçerli, geçici ve imzalı (presigned) bir indirme bağlantısı oluşturur. Bağlantı doğrudan depolamaya yönlendirir; imza bağlantının içinde yer aldığından ayrıca kimlik doğrulama gerektirmez.

---

## İstek

```http
GET https://api.vindy.vinter.me/v1/calls/12345/recording-url
Authorization: Bearer <api-key>
```

## Yol parametreleri

| Parametre | Tür | Açıklama |
|---|---|---|
| `callId` | int | Çağrının sayısal kimliği ([`POST /v1/calls/list`](list-calls/index.md) yanıtından alınır). |

## Yanıt (200 OK)

```json
{
  "url": "https://...?X-Amz-Algorithm=...&X-Amz-Signature=...",
  "expires_at": "2026-06-04T12:34:56.789Z"
}
```

| Alan | Tür | Açıklama |
|---|---|---|
| `url` | string | İmzalı indirme bağlantısı. Ses kaydını indirmek için bu adrese doğrudan bir GET isteği gönderin; imza bağlantıya gömülüdür. Bağlantı 24 saat sonra geçerliliğini yitirir, önbelleğe almayın. |
| `expires_at` | ISO string | Bağlantının geçerliliğini yitireceği an (UTC, oluşturulmasından 24 saat sonra). |

## Hatalar

| Durum | Kod | Açıklama |
|---|---|---|
| `400` | `VALIDATION_FAILED` | `callId` pozitif bir tam sayı değil. |
| `401` | `MISSING_AUTH_HEADER`, `INVALID_AUTH_FORMAT`, `INVALID_API_KEY` | Kimlik doğrulama hataları. |
| `404` | `RESOURCE_NOT_FOUND` | Çağrı bulunamadı veya sizin şirketinize ait değil. |
| `404` | `RECORDING_NOT_AVAILABLE` | Çağrı mevcut, ancak bu çağrı için hiç ses kaydı üretilmemiş. **Kalıcı** — yeniden denemek sonucu değiştirmez. `recording_status: "not_found"` taşır. |
| `409` | `RECORDING_NOT_READY` | Ses kaydı var ama henüz indirilebilir değil. `recording_status` değerini (`extensions` içinde) taşır: `failed` (kalıcı) veya `pending`/`processing` (geçici). |

**404 örneği — hiç ses kaydı üretilmemiş (kalıcı):**

```json
{
  "statusCode": 404,
  "timestamp": "2026-06-03T12:34:56.789Z",
  "path": "/v1/calls/12345/recording-url",
  "requestId": "01902f6e-...",
  "code": "RECORDING_NOT_AVAILABLE",
  "message": "No recording was produced for this call. This is permanent — there is nothing to retrieve, and retrying will not help.",
  "extensions": {
    "code": "RECORDING_NOT_AVAILABLE",
    "statusCode": 404,
    "timestamp": "2026-06-03T12:34:56.789Z",
    "path": "/v1/calls/12345/recording-url",
    "requestId": "01902f6e-...",
    "recording_status": "not_found"
  }
}
```

**409 örneği — ses kaydı hâlâ hazırlanıyor (geçici):**

```json
{
  "statusCode": 409,
  "timestamp": "2026-06-03T12:34:56.789Z",
  "path": "/v1/calls/12345/recording-url",
  "requestId": "01902f6e-...",
  "code": "RECORDING_NOT_READY",
  "message": "Recording is not ready yet. The recording transfer is still in progress — retry in a few minutes.",
  "extensions": {
    "code": "RECORDING_NOT_READY",
    "statusCode": 409,
    "timestamp": "2026-06-03T12:34:56.789Z",
    "path": "/v1/calls/12345/recording-url",
    "requestId": "01902f6e-...",
    "recording_status": "processing"
  }
}
```

## `recording_status` değerleri {#recording-status}

Bu bölümdeki her hata bir `recording_status` değeri (`extensions` içinde) taşır. Bu değer, HTTP durumuna ve hata koduna şöyle eşlenir:

| `recording_status` | HTTP | Kod | Anlamı |
|---|---|---|---|
| `pending` | `409` | `RECORDING_NOT_READY` | Geçici (nadir) — ses kaydı sırada bekliyor; birkaç dakika sonra tekrar deneyin. |
| `processing` | `409` | `RECORDING_NOT_READY` | Geçici (nadir) — aktarım sürüyor; birkaç saniye ile dakika içinde tekrar deneyin. |
| `completed` | `200` | — | Başarılı — 200 ve bağlantı alırsınız. (Nadir bir istisna için aşağıdaki nota bakın.) |
| `failed` | `409` | `RECORDING_NOT_READY` | **Kalıcı** — aktarım kalıcı olarak başarısız oldu. **Yeniden denemek sonucu değiştirmez.** Vindy ekibiyle iletişime geçin. |
| `not_found` | `404` | `RECORDING_NOT_AVAILABLE` | **Kalıcı** — bu çağrı için hiç ses kaydı üretilmemiş. **Yeniden denemek sonucu değiştirmez.** |

:::note `completed` → 200 kuralının nadir istisnası
`completed` olarak işaretlenmiş ama dosyası henüz indirilebilir olmayan bir ses kaydı, `recording_status: "completed"` taşıyan bir `409 RECORDING_NOT_READY` döndürür. Bu, geçici bir yarış koşuludur; birkaç dakika sonra tekrar deneyin.
:::

:::info Önemli
[`POST /v1/calls/list`](list-calls/index.md) endpoint'ten aldığınız `call_id` değerleri için `recording_status` neredeyse her zaman ya `completed` (200 alırsınız) ya da `failed` (409) / `not_found` (404) olur — her ikisi de kalıcıdır. Çünkü bir çağrı, ses kaydı kalıcı bir duruma ulaşmadan listede görünmez. `pending` ve `processing` durumları normal API akışının parçası değildir; nadir görülen yarış koşullarına (race condition) karşı önlem olarak burada listelenmiştir.
:::

## Notlar

- **Bağlantıyı önbelleğe almayın.** Bağlantı 24 saat sonra geçerliliğini yitirir. Veritabanınıza kaydederseniz, geçerliliğini yitirmiş bağlantılarla karşılaşırsınız. Bağlantıyı her gerektiğinde yeniden oluşturun.
- **Birden çok indirme.** Aynı bağlantıyı 24 saat boyunca birden çok GET isteğiyle kullanabilirsiniz. Kayıtları farklı kullanıcılara iletiyorsanız, **her kullanıcı için ayrı bir bağlantı oluşturun**.
- **Kalıcı vs. geçici.** [`POST /v1/calls/list`](list-calls/index.md) endpoint'ten alınan `call_id` değerlerinde hazır-değil yanıtı neredeyse her zaman kalıcıdır — ya `404 RECORDING_NOT_AVAILABLE` (`recording_status: "not_found"`) ya da `recording_status: "failed"` taşıyan bir `409 RECORDING_NOT_READY`. İkisinde de yeniden denemek sonucu değiştirmez. Nadir yarış koşullarında `processing` / `pending` taşıyan bir `409` görebilirsiniz; bu durumda birkaç dakika sonra tekrar deneyebilirsiniz.
- **Biçim.** Ses dosyaları genellikle `.wav` biçimindedir (mono, 8 kHz veya 16 kHz). Bazı kayıtlar farklı bir codec kullanabileceğinden, `Content-Type` header'ını denetlemeniz güvenli olur.
- **Boyut.** Tipik olarak 1–10 MB; uzun çağrılarda 30 MB'a kadar çıkabilir.

## Örnekler

<Tabs groupId="lang">
<TabItem value="curl" label="curl">

```bash
# 1. Bağlantıyı alın
curl -H "Authorization: Bearer $VINDY_API_KEY" \
  https://api.vindy.vinter.me/v1/calls/12345/recording-url
# → { "url": "https://...call.wav?X-Amz-...", "expires_at": "..." }

# 2. İndirin (bağlantıyı tırnak içine alın — sorgu dizesi uzundur)
curl -o call-12345.wav "https://...call.wav?X-Amz-..."
```

</TabItem>
<TabItem value="node" label="Node.js">

```javascript
import { writeFile } from "node:fs/promises";

async function downloadRecording(callId) {
  // 1. Güncel bir imzalı bağlantı alın
  const response = await fetch(
    `https://api.vindy.vinter.me/v1/calls/${callId}/recording-url`,
    { headers: { Authorization: `Bearer ${process.env.VINDY_API_KEY}` } },
  );

  if (response.status === 404) {
    const error = await response.json();
    if (error.code === "RECORDING_NOT_AVAILABLE") {
      return null; // kalıcı — bu çağrı için hiç ses kaydı üretilmemiş
    }
    throw new Error(`${error.code}: ${error.message}`); // RESOURCE_NOT_FOUND
  }
  if (response.status === 409) {
    const error = await response.json();
    const status = error.extensions?.recording_status;
    if (status === "failed") {
      return null; // kalıcı — aktarım kalıcı olarak başarısız, yeniden denemeyin
    }
    throw new Error(`Ses kaydı geçici durumda: ${status} — sonra tekrar deneyin`);
  }
  if (!response.ok) {
    const error = await response.json();
    throw new Error(`${error.code}: ${error.message}`);
  }

  // 2. Ses dosyasını indirin
  const { url } = await response.json();
  const audio = await fetch(url);
  await writeFile(`call-${callId}.wav`, Buffer.from(await audio.arrayBuffer()));
  return `call-${callId}.wav`;
}

await downloadRecording(12345);
```

</TabItem>
<TabItem value="python" label="Python">

```python
import os
import requests

def download_recording(call_id):
    # 1. Güncel bir imzalı bağlantı alın
    response = requests.get(
        f"https://api.vindy.vinter.me/v1/calls/{call_id}/recording-url",
        headers={"Authorization": f"Bearer {os.environ['VINDY_API_KEY']}"},
    )

    if response.status_code == 404:
        error = response.json()
        if error.get("code") == "RECORDING_NOT_AVAILABLE":
            return None  # kalıcı — bu çağrı için hiç ses kaydı üretilmemiş
        raise RuntimeError(f"{error.get('code')}: {error.get('message')}")  # RESOURCE_NOT_FOUND
    if response.status_code == 409:
        error = response.json()
        status = error.get("extensions", {}).get("recording_status")
        if status == "failed":
            return None  # kalıcı — aktarım kalıcı olarak başarısız, yeniden denemeyin
        raise RuntimeError(f"Ses kaydı geçici durumda: {status} — sonra tekrar deneyin")
    if not response.ok:
        error = response.json()
        raise RuntimeError(f"{error.get('code')}: {error.get('message')}")

    # 2. Ses dosyasını indirin
    url = response.json()["url"]
    audio = requests.get(url)
    audio.raise_for_status()

    path = f"call-{call_id}.wav"
    with open(path, "wb") as f:
        f.write(audio.content)
    return path

download_recording(12345)
```

</TabItem>
</Tabs>
