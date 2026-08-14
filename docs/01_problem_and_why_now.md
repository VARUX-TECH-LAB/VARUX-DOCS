# Problem Ve Neden Simdi

## Kurumsal Risk Sinifi

- Kritik veritabanlarinda en buyuk risklerden biri, yetkili kanaldan gelen ama is sonucu tehlikeli olan yazma islemleridir.
- Ornek risk siniflari:
  - Yanlis migration veya operasyon scripti.
  - Hatalı uygulama kodu veya ORM davranisi.
  - Ele gecirilmis uygulama kimlik bilgileri.
  - Iyi niyetli ama yanlis kapsamli toplu veri degisikligi.
- Bu islemler cogu zaman veritabanina gecerli kullanici, gecerli sifre ve gecerli ag yolu ile gelir.

## Mevcut Araclarin Goremeyebilecegi Katman

- Firewall, WAF ve klasik ag kontrolleri paketin nereye gittigini bilir; veritabani isteginin anlamini genellikle bilmez.
- Bir istegin "veri okuma" mi, "toplu veri silme" mi, "genis kapsamli degistirme" mi oldugunu anlamak SQL semantigi gerektirir.
- Uygulama loglari olaydan sonra faydalidir; ancak tehlikeli islemi calismadan once durdurmak icin yeterli olmayabilir.

## AXIS'in Hedefledigi Bosluk

- AXIS, veritabani isteginin calismadan hemen onceki noktasinda karar vermeyi hedefler.
- Amac, normal yetki zinciri icinden gelen ama riskli olan islemleri policy ile durdurmak veya ek onaya yonlendirmektir.
- Bu, mevcut IAM, veritabani yetkileri, ag segmentasyonu ve audit kontrollerinin yerine gecmez; onlarin onune ek bir son kontrol noktasi koyar.
