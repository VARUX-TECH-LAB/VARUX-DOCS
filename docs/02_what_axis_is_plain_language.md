# AXIS Nedir

AXIS, uygulama ile PostgreSQL arasina seffaf sekilde giren ve tehlikeli veritabani islemlerini calismadan once durduran bir politika katmanidir.

## Duz Anlatim

- Uygulama PostgreSQL'e baglaniyormus gibi davranir.
- AXIS, istegin ne yapmaya calistigini siniflandirir.
- Izinli istekler PostgreSQL'e ulasir.
- Izin verilmeyen istekler veritabanina gonderilmeden durdurulur.
- Uygulama, engellemeyi PostgreSQL uyumlu bir hata olarak gorur ve kendi hata veya geri alma mekanizmasini kullanabilir.

## Audit Ve Kanit Durumu

- Repo, yerel audit WAL kayitlari, hash zinciri alanlari ve dogrulama endpointleri icin testler iceriyor.
- Testlerde audit dogrulama, zincir bozulmasini "tampered" olarak raporlayabiliyor.
- Testlerde imzali export yapilandirildiginda, degistirilmis export bundle imza dogrulamasindan gecmiyor.
- Bu, yerel kanit butunlugu ve dogrulama davranisinin test edildigini gosterir.
- Bu repo, harici degistirilemez defter, KMS destekli imza dagitimi veya coklu instance audit konsensusu iddiasini kanitlanmis uretim ozelligi olarak sunmuyor.
