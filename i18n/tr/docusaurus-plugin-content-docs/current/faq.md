---
title: SSS
sidebar_label: SSS
sidebar_position: 8
---

# Sık Sorulan Sorular

## Başka bir şirketin verisini görebilir miyim?

Hayır. Her API anahtarı yalnızca tek bir şirkete bağlıdır ve her istek otomatik olarak o şirkete kapsanır. Başka bir şirketin `call_id` değerini kullandığınızda 404 yanıtı alırsınız ve böyle bir kaydın var olup olmadığını dahi anlayamazsınız. Bkz. [Multi-tenancy](concepts/multi-tenancy.md).

## API anahtarımı kaybettim. Kurtarabilir misiniz?

Hayır. Anahtarın açık metni yalnızca oluşturma anında bir kez gösterilir. Yeni bir anahtar oluşturun ve eskisini iptal edin. Bkz. [Kimlik Doğrulama](authentication.md).

## Bir çağrı neden `POST /v1/calls/list` listesinde görünmüyor?

Bu endpoint yalnızca tamamen kesinleşmiş çağrıları döndürür: sona ermiş **ve** ses kaydı aktarımı sonuçlanmış çağrılar. Yeni sona eren bir çağrının listede görünmesi kısa bir süre alabilir. Hâlâ devam eden çağrılar hiçbir zaman görünmez. Bkz. [yarım veri döndürülmez](api-reference/list-calls/index.md).

## Bir kayıt Vindy panelinde görünüyor ancak API `available: false` döndürüyor. Bu bir hata mı?

Hayır, beklenen bir durumdur. Panel, ses kayıtlarını geçici kaynaklardan gösterebilir; API ise yalnızca kalıcı depolamadaki kayıtları sunar. Müşteri tarafı için bağlayıcı olan, API yanıtıdır. Bkz. [açıklama](api-reference/list-calls/index.md#recording-not-available).

## `call_recording.available` değeri `false`. Yeniden denemeli miyim?

Hayır; bu durum **kalıcıdır**. Ya hiç kayıt üretilmemiştir ya da aktarımı kalıcı olarak başarısız olmuştur. Bkz. [kayıt indirme](guides/recording-retrieval.md).

## İstekleri yeniden denemek güvenli mi?

Evet. Tüm `GET` endpoint'leri idempotenttir. `POST /v1/calls/list` ise bir gövde kullanmasına karşın **bir değişiklik (mutation) değil, bir sorgudur**; yan etkisi yoktur ve yeniden denenmesi güvenlidir. Kayıtları kendi tarafınızda upsert ettiğinizde (`call_id` üzerinde benzersizlik kısıtı) yeniden denemeler zararsız hâle gelir.

Yazma istekleri farklıdır: `POST /v1/calls/bulk` çağrı oluşturur; bu yüzden körlemesine yeniden denemek ikinci bir batch başlatıp kişileri iki kez aratabilir (eşzamanlı bir yeniden deneme `409 BATCH_IN_PROGRESS` ile engellenir). İptal endpoint'leri ise güvenle tekrar çağrılabilir.

## Ne sıklıkla sorgulama yapmalıyım?

Dakikada birden fazla yapmamanız önerilir. Sürekli senkronizasyon için `from_date` değerini son senkronizasyon zamanınızla kullanabilirsiniz; bkz. [artımlı senkronizasyon](guides/incremental-sync.md).

## Tarih filtrelerim neden 400 döndürüyor?

Büyük olasılıkla eksik bir saat dilimi (`MISSING_TIMEZONE`) veya ISO dışı bir biçim (`INVALID_DATE_FORMAT`) söz konusudur. Kabul edilen ve reddedilen biçimler için [Filtreleme ve Sayfalama](api-reference/list-calls/filtering-pagination.md) sayfasına bakabilirsiniz.

## Bir sorunu nasıl bildiririm?

Aşağıdakilerin tümünü ekleyin; bu, sorunun çözümünü belirgin biçimde hızlandırır:

- Tam HTTP yöntemi ve URL
- İstek header'ları (**Authorization anahtarını maskeleyin**: `Bearer 01902f6e...***`)
- İstek gövdesi
- Yanıt durumu ve gövdesi
- Yanıttaki `X-Request-Id` header'ının değeri
