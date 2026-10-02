# 1. Executive Summary & Architectural Drivers

## 1.1 Misja systemu

IC IGA jest platformą klasy Identity Governance and Administration przeznaczoną do wdrożenia wyłącznie On-Premise, której zadaniem jest utrzymanie **jednego, weryfikowalnego źródła prawdy o tym, kto, do czego, na jakiej podstawie i od kiedy ma dostęp** w organizacji liczącej 100 000 aktywnych tożsamości. System realizuje trzy nierozłączne funkcje:

1. **Administracja** (Identity Administration): automatyczny cykl życia Joiner–Mover–Leaver sterowany zdarzeniami z systemów kadrowych, provisioning i deprovisioning kont oraz uprawnień w ok. 650 systemach docelowych (180 zintegrowanych konektorami automatycznymi, 470 obsługiwanych w trybie Manual Fulfillment).
2. **Nadzór** (Governance): wnioski o dostęp z dynamicznymi ścieżkami akceptacji, prewencyjna i detekcyjna kontrola Separation of Duties, kampanie recertyfikacyjne, zarządzanie wyjątkami z twardą datą wygaśnięcia.
3. **Dowodzenie zgodności** (Assurance): kryptograficznie weryfikowalny dziennik audytowy (hash-chain + podpisy Ed25519 + kotwiczenie RFC 3161), pakiety dowodowe dla audytorów SOX / SOC 2 / ISO 27001, mechanizmy pseudonimizacji zgodne z RODO oraz raportowanie incydentów na potrzeby NIS2 / KSC.

Granice systemu są jednoznaczne: IC IGA **nie jest** dostawcą tożsamości (IdP), nie realizuje uwierzytelniania użytkowników końcowych do aplikacji biznesowych ani nie zastępuje systemu PAM. Integruje się z nimi jako źródło polityki i konsument zdarzeń.

## 1.2 Kluczowe wskaźniki architektoniczne (SLO / SLA)

Wskaźniki poniżej są zobowiązaniami projektowymi. Każdy ma zdefiniowaną metodę pomiaru, budżet błędu i komponent odpowiedzialny. Wartości p95/p99 mierzone są po stronie serwera (server-side latency) w oknie 30-dniowym.

| ID | Wskaźnik | Cel (SLO) | Metoda pomiaru | Budżet błędu (30 dni) | Komponent |
|---|---|---|---|---|---|
| SLO-01 | Latencja oceny SoD dla pojedynczego wniosku (koszyk ≤ 10 pozycji, ≤ 100 aktywnych reguł) | p95 < 5 ms, p99 < 20 ms | Histogram `sod_eval_duration_seconds` w Policy/SoD Evaluator | 0,5 % żądań powyżej p99 | Policy/SoD Evaluator |
| SLO-02 | Latencja oceny SoD dla pełnego koszyka (≤ 50 pozycji, ≤ 500 reguł) | p95 < 150 ms | jak wyżej, etykieta `scope=basket` | 0,5 % | Policy/SoD Evaluator |
| SLO-03 | Throughput przetwarzania zdarzeń JML z HR | 20 zdarzeń/s w trybie ciągłym, 200 zdarzeń/s w trybie burst przez 60 min | Licznik `jml_events_processed_total` / okno 1 min | Lag konsumenta Kafka > 10 000 wiadomości przez > 15 min = naruszenie | Identity Core, Governance Engine |
| SLO-04 | Czas od zdarzenia HR Leaver (emergency) do wykonania wszystkich operacji blokujących konta w systemach automatycznych | p95 < 120 s, p99 < 300 s | Różnica `provisioning_task.completed_at - hr_event.received_at` | 1 % zdarzeń powyżej p99 | Provisioning Engine |
| SLO-05 | Czas od zdarzenia HR Joiner do zakończenia provisioningu ról birthright (systemy automatyczne) | p95 < 15 min | jak wyżej | 2 % | Provisioning Engine |
| SLO-06 | Throughput operacji provisioningowych (create/modify/disable/delete) | 50 op/s ciągłe (3 000/min), 150 op/s burst | Licznik `provisioning_ops_completed_total` | Opóźnienie kolejki > 30 min = naruszenie | Provisioning Engine, konektory |
| SLO-07 | Czas pełnej pętli uzgadniania (full reconciliation) dla wszystkich 180 konektorów automatycznych, 1,5 mln kont, 5 mln uprawnień | < 6 h w oknie nocnym (22:00–04:00) | `reconciliation_run.finished_at - started_at` | 2 przekroczenia / 30 dni | Reconciliation Service |
| SLO-08 | Czas przyrostowej pętli uzgadniania (delta) dla pojedynczego konektora klasy A (AD, SAP) | < 15 min | jak wyżej, etykieta `mode=delta` | 5 % | Reconciliation Service |
| SLO-09 | Dostępność API zarządczego (Identity Core API, Governance API) | 99,95 % miesięcznie (≤ 21 min 54 s niedostępności / miesiąc) | Sondy syntetyczne co 15 s z 3 lokalizacji sieci wewnętrznej, sukces = HTTP 2xx w < 2 s | 21 min 54 s | Cały stos |
| SLO-10 | Dostępność przyjmowania zdarzeń HR (ingress) | 99,99 % (zdarzenia buforowane w Kafka, akceptowalny lag przetwarzania) | Metryka producenta HR Gateway | 4 min 19 s | HR Gateway, Kafka |
| SLO-11 | Czas publikacji zdarzenia z Outbox do brokera | p99 < 2 s | `outbox_publish_lag_seconds` | 0,1 % | Outbox Relay |
| SLO-12 | Czas weryfikacji integralności łańcucha audytowego za 24 h (ok. 1,4 mln bloków) | < 10 min | Job `audit_chain_verify` | 1 nieudana weryfikacja / 30 dni = incydent P1 | Audit Service |
| SLO-13 | Czas wykonania pełnego role miningu w trybie AUTO dla 100 000 tożsamości × 450 000 definicji uprawnień | < 4 h (M1, M2, M3, M5, M6, M7, M10), < 12 h z M9 | `mining_run.finished_at - started_at` | Brak budżetu, job wsadowy | Mining Engine |
| SLO-14 | RPO bazy danych | 0 s (replikacja synchroniczna do co najmniej jednej repliki w drugiej serwerowni) | Monitoring `pg_stat_replication.sync_state` | Przełączenie na async dłużej niż 5 min = incydent | PostgreSQL / Patroni |
| SLO-15 | RTO (awaria węzła primary PostgreSQL) | < 60 s automatyczny failover, < 15 min pełne przywrócenie wszystkich workerów | Testy chaos co kwartał | 1 niespełniony test / kwartał | Patroni, orkiestracja |

Umowy SLA z jednostkami biznesowymi wyprowadzane są z powyższych SLO z marginesem bezpieczeństwa (np. SLA na blokadę konta Leavera: 15 min, przy SLO 5 min).

## 1.3 Macierz atrybutów jakościowych (ISO/IEC 25010)

| Charakterystyka ISO 25010 | Podcharakterystyka | Wymaganie dla IC IGA | Mechanizm architektoniczny | Weryfikacja |
|---|---|---|---|---|
| Niezawodność | Dostępność | 99,95 % dla API zarządczego | Active-Active dla warstwy bezstanowej (API, workery), Active-Passive z automatycznym failoverem dla PostgreSQL (Patroni + etcd), klaster Kafka 5 brokerów (RF=3, min.insync.replicas=2), Redis Sentinel | Sondy syntetyczne, raport miesięczny |
| Niezawodność | Odporność na błędy (Fault tolerance) | Awaria dowolnego pojedynczego węzła lub jednej z dwóch serwerowni nie przerywa przyjmowania zdarzeń ani pracy API | Rozkład komponentów na dwie serwerownie (DC-A, DC-B) + węzeł arbitrażowy etcd/ZooKeeper w DC-C; kworum 2 z 3 | Test odłączenia serwerowni co pół roku |
| Niezawodność | Odtwarzalność (Recoverability) | RPO 0 dla DB, RTO 15 min | Synchroniczna replikacja streaming, pgBackRest z WAL archiving na NAS, backupy pełne co dobę, przyrostowe co godzinę | Test odtworzenia co miesiąc |
| Niezawodność | Odporność na partycjonowanie sieci On-Premise | Rozdzielenie DMZ i strefy zaufanej nie może prowadzić do podwójnego wykonania operacji provisioningowych ani do utraty zadania | Pull-based agenci (brak połączeń inicjowanych z DMZ do Core), idempotency_key per operacja, Transactional Outbox, leasing zadań z TTL | Test chaos: iptables DROP między strefami przez 30 min |
| Bezpieczeństwo | Niezaprzeczalność (Non-repudiation) | Każda decyzja (akceptacja wniosku, recertyfikacja, wyjątek SoD) musi być przypisana do uwierzytelnionego aktora w sposób niemodyfikowalny | Hash-chain SHA-256, checkpointy podpisywane Ed25519 kluczem w HSM, kotwiczenie RFC 3161, eksport WORM | Codzienna weryfikacja łańcucha, audyt zewnętrzny |
| Bezpieczeństwo | Integralność | Brak możliwości wstecznej zmiany historii uprawnień i zdarzeń audytowych; zmiany stanu konta tylko przez zdefiniowane przejścia automatu | Tabele append-only z blokadą UPDATE/DELETE na poziomie ról DB (REVOKE) i triggerów, SCD2 dla historii uprawnień, automaty stanów egzekwowane w kodzie i CHECK constraints | Testy mutacyjne, skanowanie uprawnień DB |
| Bezpieczeństwo | Poufność | Poświadczenia konektorów nigdy nie występują w postaci jawnej w DB, logach ani pamięci dłużej niż czas użycia | Envelope encryption (KEK w HSM/Vault, DEK AES-256-GCM per wpis), mTLS, TLS 1.3, maskowanie w logach | Pentest roczny, skan sekretów w CI |
| Bezpieczeństwo | Rozliczalność (Accountability) | Pełna ścieżka audytowa „kto zatwierdził, na jakiej podstawie, jaką politykę oceniono” | AuditEvent z `policy_snapshot_hash`, wersjonowanie ról i reguł, immutable revisions | Pakiet dowodowy kampanii |
| Wydajność | Zachowanie w czasie | SLO-01 do SLO-08 | Kompilacja reguł SoD do bitsetów w pamięci, indeksy częściowe, partycjonowanie zakresowe, cache Redis dla hierarchii ról | Testy obciążeniowe k6/Locust przed każdym wydaniem |
| Wydajność | Wykorzystanie zasobów | Pełny mining 100k × 450k w < 4 h na 32 vCPU / 256 GB RAM | Roaring Bitmaps, numpy/scipy, partycjonowanie macierzy per aplikacja | Benchmark na danych syntetycznych |
| Wydajność | Pojemność | 100 000 tożsamości, 1,5 mln kont, 5 mln uprawnień, 5 lat retencji audytu | Sizing w rozdziale 2.4 | Przegląd pojemności kwartalny |
| Utrzymywalność | Modularność | Komponenty wymienne bez zmian w pozostałych (konektory, ewaluator polityk, broker) | Kontrakty interfejsów (Pydantic), broker abstrakcyjny (Kafka lub RabbitMQ), plugin SDK konektorów | Testy kontraktowe |
| Utrzymywalność | Testowalność | Każdy automat stanów ma pełną macierz przejść testowaną property-based | Hypothesis, testy tabelaryczne przejść | Pokrycie przejść 100 % |
| Zgodność (Compatibility) | Interoperacyjność | SCIM 2.0 (RFC 7643/7644), REST/SOAP, JDBC/ODBC, LDAP, CSV/XLSX | Warstwa konektorów z adapterami | Testy zgodności SCIM |
| Użyteczność | Ochrona przed błędami użytkownika | Pre-request SoD check, blokada self-review, wymuszony dowód realizacji dla zadań ręcznych | Walidacje w Governance Engine | Testy UX scenariuszy S1–S12 |
| Przenośność | Instalowalność | Wdrożenie na RHEL 9 / Ubuntu 22.04 LTS, Kubernetes on-prem (RKE2 / OpenShift) lub bare-metal systemd | Obrazy OCI, Helm chart, playbooki Ansible | Instalacja referencyjna w labie |

## 1.4 Modelowanie zagrożeń STRIDE dla warstwy IGA

Model zagrożeń obejmuje specyficzne dla IGA wektory, w których sam system nadzoru staje się celem: skompromitowana platforma IGA jest najkrótszą drogą do eskalacji uprawnień w całej organizacji. Każde zagrożenie ma przypisane kontrole prewencyjne, detekcyjne i komponent właścicielski.

| Kategoria STRIDE | Zagrożenie specyficzne dla IGA | Wektor | Kontrole prewencyjne | Kontrole detekcyjne | Komponent |
|---|---|---|---|---|---|
| **S**poofing | Podszycie się pod system HR i wstrzyknięcie fałszywego zdarzenia Joiner/Mover nadającego uprawnienia | Fałszywy producent Kafka, przechwycony webhook | mTLS z certyfikatami klienckimi per producent, ACL Kafka na topic `hr.events.v1` tylko dla principal `hr-gateway`, podpis HMAC payloadu kluczem z Vault | Alert na zdarzenia HR spoza okna harmonogramu lub o anomalnym wolumenie (> 3σ), korelacja z logiem HR Gateway | HR Gateway, Kafka |
| Spoofing | Podszycie się pod agenta on-premise i odebranie zadań provisioningowych innego systemu | Kradzież tokena agenta | Certyfikat kliencki per agent wystawiany przez wewnętrzne CA, binding `agent_id ↔ cert fingerprint ↔ dozwolone application_id` w DB, krótkotrwałe leasingi zadań | Alert przy odebraniu zadania z niezgodnym fingerprintem, geolokalizacja IP agenta w strefie DMZ | Agent Gateway |
| Spoofing | Podszycie się pod akceptującego w workflow | Przejęcie sesji, replay tokenu | OIDC z PKCE przeciwko korporacyjnemu IdP, step-up MFA dla akceptacji wniosków o ryzyku HIGH/CRITICAL, bindowanie sesji do `sid` i fingerprintu TLS | Audyt wszystkich decyzji z `actor_session_id`, wykrywanie równoległych sesji z różnych podsieci | Governance Engine |
| **T**ampering | Modyfikacja wsteczna rekordu audytowego (ukrycie nieautoryzowanego nadania) | Dostęp administratora DB, SQL injection | Role DB bez UPDATE/DELETE na tabelach audytowych, hash-chain, podpisane checkpointy w HSM, eksport do WORM | Codzienna pełna weryfikacja łańcucha (SLO-12), weryfikacja losowych próbek co godzinę, porównanie z kopią WORM | Audit Service |
| Tampering | Manipulacja definicją reguły SoD w celu uniknięcia wykrycia konfliktu | Nieautoryzowana zmiana w tabeli `policy_rule` | Wersjonowanie reguł (immutable revisions), zmiana reguły wymaga akceptacji dual-control (Security Officer + Compliance), `policy_snapshot_hash` w każdej ocenie | Alert na zmianę reguły poza workflow, dzienny diff definicji polityk | Policy Engine |
| Tampering | Manipulacja zadaniem w kolejce Outbox (zmiana `target_account` lub `operation`) | Dostęp do DB przez skompromitowany worker | Payload Outbox podpisywany HMAC kluczem per worker-pool, weryfikacja przed wykonaniem, hash payloadu w AuditEvent | Rozbieżność HMAC = DLQ + alert P1 | Outbox Relay, Provisioning Engine |
| Tampering | Desynchronizacja uprawnień: ręczne nadanie w systemie docelowym (out-of-band) ukrywane przed IGA | Administrator aplikacji nadaje uprawnienia bezpośrednio | Polityka „IGA jako jedyny kanał zmian” egzekwowana przez reconciliation z auto-revert dla aplikacji klasy A | Detective check w Reconciliation Loop, raport out-of-band dzienny, SIEM | Reconciliation Service |
| **R**epudiation | Akceptujący zaprzecza, że zatwierdził wniosek | Brak dowodu kryptograficznego | Każda decyzja zapisana w hash-chain z `actor_id`, `session_id`, `decision_hash`, znacznikiem czasu TSA; dla decyzji HIGH wymagane step-up MFA zapisane jako `amr` | Pakiet dowodowy generowany on-demand | Audit Service |
| Repudiation | Realizator zadania ręcznego twierdzi, że wykonał operację, której nie wykonał | Brak closed-loop | Wymóg dowodu (ticket ITSM, załącznik z hashem SHA-256), stan `CONFIRMED_MANUAL` odrębny od `VERIFIED_AUTOMATED`, okresowy import CSV i diff | Reconciliation diff ujawnia rozbieżność, eskalacja do właściciela aplikacji | Manual Fulfillment Connector |
| **I**nformation Disclosure | Wyciek poświadczeń konektorów (hasła serwisowe do AD, SAP, baz) | Dump DB, log debug | Envelope encryption, DEK w pamięci tylko na czas operacji, zeroizacja, zakaz logowania payloadów konektorów, Vault Transit | Skan logów pod kątem wzorców sekretów, alert na masowe odczyty tabeli `connector_credential` | Secrets Manager |
| Information Disclosure | Ujawnienie danych osobowych poprzez API wyszukiwania tożsamości | Nadmierne uprawnienia API | RBAC administracyjny (scope-based), ABAC per jednostka organizacyjna dla menedżerów, pola PII maskowane domyślnie, rate-limiting | Audyt odczytów PII, wykrywanie enumeracji | Identity Core API |
| Information Disclosure | Wyciek macierzy User × Entitlement z silnika miningu | Eksport wyników | Mining działa na pseudonimizowanych identyfikatorach (`identity_pseudo_id`), re-identyfikacja tylko przez Governance Engine z uprawnieniem `mining:reveal` | Audyt operacji reveal | Mining Engine |
| **D**enial of Service | Zalanie HR Gateway zdarzeniami blokujące JML | Błąd w HR, atak | Rate-limiting per producent, backpressure Kafka, oddzielne partycje dla priorytetu `LEAVER_EMERGENCY` | Alert lag konsumenta | HR Gateway |
| Denial of Service | Blokada tabeli Outbox przez długą transakcję uniemożliwiająca provisioning | Błędny worker, vacuum | `SELECT FOR UPDATE SKIP LOCKED`, `statement_timeout`, `idle_in_transaction_session_timeout=30s`, partycjonowanie Outbox | Alert `outbox_publish_lag_seconds` > 10 s | Outbox Relay |
| Denial of Service | Wyczerpanie zasobów przez mining na produkcyjnej DB | Ciężkie zapytania | Mining czyta ze snapshotu (replika read-only, eksport do Parquet), osobna pula zasobów | Monitoring repliki | Mining Engine |
| **E**levation of Privilege | Privilege escalation przez samodzielne nadanie sobie roli administratora IGA | Nadużycie API | Zakaz self-approval egzekwowany w DB (CHECK + trigger) i kodzie, role administracyjne IGA wymagają dual-control, break-glass z czasowym tokenem i obowiązkowym post-review | Alert na każdą zmianę ról administracyjnych, raport dzienny | Governance Engine |
| Elevation of Privilege | Obejście SoD przez rozłożenie wniosków w czasie (A dziś, B za tydzień) | Okno czasowe | Ocena SoD zawsze na pełnym stanie docelowym (current + pending + requested), nie tylko na koszyku | Detective check w reconciliation wykrywa nagromadzenie | Policy Engine |
| Elevation of Privilege | Obejście SoD przez konto współdzielone / serwisowe | Konto bez właściciela | Klasyfikacja kont (`PERSONAL`, `SERVICE`, `SHARED`, `ORPHAN`), wymóg właściciela dla SERVICE/SHARED, SoD liczony na poziomie tożsamości właściciela | Raport kont bez właściciela, kampania Application Owner | Reconciliation Service |
| Elevation of Privilege | Wykorzystanie Rehire do odziedziczenia dawnych uprawnień | Reaktywacja bez przeglądu | Rehire zawsze startuje z pustym zbiorem uprawnień, dawne uprawnienia tylko jako „sugestia do wniosku” wymagająca akceptacji | Audyt scalania tożsamości | Identity Core |
| Elevation of Privilege | Eskalacja przez mining promujący rolę jednoosobową z nadmiarowymi uprawnieniami do roli biznesowej | Automatyczna promocja | Typ `EXCEPTION` z flagą `auto_promote_forbidden=true`, promocja kandydata do roli produkcyjnej wymaga akceptacji właściciela + SoD check | Audyt promocji | Mining Engine, Governance Engine |

### 1.4.1 Założenia graniczne modelu zagrożeń

- Zaufana baza obliczeniowa (TCB) obejmuje: HSM, serwery Vault, węzły PostgreSQL w strefie zaufanej, brokery Kafka. Kompromitacja TCB jest poza zakresem kontroli technicznych i podlega kontrolom organizacyjnym (dual-control fizyczny, segmentacja).
- Agenci w DMZ są traktowani jako **półzaufani**: mogą wykonywać wyłącznie operacje jawnie przypisane do ich `application_id`, nie mają możliwości odczytu danych innych aplikacji ani inicjowania połączeń do Core.
- Administrator DB ma możliwość zniszczenia danych (dostępność), ale nie ma możliwości ich **niewykrywalnej** modyfikacji dzięki łańcuchowi skrótów i kopii WORM poza jego zasięgiem.

## 1.5 Ograniczenia i założenia projektowe

| ID | Ograniczenie / założenie | Konsekwencja architektoniczna |
|---|---|---|
| C-01 | Zero zależności od chmur publicznych | Brak managed services; wszystkie komponenty (Kafka, Redis, Vault, PostgreSQL, obiektowy storage S3-compatible MinIO) utrzymywane lokalnie |
| C-02 | Dwie serwerownie w odległości < 50 km (RTT < 2 ms) + trzecia lokalizacja arbitrażowa | Umożliwia synchroniczną replikację PostgreSQL bez degradacji latencji zapisu |
| C-03 | Systemy docelowe w sieciach izolowanych (OT, strefy produkcyjne) | Agent pull-based przez mTLS, brak połączeń przychodzących do tych sieci |
| C-04 | System HR (SAP SuccessFactors On-Prem / własny) publikuje zdarzenia bez gwarancji kolejności | Wersjonowanie stanu tożsamości, klucze idempotencji, bufor reorder |
| C-05 | 470 aplikacji bez API | Manual Fulfillment Connector jako pełnoprawny komponent, nie wyjątek |
| C-06 | Retencja audytu 7 lat (SOX), prawo do usunięcia (RODO) | Pseudonimizacja PII w audycie, krypto-shredding kluczy per tożsamość |
| C-07 | Stack: Python 3.11+, FastAPI, PostgreSQL 15+, Kafka 3.6+ (alternatywnie RabbitMQ 3.12+), Redis 7+, Vault 1.15+ / HSM PKCS#11 | Wszystkie kontrakty definiowane w Pydantic v2 |
