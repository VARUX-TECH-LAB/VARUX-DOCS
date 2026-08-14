# Ek: Kisa Sozluk

- Extended Query: PostgreSQL client'larinin parametreli sorgulari parcalara ayirarak gonderdigi protocol akisi.
- Savepoint: Bir transaction icinde geri donulebilen ara nokta; tum islemi degil sadece sonraki riskli adimi geri almak icin kullanilabilir.
- Redaction/hash: Ham degeri saklamak yerine degeri gizlemek veya karsilastirilabilir bir ozetle temsil etmek.
- Fail-closed: Sistem emin degilse veya desteklemiyorsa istegi gecirmek yerine kontrollu sekilde reddetmesi.
- Tamper-evident audit: Repo testleri yerel hash-chain dogrulama ve imzali export kurcalama tespiti kapsiyor; harici degistirilemez ledger bugunku kanitli kapsam degil.
- Politika motoru: Istegin turunu, hedefini ve riskini degerlendirip izin, engelleme veya onay gerektirir sonucunu ureten karar katmani.
- PostgreSQL wire protocol: Uygulama ile PostgreSQL arasindaki dusuk seviyeli iletisim dili.
- COPY protokolu: PostgreSQL'in toplu veri alma veya verme yolu; AXIS pgwire modunda desteklenmez ve fail-closed reddedilir.
- GUC/session parameter: PostgreSQL oturum ayari; AXIS yalnizca guvenli kabul edilen dar bir listeyi gecirir.
- Driver matrix: Gercek client ve ORM'lerle hangi davranislarin test edildigini gosteren kanit tablosu.
