# FIRST REVIEW - AXIS Ilk Mimari Paylasim Kaynagi

AXIS, uygulama ile PostgreSQL veritabani arasina girerek riskli veritabani islemlerini calismadan once kontrol eden ve izin verilmeyenleri durduran bir guvenlik katmanidir; uygulama normal veritabanina baglaniyormus gibi davranir, ancak korunan istekler once AXIS politikasindan gecer.

## Kanit Noktalari

- Real-driver test paketi 2026-07-05 tarihinde 37 test olarak gecmis gorunuyor.
- Dort ekosistem repo kanitlarinda yer aliyor: psycopg3, asyncpg, Prisma Client 6.19.3 ve PostgreSQL JDBC pgjdbc 42.7.7.
- Parametreli SELECT, INSERT, UPDATE ve policy tarafindan engellenen DELETE senaryolari bu dort ekosistemde test matrisi kapsaminda.
- Engellenen DELETE akislari SQLSTATE 42501 ile dogrulaniyor; baglanti sonraki islemler icin kullanilabilir kaliyor.
- Savepoint ile kismi kurtarma psycopg3, asyncpg, Prisma ve PostgreSQL JDBC icin test matrisinde kapsaniyor.

Bu materyal, AXIS'in mimarisini ve bugunku kanitli sinirlarini Gokay'a ilk kez tam mimari baglamiyla anlatmak icin hazirlanmis sunum kaynagidir.
