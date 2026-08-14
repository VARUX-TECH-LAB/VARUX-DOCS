# Pilot Taslagi Ve Sonraki Adimlar

Bu bolum taslaktir, Selman ticari detaylari kendisi doldurmadan gonderilmemelidir.

## Baglayici Olmayan Pilot Secenekleri

- Tek uygulama ile sinirli pilot:
  - Kapsam dar tutulur.
  - Sadece belirlenen write path AXIS uzerinden gecirilir.
  - Basari kriterleri onceden yazilir.
- Gozlem modu ile baslama:
  - Ilk amac SQL sekillerini ve policy etkisini anlamaktir.
  - Engelleme politikalari daha sonra sinirli kapsamda acilir.
- Non-production ortamda teknik dogrulama:
  - Gercek driver ve ORM kombinasyonlari profillenir.
  - Migration, batch ve admin isleri kapsam disi veya ayri yol olarak tanimlanir.
- Dar policy pilotu:
  - Oncelik destructive write siniflarina verilir.
  - Bilinmeyen veya karmasik SQL sekilleri fail-closed olarak ele alinir.
- Ortak kanit paketi:
  - Test listesi, driver matrisi, limitation listesi ve pilot bulgulari birlikte saklanir.

## Gonderimden Once Doldurulacak Noktalar

- Pilot ortami: non-production, staging veya kontrollu production shadow.
- Kapsama alinacak uygulama ve veritabani rolu.
- Hangi islem siniflarinin engellenecegi veya sadece gozlemlenecegi.
- Basari kriterleri: baglanti uyumu, engelleme davranisi, rollback/kurtarma, audit gorunurlugu.
- Operasyonel sahiplik: kim policy degistirir, kim incident durumunda karar verir.
- Ticari ve hukuki sartlar: Selman tarafindan ayrica doldurulmali.
