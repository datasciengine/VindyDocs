---
title: Kayıt İndirme
sidebar_label: Kayıt İndirme
sidebar_position: 2
---

# Kayıt İndirme

Çağrı kayıtlarını güvenilir biçimde nasıl indireceğinizi ve indirilecek bir kayıt olmadığını nasıl anlayacağınızı bu bölümde bulabilirsiniz.

---

## Yaklaşım

1. Çağrıları [`POST /v1/calls/list`](../api-reference/list-calls/index.md) ile alın. Bir ses kaydı mevcut ve erişilebilir durumdaysa, `call_recording` içinde doğrudan bir bağlantı döner.
2. `call_recording.available: false` ise bu **kalıcı bir durumdur**; ya kayıt hiç üretilmemiştir ya da aktarımı kalıcı olarak başarısız olmuştur. Yeniden denemek sonucu değiştirmez. Kesin nedeni öğrenmek için [`GET /v1/calls/:callId/recording-url`](../api-reference/get-recording-url.md) endpoint'i çağırın ve `recording_status` alanını denetleyebilirsiniz.
3. Ses dosyasını **kendi depolama alanınıza** indirin; imzalı bağlantı 24 saat sonra geçerliliğini yitirir. Bağlantıyı veritabanınızda saklamanız önerilmez.
4. İndirme sırasında bağlantının süresi dolarsa, aynı endpoint'e yeniden GET isteği göndererek 24 saat geçerli yeni bir bağlantı alın.

---

## Karar tablosu

| Gördüğünüz | Anlamı | Yapılması gereken |
|---|---|---|
| `call_recording.available: true` + `url` | Ses kaydı mevcut | Hemen indirin ya da daha sonra güncel bir bağlantı oluşturun |
| `call_recording.available: false` | **Kalıcı** — kayıt yok veya aktarım kalıcı olarak başarısız | Yeniden denemenize gerek yoktur. İsterseniz kesin nedeni `recording_status` ile öğrenin |
| 404 `RECORDING_NOT_AVAILABLE` (`recording_status: not_found`) | **Kalıcı** — hiç ses kaydı üretilmemiş | Yeniden denemenize gerek yoktur. Kaydın var olması gerektiğini düşünüyorsanız Vindy'ye bildirebilirsiniz |
| `recording_status: failed` taşıyan 409 `RECORDING_NOT_READY` | **Kalıcı** — aktarım kalıcı olarak başarısız | Yeniden denemenize gerek yoktur. Kaydın var olması gerektiğini düşünüyorsanız Vindy'ye bildirebilirsiniz |
| `recording_status: pending` / `processing` taşıyan 409 `RECORDING_NOT_READY` | Nadir yarış koşulu | Birkaç dakika sonra tekrar deneyebilirsiniz |

:::caution En sık yapılan hata
`call_recording.available` değeri `false` olan bir çağrı için `recording-url` endpoint'i bir yeniden deneme döngüsünde sürekli sorgulamak. Bu durum **kesindir**; bir çağrı, ses kaydı kalıcı bir duruma ulaşmadan zaten `/v1/calls/list` listesinde görünmez. Yeniden deneme bütçenizi gerçek ağ hataları için saklayın.
:::

---

## Temel kurallar

- **İmzalı bağlantıyı hiçbir zaman saklamamalısınız.** Bunun yerine `call_id` değerini saklayın ve bağlantıyı gerektiğinde yeniden oluşturun.
- **Her alıcı için ayrı bir bağlantı.** Kayıtları kendi kullanıcılarınıza iletecekseniz, tek bir bağlantıyı paylaşmak yerine her kullanıcı için ayrı bir bağlantı oluşturun.
- **`Content-Type` değerini denetleyebilirsiniz.** Ses dosyaları genellikle `.wav` biçimindedir (mono, 8 kHz veya 16 kHz); ancak bazı kayıtlar farklı bir codec kullanabilir.
- **Kayıt başına 1–10 MB** bekleyin; uzun çağrılar 30 MB'a kadar çıkabilir.

Node.js ve Python için eksiksiz indirme kodunu [recording-url örneklerinde](../api-reference/get-recording-url.md#örnekler) bulabilirsiniz.
