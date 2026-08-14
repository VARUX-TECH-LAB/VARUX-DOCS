# Runtime Akışları

Bu dosya, AXIS'in temel runtime akışlarını ayrı ayrı gösterir. Diyagramlar basit tutulmuştur; amaç reviewer'ın execution öncesi karar ve evidence bağlantısını hızlıca görmesidir.

## 1. Safe READ allowed

Safe read, policy default veya explicit allow ile çalışabilir. Yine de request ve decision evidence üretilebilir.

```mermaid
flowchart TD
  A[Safe SELECT isteği] --> B[Parse ve classify READ]
  B --> C[Policy evaluate]
  C -->|ALLOW| D[Decision evidence]
  D --> E[PostgreSQL execution]
  E --> F[Result evidence]
  F --> G[Response 200]
```

## 2. Safe WRITE allowed

Scoped write, policy'de explicit allow varsa çalışabilir. Protected write için decision evidence execution öncesinde commit edilmelidir.

```mermaid
flowchart TD
  A[Scoped WRITE isteği] --> B[Scope ve target çıkar]
  B --> C[Policy explicit allow]
  C --> D[Audit decision commit]
  D --> E[Database adapter]
  E --> F[PostgreSQL]
  F --> G[Execution result evidence]
```

## 3. Risky WRITE requiring approval

Approval gereken write ilk istekte çalışmaz.

```mermaid
flowchart TD
  A[Riskli WRITE] --> B[Classifier risk sinyalleri]
  B --> C[Policy evaluate]
  C -->|APPROVAL_REQUIRED| D[Approval created evidence]
  D --> E[PENDING approval]
  E --> F[Response 202 approval_id]
  F --> G[Execution yok]
```

## 4. BLOCK flow

Block kararı PostgreSQL'e gitmez.

```mermaid
flowchart TD
  A[Tehlikeli veya geçersiz SQL] --> B[Parser veya policy]
  B --> C[BLOCK]
  C --> D[Block evidence]
  D --> E[Execution yok]
  E --> F[Response 4xx]
```

## 5. Approval resolve flow

Approve/reject final state üretir. Approve execution değildir; caller aynı request ile retry yapar.

```mermaid
flowchart TD
  A[PENDING approval] --> B[Operator resolve]
  B --> C{Karar}
  C -->|Approve| D[APPROVED state]
  C -->|Reject| E[REJECTED state]
  D --> F[Resolved evidence]
  E --> F
  F --> G[Final state immutable]
```

## 6. Audit verification flow

Verification WAL'ı okur, event hash ve previous hash zincirini kontrol eder.

```mermaid
flowchart TD
  A[Audit WAL] --> B[Event satırlarını oku]
  B --> C[event_hash recompute]
  C --> D[previous_hash kontrol]
  D --> E{Tutarlı mı?}
  E -->|Evet| F[verified]
  E -->|Hayır| G[tampered veya unverifiable]
```

## 7. Startup policy integrity flow

Startup policy bütünlüğü doğrulanmadan trafik kabul edilmemelidir.

```mermaid
flowchart TD
  A[Başlangıç] --> B[Audit WAL continuity]
  B --> C[Policy manifest oku]
  C --> D[Policy SHA-256 doğrula]
  D --> E[Schema validation]
  E --> F[Activation dry-run]
  F --> G{Geçti mi?}
  G -->|Evet| H[Listener trafik kabul eder]
  G -->|Hayır| I[Fail-fast]
```

## 8. Corrupt audit fail-fast flow

Audit WAL bozuksa sistem güvenli başlamamalıdır.

```mermaid
flowchart TD
  A[AXIS başlıyor] --> B[Audit WAL oku]
  B --> C{Hash zinciri geçerli mi?}
  C -->|Evet| D[Son hash ile devam et]
  C -->|Hayır| E[Startup fail-fast]
  E --> F[Protected traffic yok]
```

## 9. Evidence export flow

Evidence export, seçilmiş event'leri ve verification metadata'yı taşınabilir hale getirir.

```mermaid
flowchart TD
  A[Audit WAL] --> B[Filtrele ve event seç]
  B --> C[Verification metadata ekle]
  C --> D[Policy metadata ekle]
  D --> E[Bundle üret]
  E --> F[Offline verification için paylaş]
```

