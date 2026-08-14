# Real-driver Kanitlari

## Test Sayisi Ve Kapsam

- docs/technical/driver_matrix.md, full real-driver suite icin 2026-07-05 tarihinde 37 testin gectigini belirtiyor.
- Kapsanan ekosistemler:
  - psycopg3, Python.
  - asyncpg, Python.
  - Prisma Client 6.19.3, TypeScript/JavaScript.
  - PostgreSQL JDBC, Java, pgjdbc 42.7.7.

## Ekosistem Bazinda Kanit Ozeti

- psycopg3 ile: parametreli sorgular, INSERT, UPDATE, engellenen DELETE, transaction abort, savepoint recovery, prepared reuse, SQL-level prepare/execute deny ve audit raw-value leakage kontrolleri dogrulandi.
- asyncpg ile: parametreli sorgular, prepared statement akisi, engellenen DELETE, transaction abort, savepoint recovery, SQL-level prepare/execute deny ve audit raw-value leakage kontrolleri dogrulandi.
- Prisma ile: parametreli SELECT, tekil INSERT, tekil UPDATE, engellenen DELETE, transaction abort ve interactive transaction icinde savepoint recovery dogrulandi.
- PostgreSQL JDBC ile: parametreli SELECT, tekil INSERT, tekil UPDATE, engellenen DELETE, transaction abort, savepoint recovery, prepared reuse ve default connection startup dogrulandi.

## Engelleme Ve Kurtarma Kanitlari

- Engellenen DELETE davranisi dort ekosistemde SQLSTATE 42501 beklentisiyle test ediliyor.
- Deny sonrasi baglanti kullanilabilirligi psycopg3, asyncpg, Prisma ve JDBC senaryolarinda kapsaniyor.
- JDBC batch execution artik destekleniyor ve Hibernate 6 + pgjdbc ile uçtan uca doğrulandi (explicit-OID bind, tek Sync oncesi coklu Bind/Execute, savepoint/rollback); pozitif ve negatif policy-enforcement kontroleri ve imzali audit evidence ile. Asiri buyuk batch'ler reddediliyor; engellenen batch item'lari fail-closed ile kurtariliyor. (Kaynak: driver_matrix.md status update 2026-08 ve round-8 ORM ground-truth evidence.)
- JDBC default connection artik assumeMinServerVersion=12 olmadan dogrulaniyor.

## Sızıntı Kontrolu Durumu

- psycopg3 ve asyncpg icin audit WAL marker kontrolleri var.
- Prisma ve JDBC icin production leakage testi henuz eklenmedi; bu durum driver matrix'te acikca not edilmis.
