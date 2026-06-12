---
title: Hata Kodları
sidebar_label: Hata Kodları
sidebar_position: 6
---

# Hata Kodları

Her hata, makine-okunabilir bir `code` değerini `extensions.code` içinde taşır; aşağıdaki katalogdaki hatalarda bu değer üst düzeyde de tekrarlanır. Kendi sisteminizde dallanma yaparken, zamanla değişebilen insan-okunabilir `message` alanını değil bu kodu kullanın. Tek istisna beklenmeyen sunucu hatalarıdır (`500`): bu hatalarda üst düzey `code` bulunmaz ve kod yalnızca `extensions.code` içinde `HTTP_500` olarak yer alır.

Tüm hata yanıtları aynı JSON yapısını paylaşır; bkz. [hata formatı](concepts/response-envelopes.md#error-envelope).

---

## Katalog

| Kod | Durum | Açıklama | Ne yapmalı? |
|---|---|---|---|
| `MISSING_AUTH_HEADER` | 401 | `Authorization` header'ı eksik. | Header'ı ekleyin. |
| `INVALID_AUTH_FORMAT` | 401 | Header `Bearer <api-key>` biçiminde değil. | Biçimi düzeltin. |
| `INVALID_API_KEY` | 401 | API anahtarı geçersiz, süresi dolmuş veya iptal edilmiş. | Yeni bir anahtar oluşturun. |
| `RESOURCE_NOT_FOUND` | 404 | Kaynak (örneğin çağrı) bulunamadı veya sizin şirketinize ait değil. | Kimliği doğrulayın. |
| `VALIDATION_FAILED` | 400 | Gövde veya yol parametresi doğrulaması başarısız. | `validation_errors` doğrultusunda düzeltin. |
| `INVALID_DATE_FORMAT` | 400 | Tarih ISO 8601 biçiminde değil. | Bkz. [Filtreleme ve Sayfalama](api-reference/list-calls/filtering-pagination.md). |
| `MISSING_TIMEZONE` | 400 | Tarih-saat değerinde saat dilimi (Z veya offset) eksik. | `Z` veya `+HH:MM` ekleyin. |
| `DATE_RANGE_INVALID` | 400 | `from_date >= to_date` veya başka biçimde geçersiz aralık. | Aralığı düzeltin. |
| `INVALID_CURSOR` | 400 | Cursor çözümlenemedi. | Önceki bir yanıttan alınan güncel bir cursor kullanabilirsiniz. |
| `MALFORMED_CURSOR` | 400 | Cursor içeriği beklenen yapıda değil. | Cursor'u değiştirmemelisiniz; döndürüldüğü biçimde kullanabilirsiniz. |
| `RECORDING_NOT_AVAILABLE` | 404 | Çağrı mevcut ama bu çağrı için hiç ses kaydı üretilmemiş. **Kalıcı.** | Yeniden denemeyin. `recording_status: "not_found"` taşır; bkz. [recording-url](api-reference/get-recording-url.md#recording-status). |
| `RECORDING_NOT_READY` | 409 | Ses kaydı var ama henüz indirilebilir değil — ya kalıcı olarak `failed` (kalıcı) ya da hâlâ `pending`/`processing` (geçici). | `recording_status` değerini denetleyebilirsiniz; bkz. [recording-url](api-reference/get-recording-url.md#recording-status). |
| `INVALID_PHONE_NUMBER` | 400 | Bir `calls[i].phone_number` geçerli bir Türkiye numarası değil. | Türkiye numarası kullanın; hatalı indeks `extensions.details.index` içindedir. Bkz. [Toplu Çağrı Oluştur](api-reference/bulk-create-calls.md#phone-numbers). |
| `INVALID_METADATA` | 400 | Bir çağrının metadata'sı limitleri aşıyor veya geçersiz bir değer tipi kullanıyor. | Limitlere uyun; hatalı indeks `extensions.details.index` içindedir. Bkz. [Toplu Çağrı Oluştur](api-reference/bulk-create-calls.md#metadata). |
| `ASSISTANT_NOT_FOUND` | 404 | Asistan bulunamadı, sizin şirketinize ait değil veya arama için uygun değil. | `assistant_id` değerini doğrulayın. |
| `SQUAD_NOT_FOUND` | 404 | Squad bulunamadı, sizin şirketinize ait değil veya arama için uygun değil. | `squad_id` değerini doğrulayın. |
| `NO_OUTBOUND_PHONE_NUMBER` | 409 | Hesabınızda giden arama yapabilecek bir numara tanımlı değil. | Vindy ekibiyle iletişime geçin. |
| `BATCH_IN_PROGRESS` | 409 | Hesabınızda zaten devam eden bir toplu arama var. | Bitmesini bekleyip tekrar deneyin. |
| `ERR_CALL_NOT_FOUND` | 400 | İptal edilecek çağrı bulunamadı veya sizin şirketinize ait değil. | `callId` değerini doğrulayın. Bkz. [Tek Bir Çağrıyı İptal Et](api-reference/cancel-call.md). |
| `ERR_CALL_PROCESSING_CANCEL` | 409 | Çağrı şu anda aranıyor ve artık iptal edilemez. | Yeniden denemeyin; sonucu bekleyin. Bkz. [Tek Bir Çağrıyı İptal Et](api-reference/cancel-call.md). |
| `ERR_CALL_ALREADY_FINAL` | 400 | Çağrı zaten sonlanmış (tamamlandı/başarısız/iptal). | İptal edilecek bir şey yok. Bkz. [Tek Bir Çağrıyı İptal Et](api-reference/cancel-call.md). |
| `ERR_BATCH_NOT_FOUND` | 400 | İptal edilecek batch bulunamadı veya sizin şirketinize ait değil. | `batchId` değerini doğrulayın. Bkz. [Bir Batch'i İptal Et](api-reference/cancel-batch.md). |
| `ERR_BATCH_ALREADY_FINAL` | 400 | Batch artık çalışmıyor. | İptal edilecek bir şey yok. Bkz. [Bir Batch'i İptal Et](api-reference/cancel-batch.md). |
| `ERR_BATCH_NO_PENDING_CALLS` | 400 | Batch'te iptal edilecek bekleyen çağrı kalmamış. | İptal edilecek bir şey yok. Bkz. [Bir Batch'i İptal Et](api-reference/cancel-batch.md). |
| `HTTP_500` | 500 | Beklenmeyen sunucu hatası. `extensions.code` içinde yer alır; üst düzey `code` bulunmaz. | `requestId` ile birlikte bize bildirebilirsiniz. |
