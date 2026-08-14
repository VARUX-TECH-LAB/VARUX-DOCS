# Zihinsel Model

AXIS'i production veritabanının önündeki kontrollü kapı gibi düşünün. Bu kapı, gelen isteğin kimden geldiğine bakar; fakat asıl olarak çalıştırılmak üzere olan SQL'in ne yaptığını inceler.

Bu benzetme basittir, ama çocukça değildir: production verisi üzerinde işlem yapmadan önce kontrol noktası oluşturmak ciddi bir operasyonel güvenlik ihtiyacıdır.

## Model 1: Veritabanı kapısı

Uygulama doğrudan production PostgreSQL'e write göndermek yerine AXIS'e gelir. AXIS, isteği geçirebilir, durdurabilir veya onay isteyebilir.

```text
Uygulama / Script / Servis / AI Agent
                |
                v
        +----------------+
        | AXIS Kapısı    |
        | parse + policy |
        +----------------+
          |      |      |
          |      |      +--> Onay bekle
          |      +---------> Blokla
          +----------------> PostgreSQL'e gönder
```

## Model 2: Güvenlik kontrol noktası

AXIS bir firewall gibi paket bakmaz; SQL operasyonunun anlamına bakar. Örneğin `DELETE FROM users` ve `DELETE FROM users WHERE id = ? AND tenant_id = ?` aynı riskte değildir.

Kontrol edilen sinyaller:

- operasyon tipi: read, insert, update, delete, DDL
- scope: single, batch, unknown
- target: database, schema, table
- risk sinyalleri: `delete_without_where`, `bulk_operation`, `unknown_target`
- ortam: prod veya non-prod
- aktör tipi: human, service, ci_cd, ai_agent

## Model 3: Policy hakemi

Policy engine hakem gibi çalışır. İstek için bir karar üretir:

| Karar | Anlam |
|---|---|
| `ALLOW` | İstek çalıştırılabilir. |
| `BLOCK` | İstek PostgreSQL'e gönderilmez. |
| `APPROVAL_REQUIRED` / `REQUIRE_APPROVAL` | İstek ilk aşamada çalıştırılmaz; approval kaydı oluşur. |

Mevcut API'de karar adı `REQUIRE_APPROVAL` olarak görülebilir. Bu dokümanda `APPROVAL_REQUIRED`, aynı kavramı anlatmak için kullanılır.

## Model 4: Evidence kaydedici

AXIS sadece "izin verdim" veya "blokladım" demez. Kararın neden verildiğini audit evidence içine yazar.

Evidence şunları bağlar:

- request identity ve context
- SQL fingerprint
- classifier sonucu
- policy id, version ve SHA-256
- decision ve reason code
- approval id varsa approval bağı
- `previous_hash`
- `event_hash`

Bu sayede reviewer "ne oldu?" sorusuna normal runtime log yerine doğrulanabilir audit event üzerinden bakabilir.

## Kısa özet

AXIS'in zihinsel modeli dört parçadır:

```text
Kontrollü kapı
    -> SQL'i anlamaya çalışan sınıflandırıcı
    -> policy hakemi
    -> audit evidence kaydedici
```

Bu model, deployment disiplinine bağlıdır. AXIS'ten geçmeyen doğrudan database write path, AXIS tarafından korunmuş sayılmaz.

