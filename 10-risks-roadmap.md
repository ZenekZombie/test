# 10. Ryzyka, Ograniczenia Prototypu i Roadmap

## 10.1 Ryzyka Architektoniczne i Projektowe

### 10.1.1 Macierz ryzyk

| ID | Ryzyko | Prawdopodobieństwo | Wpływ | Mitygacja | Właściciel |
|---|---|---|---|---|---|
| R-01 | Latencja oceny SoD przekracza SLO-01 przy kumulacji reguł (>1 000) | SREDNIE | WYSOKI | Profiling na zbiorze 1000 reguł przed cutoverem; aktualizacja kompilatora bitsetowego; cache skompilowanych reguł per revision | Arch Lead |
| R-02 | Awaria HSM / Vault blokuje cały provisioning (DEK niedostępny) | NISKIE | KRYTYCZNY | Vault HA (3 węzły, auto-unseal z cloud KMS lub HSM backup); cache zaszyfrowanych DEK per worker (TTL 60 s, tylko dla provisioningu; nie dla odczytu PII) | Security Architect |
| R-03 | Eksplozja tabeli `audit_event` (>500M wierszy w 3 lata przy gwałtownym wzroście zdarzeń) | SREDNIE | WYSOKI | Partycjonowanie z automatycznym detachowaniem i archiwizacją do WORM; monitoring wzrostu per partycja; alert przy > 200M wierszy / miesiąc | DBA |
| R-04 | Konektor LDAP / AD nie zwraca spójnych wyników (replication lag) | WYSOKIE | SREDNI | grace_minutes = 15 dla kont klasy A (atrybuty repliki nie są jeszcze propagowane); weryfikacja zawsze po primary DC; alert przy lag > 30 s w monitoringu LDAP | Connector Team |
| R-05 | Mining na 100k tożsamości × 450k uprawnień przekracza 4 h (SLO-13) | SREDNIE | SREDNI | Partycjonowanie macierzy per aplikacja; M3 z próbkowaniem dla n>50k; M9 wyłączony domyślnie dla pełnego zakresu (opt-in); benchmark przed wdrożeniem | Mining Team |
| R-06 | Dual-control publikacji reguł SoD staje się wąskim gardłem (Security Officer niedostępny) | WYSOKIE | SREDNI | Delegacja per règle severity (LOW → jeden approver; HIGH/CRITICAL → dual-control); break-glass z timeoutem 4 h i audytem; SLA dla Security Officer | Compliance |
| R-07 | Agent on-premise kompromitacja (przejęcie certyfikatu) → wykonanie operacji spoza swojego application_id | NISKIE | KRYTYCZNY | Binding fingerprint + application_id w DB; fencing_token leasingowy; fencing sprawdzany przez Gateway przed akceptacją wyniku; alert przy rozbieżności fingerprint | Security Architect |
| R-08 | Fluktuacja składu grupy realizatorów (fulfillers) → zadania bez realizatora | WYSOKIE | WYSOKI | Grupa, nie osoba, jako assignee; eskalacja po SLA do managera grupy; fallback: wniosek wraca do wnioskodawcy z opcją wycofania; monitoring SLA per aplikacja | Operations |
| R-09 | Reconciliation z błędnym eksportem (np. zmiana base DN) prowadzi do masowego revoke | SREDNIE | KRYTYCZNY | Sanity floor check (sekcja 4.3.3): przerwanie diffowania przy wolumenie < 50% poprzedniego; wymagane dual-control potwierdzenie przed przetworzeniem > 10 000 deltas AUTO_REVERT | Reconciliation Team |
| R-10 | RODO: wniosek o usunięcie tożsamości z 7-letnim dziennikiem SOX | WYSOKIE | WYSOKI | Crypto-shredding (sekcja 7.3.2): zniszczenie DEK PII eliminuje możliwość re-identyfikacji przy zachowaniu pseudonimizowanych wpisów audytu; ocena prawna per jurysdykcja | DPO + Legal |
| R-11 | Role mining produkuje kandydatów promowanych automatycznie z naruszeniem SoD | NISKIE | KRYTYCZNY | Dla każdego kandydata: SoD check przed wyświetleniem w UI; blokada promocji kandydata z naruszeniem bez wyjątku; `auto_promote_forbidden=true` dla EXCEPTION | Policy Team |
| R-12 | Outbox Relay zatrzymuje się cicho (deadlock / OOM) → zdarzenia domenowe nie docierają do konsumentów | SREDNIE | WYSOKI | Healthcheck `/health` z metryką `outbox_max_lag_seconds`; alert przy lag > 30 s; automatyczny restart przez systemd/k8s; monitoring tabeli outbox per partition | Platform |
| R-13 | Dependency injection w NiceGUI — wyciek stanu między sesjami użytkowników (shared global state) | SREDNIE | WYSOKI | Każda sesja NiceGUI ma własny kontekst (`ui.run(storage_secret=...)`); stan per user w `ui.storage.user`; brak globalnych zmiennych mutowalnych poza singletonami DB/broker | UI Lead |

---

## 10.2 Ograniczenia Prototypu

### 10.2.1 Wykaz uproszeń prototypu vs architektura docelowa

| Obszar | Prototyp (co jest uproszczone) | Architektura docelowa (co zastąpi) | Konsekwencja dla testowania |
|---|---|---|---|
| **Persystencja** | SQLite, brak partycjonowania, brak SKIP LOCKED | PostgreSQL 15+ z Patroni, partycjonowanie, advisory locks | Testy scenariuszy S1–S12 działają; brak testów HA i równoległości workerów |
| **Broker wiadomości** | `asyncio.Queue` in-process (InProcessBroker) | Apache Kafka 3.6 z 5 brokerami RF=3 | Zdarzenia znikają po restarcie; brak at-least-once poza Outbox |
| **HSM / Vault** | Symulowany Vault (`FakeVault` z kluczami w pamięci), AES-256-GCM w bibliotece `cryptography` | HashiCorp Vault + HSM PKCS#11, Transit Engine, PKI CA | Kryptografia jest demonstrowana i testowana; klucze nie są chronione hardware |
| **Uwierzytelnianie UI** | Uproszczone logowanie sesyjne z predefiniowanymi rolami (`admin`, `approver`, `fulfiller`, `auditor`) | OIDC/SAML z korporacyjnym IdP, step-up MFA | Kontrola dostępu role-based działa; brak integracji z AD/LDAP |
| **Konektory** | Mock SCIM, Mock API, SQLite jako target, symulacja agenta pull-based | Rzeczywiste konektory do AD, SAP, Oracle, Workday itp. | Wszystkie wzorce (retry, DLQ, idempotency) są demonstrowane; brak real integracji |
| **Notyfikacje e-mail** | Mock SMTP (zapis do tablicy notifikacji w UI, logi) | Prawdziwy SMTP / e-mail gateway z szablonami | Przepływ powiadomień widoczny w UI jako [SYMULACJA SMTP] |
| **TSA / RFC 3161** | Symulowany znacznik czasu (lokalny timestamp + mock signature) | Zewnętrzna TSA lub wewnętrzna EJBCA z timestampingiem | Hash-chain i podpisy Ed25519 są prawdziwe; TSA token jest mockiem |
| **WORM / Object Storage** | Lokalny katalog `data/worm/` z symulowanym Object Lock (niemodyfikowalne pliki przez chmod) | MinIO z Object Lock Compliance Mode | Eksport plików audytu działa; brak hardware-enforced immutability |
| **Role Mining wydajność** | Domyślnie ≤ 300 tożsamości; `--scale 100k` generuje dane i uruchamia M1, M2, M3, M10 z raportem czasu | Dedykowany klaster 32–64 vCPU, numpy optymalizacje bitsetowe, partycjonowanie per aplikacja | Scenariusze S10a–S10i testowane na rzeczywistych danych; skala 100k: benchmarki, bez pełnego UI mining |
| **Audit chain sealer** | Synchroniczny sealer w tym samym procesie, blokady przez asyncio.Lock | Dedykowany daemon z distributed lock Redis | Hash-chain jest prawdziwy; sekwencyjność gwarantowana przez single-process |
| **PostgreSQL-specific features** | SQLite nie ma FOR UPDATE SKIP LOCKED → symulacja przez in-memory mutex | SKIP LOCKED w PostgreSQL dla concurrent workers | Nie testuj współbieżności workerów na prototypie |
| **TLS / mTLS** | HTTP (brak TLS) dla lokalnego prototypu; wszystkie połączenia lokalhost | TLS 1.3 everywhere, mTLS dla konektorów i agentów | Bezpieczeństwo transportu nie jest testowane; kryptografia danych działa |
| **ITSM integracja** | Mock: zdarzenie zapisywane do UI i logów jako [ITSM MOCK] | REST do ServiceNow / Jira SM | Przepływ eskalacji i ticketów widoczny w UI |

### 10.2.2 Symulowane elementy w UI

Każdy element symulowany w UI jest oznaczony etykietą `[SYM]` lub `[MOCK]` w interfejsie:

- `[MOCK SMTP]` — powiadomienie e-mail wyświetlone w UI zamiast wysłane
- `[MOCK TSA]` — znacznik czasu RFC 3161 zastąpiony lokalnym podpisem
- `[MOCK ITSM]` — ticket zapisany w bazie, nie wysłany do ServiceNow
- `[SYM CONNECTOR]` — konektor SCIM/API/DB jest mockiem z konfigurowalnymi opóźnieniami i błędami
- `[SYM AGENT]` — agent pull-based symulowany jako background task w tym samym procesie
- `[SYM TIMEOUT]` — scenariusz S7 symuluje timeout konektora przez `asyncio.sleep(30)` + `raise TimeoutError`

### 10.2.3 Elementy w pełni działające w prototypie

Nie są to atrap — działają identycznie jak w produkcji:
- Hash-chain SHA-256 + podpisy Ed25519 + weryfikacja łańcucha
- Silnik SoD (kompilator DSL + bitsetowy ewaluator) — pełne reguły na danych testowych
- JML automaty stanów (Joiner, Mover, Leaver) z idempotencją i edge-case'ami
- Workflow wniosków z dynamicznymi ścieżkami akceptacji
- Kampanie recertyfikacyjne z eskalacjami i pakietem dowodowym
- Manual Fulfillment Connector z pełnym cyklem `OPEN → DONE` + diff CSV
- Reconciliation diff (FULL i DELTA) z silnikiem reakcji
- Role Mining M1–M10 z kaskadą, ensemble i gwarancją pokrycia
- Audit log z filtrowaniem, eksportem, przyciskiem "Verify chain" i "Zasymuluj manipulację"
- Dane testowe S1–S12 + S10a–S10i + generator skali

---

## 10.3 Roadmap

### 10.3.1 Faza 0 — Prototyp (stan bieżący)

**Cel**: Demonstracja wszystkich kluczowych funkcjonalności na wbudowanych danych testowych. Weryfikacja architektury.

**Scope**: wszystkie zakładki UI, scenariusze S1–S12, role mining M1–M10, SDD w aplikacji.

**Ograniczenia**: patrz 10.2. Uruchamialny jako `python main.py` bez zewnętrznych zależności poza PyPI.

### 10.3.2 Faza 1 — MVP On-Premise (3–6 miesięcy od decyzji)

**Cel**: Wdrożenie do środowiska DEV/TEST z prawdziwą infrastrukturą.

**Deliverables**:
- Podmiana SQLite → PostgreSQL 15 z Patroni HA
- Podmiana InProcessBroker → Apache Kafka 3.6 (lub RabbitMQ 3.12 z quorum queues)
- Wdrożenie HashiCorp Vault z HSM PKCS#11 (lub bez HSM dla DEV)
- Implementacja konektora AD/LDAP (priorytet: główny katalog tożsamości)
- OIDC integracja z korporacyjnym IdP (Keycloak lub Azure AD On-Prem ADFS)
- MinIO z Object Lock Compliance
- Helm chart + playbooki Ansible dla wdrożenia na RKE2
- Testy load: k6 dla SLO-01, SLO-02, SLO-04

**Zmiany kodu (szacunkowe)**:
- Brak zmian w logice biznesowej (architektura zaprojektowana pod podmianę)
- Implementacja `KafkaBroker(MessageBroker)` i `PostgreSQLAdvisoryLock`
- Migracje Alembic aktywujące `PARTITION BY RANGE`, `SKIP LOCKED`, `CONCURRENTLY` dla indeksów

### 10.3.3 Faza 2 — Pilot produkcyjny (6–12 miesięcy)

**Cel**: 10 000 tożsamości, 20 aplikacji (5 zautomatyzowanych + 15 manualnych), pełne JML dla jednej spółki.

**Deliverables**:
- Konektory: SAP (SCIM 2.0 lub BAPI), Oracle HRMS (LDAP/DB), systemy e-mail
- Pełna integracja ITSM (ServiceNow lub Jira SM)
- Alerty SIEM (Splunk lub Elastic SIEM)
- Pierwsze kampanie recertyfikacyjne dla systemów finansowych (SOX scope)
- Raportowanie SOX: lista kont z uprawnieniami finansowymi, historia zmian, wyjątki SoD
- PKI wewnętrzne dla mTLS konektorów (Vault PKI Secrets Engine)
- Szkolenie realizatorów Manual Fulfillment (50+ aplikacji manualnych)
- Pentest zewnętrzny (zakres: API, broker, Vault, Agent)

### 10.3.4 Faza 3 — Pełne wdrożenie (12–24 miesiące)

**Cel**: 100 000 tożsamości, 650 aplikacji, SOX/SOC 2/ISO 27001 compliance.

**Deliverables**:
- Wszystkie 180 konektorów automatycznych (AD, SAP, Oracle, Salesforce, SCIM SaaS, DB, bespoke)
- Agent on-premise dla 50 aplikacji w sieciach izolowanych (OT, produkcja)
- Pełny role mining w trybie produkcyjnym (harmonogram nocny, wyniki do przeglądu)
- SOC 2 Type II audit z pakietami dowodowymi kampanii
- Integracja z SIEM → korelacja zdarzeń IGA z alertami sieciowymi (detekcja lateral movement)
- Pseudonimizacja i crypto-shredding dla RODO (rejestr żądań usunięcia)
- NIS2: procedury zgłoszenia incydentu, playbooki IGA (blokada kont po incydencie < 15 min)
- Replikacja między serwerowniami (A–B + arbitraż C) — testy chaos co kwartał
- Role Mining: pełny przebieg 100k × 450k, M1–M10, w < 4 h (SLO-13)
- Wydajność: SLO-01 do SLO-15 potwierdzone testami obciążeniowymi

### 10.3.5 Backlog — poza zakresem obecnego SDD

Poniższe elementy są świadomie odłożone i nie są blokerami dla Fazy 1–3, ale warto je zaplanować:

| Funkcjonalność | Priorytet | Uzasadnienie odłożenia |
|---|---|---|
| PAM integration (CyberArk / BeyondTrust) | WYSOKI | Wymaga oddzielnego projektu integracyjnego; IGA dostarcza dane o właścicielach kont uprzywilejowanych, nie zarządza samymi sesjami | 
| Machine Learning risk scoring (UEBA) | SREDNI | Wymaga 6–12 miesięcy danych produkcyjnych; baseline ręczny (atrybuty ryzyka) jest wystarczający na start |
| Self-service portal mobilny | NISKI | Priorytetem jest desktop; mobile CSS przez NiceGUI responsive jest możliwe jako ulepszenie |
| Federacja wielofirmowa (holding) | SREDNI | Wymaga modelu multi-tenant w DB; bieżący model zakłada jeden `company_scope` per instalacja |
| API REST publiczne (dla integracji zewnętrznych klientów) | SREDNI | Istniejące API jest "internal"; wymaga wersjonowania, throttlingu, developer portal |
| Role mining ML (poza M1–M10) | NISKI | Ensemble M1–M10 jest wystarczający dla typowych danych korporacyjnych; ML modele wymagają danych labelowanych |
| Delegacja zarządzania per oddział (delegated administration) | WYSOKI | Wymagane dla holdingów; model ABAC na `org_unit_path` jest gotowy, brakuje UI i polityk delegacji |
| Certyfikaty dostępu czasowego (Just-In-Time) | WYSOKI | Integracja z PAM workflow; `valid_from`/`valid_to` w modelu jest gotowe, brakuje UI i automatyzacji |

---

## 10.4 Protokół Change Management dla SDD

SDD jest dokumentem żywym. Każda zmiana architektoniczna musi przejść przez:

1. **ADR (Architecture Decision Record)**: krótki dokument (kontekst → decyzja → konsekwencje → odrzucone alternatywy) zapisany jako `docs/adr/ADR-{NNN}-{title}.md`.
2. **Aktualizacja SDD**: odpowiednia sekcja aktualizowana razem z ADR; wersja dokumentu (`sdd_version` w nagłówku każdego pliku) inkrementowana.
3. **Weryfikacja diagramów Mermaid**: każda zmiana diagramu przechodzi przez CI (`mermaid --validate`) przed merge.
4. **Audyt zmiany SDD**: każdy merge do `main` gałęzi dokumentacji generuje wpis w `docs/CHANGELOG.md` z datą, autorem i numerem ADR.
5. **Przegląd kwartalny**: cały SDD przeglądany raz na kwartał przez Principal Architect i Security Officer; przestarzałe sekcje flagowane jako `[DEPRECATED]` przed usunięciem.
