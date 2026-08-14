# Ana Tasarim Kararlari

## Varsayilan Davranis: Reddet

- AXIS, anlayamadigi veya desteklemedigi istegi sessizce gecirmez.
- Neden: Guvenlik katmaninda belirsizlik, izin anlamina gelmemelidir.
- Kanit: driver matrix ve pgwire testleri, desteklenmeyen batch ve COPY gibi akislarda fail-closed davranisi belgeliyor.

## Reddedilen Islem PostgreSQL'e Benzer Hata Uretir

- Engellenen isteklerde uygulama temiz bir PostgreSQL uyumlu hata gorur.
- Engellenen DELETE senaryolarinda SQLSTATE 42501 bekleniyor.
- Baglanti sonraki guvenli istekler icin kullanilabilir kalir.
- Neden: Uygulama tarafinda normal hata ve transaction kurtarma mantigi calisabilsin.

## Ham Degerleri Azaltan Policy Payload Tasarimi

- Parametreli Extended Query yolunda ham parametre degerleri policy payload'a konmaz; hash ozetleri ve sayisal/meta bilgiler kullanilir.
- psycopg3 ve asyncpg real-driver testleri, audit WAL icinde ham marker degerlerinin bulunmadigini kontrol eder.
- Prisma ve JDBC icin driver matrix, production leakage testinin henuz eklenmedigini acikca belirtiyor.
- Neden: Guvenlik karari icin gereken sinyal korunurken musteri verisinin policy ve log yuzeyine yayilmasi azaltılır.

## Savepoint Ile Kismi Kurtarma

- AXIS, savepoint kullanan transaction akıslarında riskli adimin durdurulmasi ve uygulamanin kendi kurtarma mantigiyla devam edebilmesi davranisini test ediyor.
- psycopg3, asyncpg, Prisma ve PostgreSQL JDBC icin savepoint recovery test matrisi kapsaminda.
- Neden: Kurumsal uygulamalar tum transaction'i kaybetmeden kontrollu kurtarma yapmak isteyebilir.

## Multi-statement Ve Session Kontrolleri

- Simple Query icinde birden fazla statement varsa her statement ayri siniflandirilir.
- Session parametrelerinde yalnizca dar ve gerekceli bir GUC whitelist'i gecirilir.
- search_path, role, session_authorization ve row_security reddedilir.
- Neden: Policy karari ile veritabaninin gercek nesne veya yetki baglami arasinda ayrisma olusmamali.
