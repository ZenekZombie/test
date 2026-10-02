# 7. Bezpieczeństwo, Audyt i Niezaprzeczalność

## 7.1 Kryptografia i ochrona danych w spoczynku

### 7.1.1 Envelope Encryption — model ogólny

Cały system stosuje dwupoziomowy model klucza:

```
KEK (Key Encrypting Key)
  ↳ przechowywany wyłącznie w HSM / Vault Transit Engine
  ↳ nigdy nie opuszcza HSM w postaci jawnej
  ↳ rotacja co 12 miesięcy (hard) lub natychmiastowa w razie incydentu

DEK (Data Encrypting Key) per obiekt chroniony
  ↳ 256-bitowy losowy klucz AES
  ↳ zaszyfrowany KEK (Vault Transit: `transit/encrypt/<key_name>`)
  ↳ przechowywany jako `ciphertext_blob` obok zaszyfrowanych danych w DB
  ↳ czas życia w pamięci RAM: tylko na czas operacji; zeroizacja po użyciu
  ↳ rotacja: generacja nowego DEK przy każdej aktualizacji danych wrażliwych
```

Schemat AES-256-GCM dla każdego szyfrowanego pola:

```
ciphertext = AES-256-GCM(key=DEK, iv=random_96bit, aad=field_context_header, plaintext)
stored = base64(iv) || "." || base64(tag) || "." || base64(ciphertext)
field_context_header = canonical_json({table, column, row_id, field_version})
```

`aad` (Additional Authenticated Data) wiąże szyfrogram z konkretnym wierszem i kolumną — przesunięcie szyfrogramu do innego wiersza zostanie wykryte.

### 7.1.2 Chronione obszary danych

| Kategoria | Zaszyfrowane kolumny | Model klucza | Dodatkowe środki |
|---|---|---|---|
| **Poświadczenia konektorów** (`connector_credential`) | `credential_encrypted` | DEK per wpis, KEK w HSM; DEK-wrapped kluczem publicznym agenta dla zadań pull-based (HPKE RFC 9180) | Logowanie dostępu do tabeli (`READ_CREDENTIAL` w audycie); rate-limit zapytań; `statement_timeout` |
| **Dane PII tożsamości** (imię, nazwisko, PESEL, mail, adres) | `pii_blob` (JSONB zaszyfrowany) | DEK per tożsamość; KEK rotowany; możliwość crypto-shreddingu (7.3.2) | Maskowanie w API (pole zwracane tylko z scope `identity:pii:read`); pseudonimy w audycie i miningu |
| **Dowody realizacji zadań ręcznych** (treść ticketu, komentarze) | `evidence_encrypted` | DEK per zadanie | |
| **Pakiety dowodowe kampanii** (Object Storage) | Szyfrowanie po stronie serwera: SSE-S3 z kluczem zarządzanym przez MinIO Vault KMS | KEK Vault | Object Lock Compliance, retencja 7 lat |
| **WAL i backupy PostgreSQL** | Szyfrowanie na poziomie systemu plików (LUKS / dm-crypt) + pgBackRest szyfruje archiwa GPG | Klucz GPG w HSM | Tape Seal dla kopii poza serwerownią |
| **Hashe łańcucha audytu** | Przechowywane jawnie (ich ochrona to integralność, nie poufność) | SHA-256 deterministyczny | Redundancja: DB + WORM + TSA |

### 7.1.3 Ochrona kluczy w HSM i Vault

```
Architektura kluczy:
  Root CA (HSM, offline)
    ↳ Intermediate CA IGA (HSM, online — mTLS PKI)
    ↳ TSA Signing Certificate (HSM)
    ↳ Ed25519 Audit Signing Key (HSM, Vault Transit / PKCS#11)
      ↳ checkpoint_signing_keyring[current, previous] — rotacja co 90 dni
    ↳ Transit KEK "iga-kek-v{N}" — rotacja co 365 dni
      ↳ DEK(credential, tożsamość PII, dowód) — zaszyfrowane KEK
```

Procedura rotacji KEK:
1. Wygenerowanie nowego KEK `v{N+1}` w HSM.
2. Re-encryption wsadowy: `SELECT ... FOR UPDATE SKIP LOCKED` po 1000 wierszy, odszyfrowanie DEK kluczem `v{N}`, zaszyfrowanie nowym `v{N+1}`, zapis w transakcji razem z `key_version_id`.
3. Po zakończeniu re-encryption: zapis `KEK v{N}` jako `RETIRED` (wciąż dostępny do odszyfrowania przez 180 dni dla kopii zapasowych), rotacja flagi `current`.
4. Po 180 dniach: `REVOKE` klucza w HSM, `DESTROYED` w metadanych.

Każde użycie HSM generuje wpis `hsm_operation_log` (operacja, key_id, actor, timestamp) archiwizowany w WORM.

### 7.1.4 Szyfrowanie w tranzycie

| Połączenie | Protokół | Minimalna wersja TLS | Uwagi |
|---|---|---|---|
| Przeglądarka → Web UI | HTTPS TLS 1.3 | TLS 1.3 (fallback 1.2 wyłączony) | HSTS max-age=31536000; includeSubDomains; preload |
| Web UI / API klient → Identity Core API / Governance Engine | HTTPS TLS 1.3 | TLS 1.3 | JWT Bearer walidowany przez JWKS z IdP |
| HR System → HR Gateway | mTLS TLS 1.3 | TLS 1.3 | Certyfikat klienta per system HR, pinning CN; alternatywnie Kafka z SASL/SCRAM + TLS |
| Wszystkie konektory automatyczne → Systemy docelowe | TLS ≥ 1.2 per system; mTLS tam gdzie system obsługuje | TLS 1.2 minimum; preferowane 1.3 | Dla starszych systemów: wymóg wpisany w `connector_config.security_policy` z zatwierdzeniem Security Officer |
| Agent On-Prem → Agent Gateway | mTLS TLS 1.3 | TLS 1.3 | Certyfikat klienta per agent (ECDSA P-256), binding fingerprint w DB |
| Wszystkie komponenty wewnętrzne (Core) | mTLS TLS 1.3 | TLS 1.3 | Certyfikaty wystawiane przez Intermediate CA IGA, rotowane automatycznie co 30 dni przez Vault PKI |
| PostgreSQL — libpq | TLS 1.3 (`sslmode=verify-full`) | TLS 1.3 | Każdy komponent ma własne konto DB z minimalnym zakresem uprawnień |
| Redis | TLS (RESP over TLS) | TLS 1.2 minimum | ACL Redis per komponent; `requirepass` + certyfikaty |
| Kafka | SASL/SCRAM-SHA-512 + TLS 1.3 | TLS 1.2 minimum | ACL Kafka: producer per topic; consumer per consumer group; brak ACL globalnych |
| Vault API | HTTPS mTLS TLS 1.3 | TLS 1.3 | Każdy komponent jako osobny AppRole lub Kubernetes Service Account |

## 7.2 Kryptograficznie weryfikowalny dziennik audytowy

### 7.2.1 Model zdarzenia audytowego

```python
class AuditEvent(ContractModel):
    event_id: UUID                        # UUIDv7 (monotoniczny)
    block_no: Annotated[int, Field(ge=0)] # numer bloku w hash-chain
    sequence_no: int                      # pozycja w bloku (0-indexed)
    occurred_at: AwareDatetime            # chwila zdarzenia biznesowego
    recorded_at: AwareDatetime            # chwila zapisu do DB (≥ occurred_at)
    actor_id: UUID | None                 # None dla zdarzeń systemowych
    actor_kind: Literal["HUMAN", "SYSTEM", "SERVICE_ACCOUNT", "AGENT"]
    actor_session_id: str | None
    actor_ip: IPv4Address | IPv6Address | None
    event_type: Code                      # hierarchiczny: "identity.status.changed"
    severity: Literal["INFO", "WARN", "AUDIT", "ALERT"]
    subject_kind: Literal["IDENTITY", "ACCOUNT", "APPLICATION", "POLICY", ...] | None
    subject_id: UUID | None
    subject_pseudo_id: str | None         # pseudonim SHA-256(salt_per_identity || subject_id) dla tożsamości
    correlation_id: UUID                  # łączy zdarzenia tej samej operacji (np. JML Joiner)
    causation_id: UUID | None             # event_id zdarzenia przyczynowego
    operation: str                        # "CREATE", "UPDATE", "DELETE", "APPROVE", "REVOKE", ...
    result: Literal["SUCCESS", "FAILURE", "PARTIAL"]
    payload: dict[str, Any]               # dane kontekstowe bez PII jawnej
    policy_snapshot_hash: Sha256Hex | None
    payload_hash: Sha256Hex               # SHA-256(canonical_json(payload))
    prev_hash: Sha256Hex                  # hash bloku poprzedniego lub "genesis" dla bloku 0
    block_hash: Sha256Hex | None          # wypełniany przez Audit Sealer po zamknięciu bloku
    merkle_proof: list[Sha256Hex] | None  # ścieżka Merkle do korzenia bloku
```

Żadne pole zawierające PII (imię, nazwisko, adres, PESEL, mail) nie może pojawić się w `payload` w postaci jawnej — wyłącznie pseudonimizowane identyfikatory. Wyjątek: zdarzenia klasy `AUDIT_TRAIL_REVEAL` generowane podczas odtajniania przez uprawnionego aktora z uprawnieniem `audit:reveal` — te zdarzenia są same szyfrowane DEK `audit-reveal-kek`.

### 7.2.2 Hash-Chain: budowa bloków

```
Blok N zawiera od 1 do BLOCK_SIZE (domyślnie 1000) zdarzeń:

  merkle_leaves[i] = SHA-256(canonical_json(event[i].payload) || event[i].occurred_at.isoformat())
  merkle_root = MerkleTree(leaves).root                   # binarne drzewo, SHA-256
  block_hash[N] = SHA-256(
      block[N-1].block_hash
      || merkle_root
      || str(event_count)
      || first_event_at.isoformat()
      || last_event_at.isoformat()
      || str(block_no)
  )
  genesis (block 0): prev_hash = SHA-256("IC-IGA-GENESIS-V1" || installation_id || initialized_at.isoformat())
```

Audit Sealer (jeden proces, serialized przez distributed lock Redis):
- Działa jako dedykowany daemon.
- Co 60 sekund (lub gdy bufor osiągnie BLOCK_SIZE): pobiera nieprzetworzonych zdarzeń `SELECT ... WHERE block_no IS NULL ORDER BY sequence_no FOR UPDATE SKIP LOCKED LIMIT BLOCK_SIZE`.
- Oblicza Merkle root i `block_hash`, zapisuje w transakcji.
- Co 100 bloków: generuje `audit_checkpoint` (patrz 7.2.3).
- Wyjście zablokowane dla innych procesów przez advisory lock Postgres (`pg_try_advisory_lock`); próba równoległego uruchomienia jest odrzucana z logowaniem alarmu.

### 7.2.3 Checkpoint — kotwiczenie zewnętrzne

```python
class AuditCheckpoint(ContractModel):
    checkpoint_id: UUID
    block_no_from: int
    block_no_to: int
    cumulative_hash: Sha256Hex             # SHA-256(block[0].block_hash || ... || block[N].block_hash)
    event_count: int
    generated_at: AwareDatetime
    ed25519_signature: bytes               # Ed25519.sign(private_key=HSM, message=SHA-256(canonical(self bez pól podpisu)))
    signing_key_id: str
    tsr: bytes | None                      # RFC 3161 TimeStampToken; None jeśli TSA niedostępna (retry 3×)
    worm_object_key: str | None            # klucz obiektu w MinIO Object Lock po eksporcie
```

Checkpoint jest walidowany przez `AuditVerifier` (uruchamiany codziennie i na żądanie przez "Verify chain" w UI):

```python
def verify_chain(from_block: int, to_block: int) -> VerificationResult:
    prev_hash = genesis_hash if from_block == 0 else get_block(from_block - 1).block_hash
    for block in iter_blocks(from_block, to_block):
        computed = sha256(prev_hash || block.merkle_root || ...)
        if computed != block.block_hash:
            return VerificationResult(ok=False, tampered_block=block.block_no, detail="block_hash_mismatch")
        for event in block.events:
            leaf = sha256(canonical_json(event.payload) || event.occurred_at.isoformat())
            if not verify_merkle(leaf, event.merkle_proof, block.merkle_root):
                return VerificationResult(ok=False, tampered_event=event.event_id, detail="merkle_proof_invalid")
        prev_hash = block.block_hash
    for chk in checkpoints_in_range(from_block, to_block):
        if not Ed25519.verify(pubkey=load_pubkey(chk.signing_key_id), message=sha256(canonical(chk_without_sig)), sig=chk.ed25519_signature):
            return VerificationResult(ok=False, tampered_checkpoint=chk.checkpoint_id, detail="ed25519_invalid")
        if chk.tsr:
            verify_rfc3161(chk.tsr, sha256(canonical(chk_without_sig)))
    return VerificationResult(ok=True, blocks_verified=to_block - from_block + 1)
```

### 7.2.4 WORM i archiwizacja

Eksport do WORM (MinIO Object Lock, tryb Compliance):
- Co 24 h: Audit Sealer eksportuje bloki ostatniej doby jako `audit_export_{date}.jsonl.gz` + `audit_export_{date}.manifest.json` do bucket `audit-worm`.
- Object Lock Compliance Mode: `x-amz-object-lock-retain-until-date = now + 7 years`, `x-amz-object-lock-mode = COMPLIANCE`.
- Bucket ma włączone MFA Delete i jest dostępny tylko przez osobne konto serwisowe z uprawnieniem tylko do PUT; usunięcie wymaga Root AWS account lub ekwiwalentu MinIO admin.
- Checkpointy eksportowane natychmiast po wygenerowaniu.
- Backup cross-site: rsync zaszyfrowanych plików na serwer w drugiej serwerowni (dedykowany, bez połączenia sieciowego poza kanałem replikacji).

### 7.2.5 Zdarzenia audytowe — wymagane kategorie

Każde działanie mutujące musi generować zdarzenie audytowe. Poniżej lista wymaganych typów:

| `event_type` | Wyzwalacz | Obowiązkowe pola `payload` |
|---|---|---|
| `identity.created` | Joiner lub import HR | `identity_type`, `hr_person_id`, `department_id` |
| `identity.status.changed` | Mover, Leaver, Rehire, administracyjna zmiana statusu | `old_status`, `new_status`, `reason`, `hr_event_id` |
| `account.provisioned` / `account.deprovisioned` | Zadanie provisioningowe SUCCEEDED | `application_id`, `native_id`, `provisioning_task_id`, `connector_type`, `idempotency_key` |
| `account.state.changed` | Transition automatu stanu konta | `old_state`, `new_state`, `trigger` |
| `entitlement_assignment.granted` / `revoked` | Provisioning, JML, kampania, reconciliation | `entitlement_id`, `source_kind`, `source_id`, `via_role_id` |
| `access_request.submitted` / `approved` / `rejected` | Workflow wniosku | `request_id`, `items_count`, `sod_decision`, `risk_score`, `approver_id` |
| `sod_violation.detected` / `mitigated` / `exception_granted` | SoD evaluator, workflow wyjątku | `rule_id`, `policy_snapshot_hash`, `identity_id_pseudo`, `mitigation_type` |
| `certification.campaign_state_changed` | Kampania | `campaign_id`, `old_state`, `new_state` |
| `certification.item_decided` | Decyzja reviewera | `item_id`, `decision`, `decided_by_pseudo`, `step_up_mfa` |
| `certification.rubber_stamp_suspected` | Heurystyka | `reviewer_pseudo`, `decisions_in_window`, `window_seconds` |
| `manual_task.state_changed` | Fulfiller inbox | `task_id`, `old_state`, `new_state`, `actor_pseudo`, `evidence_hash` |
| `reconciliation.delta_detected` | Pętla uzgadniania | `delta_type`, `application_id`, `native_id`, `risk_level`, `reaction` |
| `policy.rule.published` / `retired` | Workflow dual-control reguły | `rule_id`, `approved_by_security_pseudo`, `approved_by_compliance_pseudo`, `revision_id` |
| `mining_run.started` / `completed` / `promoted` | Mining Engine | `run_id`, `algorithm_ids`, `seed`, `data_hash`, `candidates_count`, `coverage_pct` |
| `role.promoted_from_mining` | Promocja kandydata | `candidate_id`, `role_id`, `approved_by_pseudo`, `sod_check_passed` |
| `audit_chain.verified` / `tampered_detected` | Weryfikacja łańcucha | `blocks_verified`, `tampered_block_no` (jeśli dotyczy) |
| `credential.accessed` | Odszyfrowanie poświadczeń konektora | `credential_id`, `application_id`, `actor_id`, `purpose` |
| `break_glass.activated` / `deactivated` | Konto break-glass | `target_application_id`, `justification`, `approver_pseudo` |

## 7.3 Zgodność z regulacjami

### 7.3.1 Macierz wymagań regulacyjnych

| Wymaganie | Regulacja | Mechanizm IC IGA | Dowód |
|---|---|---|---|
| Kontrola dostępu oparta na regułach z separacją obowiązków | SOX §404 | Policy Engine SoD (sekcja 6), preventive + detective, wyjątki z twardą datą | Raport naruszeń SoD, dziennik wyjątków, kampanie HIGH_RISK |
| Dowód przeprowadzenia i kompletności przeglądu dostępu | SOX §404, SOC 2 CC6.2 | Pakiet dowodowy kampanii (4.4.5) z podpisem Ed25519 i TSA RFC 3161 | Plik `manifest.json` + `manifest.sig` + `manifest.tsr` |
| Zarządzanie cyklem życia dostępu uprzywilejowanego | SOX §404, SOC 2 CC6.3 | JML Leaver emergency (SLO-04: p95 < 120 s), kampanie HIGH_RISK dla entitlements CRITICAL, break-glass | Zdarzenie `account.deprovisioned` z `occurred_at − hr_event.received_at`, dziennik break-glass |
| Niezaprzeczalny dziennik audytowy | SOX §404, SOC 2 CC7.2 | Hash-chain SHA-256 + Ed25519 + RFC 3161 + WORM | Skrypt `verify.py` z pakietu dowodowego |
| Kontrola zmian systemu IGA | SOX §404, ISO 27001 A.12.1.2 | Wersjonowanie ról, reguł, konektorów; dual-control publikacji; `policy_snapshot_hash` w każdej ocenie | Tabele `policy_revision`, `role_revision`, dziennik zdarzeń `policy.rule.published` |
| Zarządzanie dostępem do danych osobowych | RODO Art. 5, 25 | Pseudonimizacja w audycie i miningu, `pii_blob` zaszyfrowany, RBAC na zakres `identity:pii:read`, crypto-shredding | Sekcja 7.3.2 |
| Prawo do usunięcia vs retencja audytu | RODO Art. 17 vs SOX/SOC 2 | Crypto-shredding: zniszczenie DEK per tożsamość eliminuje możliwość odszyfrowania PII przy zachowaniu pseudonimizowanych danych audytu | Sekcja 7.3.2 |
| Minimalizacja danych | RODO Art. 5(1)(c) | Audyt bez jawnego PII; mining na pseudonimach; pola PII zwracane tylko z dedykowanym scope | Przegląd DPIA |
| Bezpieczeństwo przetwarzania | RODO Art. 32 | AES-256-GCM, mTLS, TLS 1.3, HSM, STRIDE (sekcja 1.4) | Wynik skanu podatności, pentest roczny |
| Zarządzanie incydentami | NIS2 Art. 21, KSC | SIEM integration (alerty z Reconciliation i Policy Engine), playbooki incydentowe, SLA zgłoszenia < 24 h (early warning) | Dziennik incydentów, zdarzenia audytowe klasy `ALERT` |
| Minimalne uprawnienia administracyjne IGA | SOC 2 CC6.1, ISO 27001 A.9.2.3 | RBAC administracyjny scope-based; dual-control dla operacji krytycznych; zakaz self-approval (trigger DB) | Przegląd uprawnień administracyjnych co kwartał |
| Ciągłość działania | SOC 2 A1, ISO 27001 A.17 | RPO 0 (sync replikacja), RTO 60 s (Patroni auto-failover), backupy WAL, testy recovery | Raporty testów odtworzenia |
| Ochrona poświadczeń | ISO 27001 A.9.4.3 | Envelope encryption, DEK w pamięci tylko na czas operacji, zeroizacja | Skan logów, pentest |

### 7.3.2 RODO: Prawo do usunięcia a retencja audytu — Crypto-Shredding

Konflikt: SOX i SOC 2 wymagają retencji dziennika audytowego przez 7 lat. RODO Art. 17 daje podmiotowi prawo do usunięcia danych osobowych.

Rozwiązanie: crypto-shredding per tożsamość.

```
Dla każdej tożsamości przechowywany jest osobny DEK PII:
  identity_pii_key = generowany przy tworzeniu tożsamości
  Wszystkie PII (imię, nazwisko, mail, PESEL, adres, ...) zaszyfrowane tym kluczem w polu pii_blob.
  W audycie tożsamość pojawia się wyłącznie jako subject_pseudo_id = HMAC-SHA256(pseudonym_salt, identity_id)
  pseudonym_salt per instalacja (rotowany co rok, poprzednie wartości przechowywane w HSM)

Realizacja prawa do usunięcia:
  1. Weryfikacja podstawy prawnej (SOX retencja = uzasadniony interes prawny dla wpisów do 7 lat).
  2. Tożsamość oznaczana jako ERASURE_REQUESTED; PII w pii_blob zastępowane tokenem erasure:
       pii_blob = AES-256-GCM(key=DEK, plaintext='{"erased_at":"...", "reason":"GDPR_RIGHT_TO_ERASURE"}')
  3. Niszczenie DEK PII w Vault: `vault kv delete identity-pii-keys/{identity_id}` + `vault lease revoke`.
  4. Rekord identity pozostaje (zachowanie relacji kont, ról, historii uprawnień bez PII).
  5. Rekord audytowy zawiera wyłącznie subject_pseudo_id — po zniszczeniu DEK PII odwzorowanie pseudonimu → tożsamość staje się niemożliwe bez HSM.
  6. W raportach dla audytora: tożsamość pojawia się jako "[ERASED]" z datą usunięcia.
  7. Zdarzenie: audit_event(identity.pii_erased, reason, actor_id_pseudo, legal_basis_exception_ids[]).
```

Ograniczenia: zniszczenie DEK nie kasuje historii uprawnień (identyfikatory uprawnień, role, daty) — tylko PII jest nieosiągalne. Jest to wystarczające z punktu widzenia RODO przy uzasadnionym interesie prawnym (art. 17(3)(b)).

### 7.3.3 Pseudonimizacja w mining engine

Silnik miningu operuje na macierzy `(identity_pseudo_id) × (entitlement_id)`. Mapowanie `identity_pseudo_id → identity_id` jest dostępne wyłącznie w warstwie Governance Engine przez operację `mining:reveal` (logowaną w audycie). Mining Engine nie ma bezpośredniego dostępu do tabeli `identity` — dane są eksportowane jako widok pseudonimizowany z repliki read-only.

## 7.4 Ochrona poświadczeń konektorów

```mermaid
sequenceDiagram
    autonumber
    participant ADMIN as Administrator IGA
    participant API as Identity Core API
    participant VLT as Vault (Transit Engine)
    participant DB as PostgreSQL
    participant PE as Provisioning Engine
    participant CN as Konektor

    ADMIN->>API: PUT /applications/{id}/credentials {username, password, ...}
    API->>API: Walidacja: aktor ma scope connector:credentials:write, aplikacja jego
    API->>VLT: transit/encrypt/iga-kek-v3(plaintext=DEK_32bytes_random)
    VLT-->>API: ciphertext_blob
    API->>API: AES-256-GCM(key=DEK, aad=context_header, plaintext=json(credentials))
    API->>DB: INSERT connector_credential(application_id, ciphertext, ciphertext_blob, iv, tag, key_version_id)
    API->>DB: audit_event(credential.stored, actor, application_id) — bez wartości!
    ADMIN-->>API: 204 No Content (poświadczenia nie są zwracane nigdy)

    PE->>DB: SELECT ciphertext, ciphertext_blob, key_version_id FROM connector_credential WHERE application_id=X
    PE->>VLT: transit/decrypt/iga-kek-v3(ciphertext_blob) → DEK (plaintext, w pamięci)
    PE->>PE: AES-256-GCM.decrypt(DEK, iv, tag, ciphertext) → credentials (w pamięci)
    PE->>DB: audit_event(credential.accessed, actor=provisioning-worker, application_id, purpose=provisioning_task_id)
    PE->>CN: execute(operation, payload, credentials) — w pamięci, nie na dysku
    PE->>PE: del credentials; zeroizacja (ctypes.memset)
```

Poświadczenia nigdy nie są logowane. Każdy worker ma osobne konto AppRole Vault z polityką `update` na `transit/decrypt/iga-kek-v3` i bez prawa odczytu klucza samego. Dostęp do tabeli `connector_credential` jest przyznany wyłącznie workernemu Provisioning Engine i Agent Gateway.

## 7.5 Ochrona integralności danych biznesowych

### 7.5.1 Tabele append-only i blokady

Tabele, które nie mogą być modyfikowane wstecznie (`audit_event`, `audit_block`, `audit_checkpoint`, `entitlement_assignment_history`, `certification_decision`), mają:

1. `REVOKE UPDATE, DELETE ON TABLE audit_event FROM iga_app_role` — żaden komponent aplikacyjny nie ma uprawnień UPDATE/DELETE.
2. Trigger `BEFORE UPDATE OR DELETE ON audit_event EXECUTE FUNCTION deny_modification()` — dodatkowe zabezpieczenie.
3. Partycjonowanie zakresowe po `occurred_at` (miesięczne partycje); partycje starsze niż `retention_months` są eksportowane do WORM i detachowane, a następnie droppowane (nie truncated — DROP partycji zapisywane w audycie).
4. Kolumna `block_no` jest NOT NULL dla wpisów po zamknięciu bloku; trigger odrzuca UPDATE `block_no IS NULL → NOT NULL` z poziomu innego niż Audit Sealer (identyfikowany przez `session_user`).

### 7.5.2 SCD Type 2 dla historii uprawnień

`entitlement_assignment` przechowuje bieżący stan. Historia jest zapisywana w `entitlement_assignment_history` (append-only, SCD2): przy każdej zmianie stanu wstawiany jest nowy wiersz z `valid_from`, `valid_to`, `changed_by`, `change_reason`. Nie wolno UPDATE/DELETE wierszy historii. Audyt pokrywa każde wejście.

### 7.5.3 Wersjonowanie ról, uprawnień i reguł

- Każda zmiana definicji roli (`role_revision`) lub reguły polityki (`policy_revision`) tworzy nowy niezmienny rekord z poprzednim jako `parent_revision_id`.
- Aktywna wersja oznaczona jest flagą `is_current = true` (blokada: max jeden `is_current` per `role_id` — constraint UNIQUE z partial index `WHERE is_current`).
- `policy_snapshot_hash` w każdej ocenie SoD i w każdym wpisie audytowym związanym z decyzją pozwala odtworzyć dokładnie, jakie reguły obowiązywały w danej chwili.
- Konektory: `connector_config` wersjonowany analogicznie; wdrożenie nowej wersji konfiguracji poprzedzone testem `test_connection`.
