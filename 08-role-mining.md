# 8. Architektura Silnika Role Mining

## 8.1 Zasady projektowe

### 8.1.1 Zasada nadrzędna: gwarancja wyniku

Dla dowolnego niepustego wejścia (≥ 1 tożsamość z ≥ 1 przypisaniem uprawnienia) silnik zwraca ≥ 1 kandydata na rolę. Realizowane przez:

1. **Kaskadę z rozluźnianiem parametrów** (sekcja 8.5): kolejne szczeble, aż wymagana minimalna liczba kandydatów zostanie osiągnięta.
2. **Gwarancję pokrycia resztkowego** (sekcja 8.6): po konsolidacji ról każde niepokryte przypisanie (user, entitlement) trafia do roli `RESIDUAL`.
3. **Fallback M10** (sekcja 8.3.10): zawsze produkuje wynik, nawet dla 1 tożsamości.

Jedyny wyjątek: wejście bez żadnych przypisań uprawnień. Wtedy silnik zwraca jawną diagnozę (`NO_ASSIGNMENTS_IN_SCOPE`, nie pustą listę) oraz — jeśli istnieją polityki birthright dla jednostki organizacyjnej — propozycje `BIRTHRIGHT` wyłącznie z atrybutów.

### 8.1.2 Reprezentacja danych: bitsety Pythona

Macierz User × Entitlement jest reprezentowana jako słownik `{user_idx: int}`, gdzie wartość `int` jest bitmapą uprawnień (bity pozycji entitlement_idx). Python `int` ma nieograniczoną precyzję — bitmapa na 450 000 uprawnień zajmuje ~56 KB per użytkownik.

```python
# Kompaktowy bitset: uprawnienie e ∈ zbiór użytkownika u ⟺ (matrix[u] >> e) & 1 == 1
intersection = matrix[u1] & matrix[u2]           # O(M/64) bitów
jaccard = popcount(matrix[u1] & matrix[u2]) / popcount(matrix[u1] | matrix[u2])
support_of_entitlement_e = sum(1 for v in matrix.values() if (v >> e) & 1)
```

Dla dużych zbiorów (> 50 000 tożsamości) stosowane jest partycjonowanie per aplikacja — każda aplikacja ma własną macierz. Mining odbywa się per partycja, a wyniki są konsolidowane.

### 8.1.3 Deterministyczność i powtarzalność

Każdy przebieg miningu (MiningRun) rejestruje:
- `seed` (losowość Pythona `random.seed(seed)` i `numpy.random.default_rng(seed)`)
- `data_hash = SHA-256(canonical_json(sorted([(u_idx, bin(bitmask)) for u_idx, bitmask in matrix.items()])))`
- `algorithm_versions` per miner
- `parameters_json` (pełny zestaw parametrów po rozluźnieniach)

Dla tego samego `(seed, data_hash, parameters, algorithm_version)` wynik jest identyczny. `relaxation_trace` zapisuje każde rozluźnienie z numerem szczebla i powodem.

## 8.2 DatasetProfiler

Przed doborem metod `DatasetProfiler` oblicza metryki profilu:

```python
class DatasetProfile(ContractModel):
    n_users: int                          # liczba tożsamości w zakresie
    n_entitlements: int                   # liczba unikalnych uprawnień
    density: float                        # średnia liczba uprawnień / n_entitlements (0..1)
    entitlement_set_sizes: list[int]      # histogram rozmiarów zbiorów (percentyle p25/p50/p75/p90)
    mean_jaccard: float                   # średnie podobieństwo Jaccard między losową próbką par (max 10k par)
    attr_completeness: dict[str, float]   # kompletność atrybutów HR (fraction non-null)
    attr_cardinality: dict[str, int]      # kardynalność wartości atrybutu
    n_singleton_entitlements: int         # uprawnienia z support = 1
    n_island_users: int                   # użytkownicy z Jaccard = 0 do wszystkich innych
    n_applications: int
    has_manual_applications: bool
    namespace_depth_distribution: dict[int, int]  # głębokość tokenów namespace
    profiling_duration_ms: int
```

Reguły doboru metod w trybie AUTO (zaimplementowane jako `StrategySelector.select(profile) -> list[MinerId]`):

```
if profile.n_users < 20:
    methods = [M4, M6, M10]
    if profile.n_users >= 4: methods += [M7(k=2)]
    if profile.n_singleton_entitlements > 0.5 * profile.n_entitlements: methods += [M8]
    significance_tests = False; metrics = ABSOLUTE
elif 20 <= profile.n_users < 200:
    methods = [M4, M2, M3, M5, M6, M7]
    if profile.n_island_users > 0.3 * profile.n_users: methods += [M8]
else:  # >= 200
    methods = [M1, M2, M3, M5, M6, M7, M8, M9, M10]
    if profile.density > 0.3: methods = [M2, M3, M1, M5, M6, M10]  # gęsta → bottom-up priorytet

if any(profile.attr_completeness.get(a, 0) < 0.7 for a in ['department', 'job_title', 'location']):
    methods.remove_if_present(M1)
    warnings += "M1 pominięty: kompletność atrybutów HR < 70%"

if profile.n_island_users > 0 and M6 not in methods:
    methods.insert(0, M6)  # izolowani użytkownicy → namespace miner priorytetowy

if profile.n_singleton_entitlements > 0.7 * profile.n_entitlements:
    warnings += ">70% uprawnień unikalnych — metody bottom-up mogą nie znaleźć wzorców; M6, M8, M10 priorytetowe"
```

UI wyświetla uzasadnienie: tabelę z profilem + listę wybranych minerów i powody doboru (lub pominięcia).

## 8.3 Algorytmy Role Mining (M1–M10)

Każdy algorytm implementuje:

```python
class RoleMiner(Protocol):
    miner_id: ClassVar[str]           # "M1", "M2", ...
    miner_version: ClassVar[str]      # semver

    def mine(
        self,
        matrix: dict[int, int],              # {user_idx: entitlement_bitmask}
        user_attrs: dict[int, dict],         # {user_idx: {attr: value}}
        entitlement_meta: dict[int, EntitlementMeta],
        params: MinerParams,
        timeout_seconds: float,
        max_candidates: int,
        rng: np.random.Generator,
    ) -> list[RoleCandidate]:
        """Deterministyczny. Zgłasza TimeoutError po timeout_seconds."""
```

```python
class RoleCandidate(ContractModel):
    candidate_id: UUID
    miner_id: str
    entitlement_set: frozenset[int]       # indeksy uprawnień
    member_set: frozenset[int]            # indeksy użytkowników
    candidate_type: Literal["BIRTHRIGHT", "BUSINESS", "TECHNICAL", "APP_SCOPED", "EXCEPTION", "RESIDUAL"]
    confidence: Literal["HIGH", "MEDIUM", "LOW"]
    rationale: str                         # czytelne uzasadnienie
    metrics: CandidateMetrics
    found_by: list[str] = []              # wypełniane przez Ensemble po konsolidacji
    consensus_score: float = 0.0
    parent_candidate_id: UUID | None = None
    sod_violation_ids: list[UUID] = []    # SoD-aware mining
    auto_promote_forbidden: bool = False  # dla EXCEPTION
    relaxation_level: int = 0             # szczebel kaskady (0 = parametry oryginalne)
    app_scope: list[int] = []             # application_id's pokrytych uprawnień
```

### 8.3.1 M1 — Attribute-Lift (top-down)

**Typ**: ABAC/HR. **Złożoność**: O(|attrs| × |values| × n_users + |candidates| × M) gdzie M = liczba uprawnień.

**Algorytm**:
1. Dla każdej pary `(atrybut, wartość)` oblicz `group_mask = bitmask(users z attr=val)`.
2. Dla każdego uprawnienia `e`: `support_in_group = popcount(group_mask & users_with_e_mask)`.
3. `confidence = support_in_group / popcount(group_mask)`, `lift = confidence / global_support(e)`.
4. Zbierz pary z `confidence >= params.min_confidence` i `lift >= params.min_lift`.
5. Łącz atrybuty w reguły kompozytowe (AND): dla top-k atrybutów oblicz `intersection_group` i powtórz.
6. Kandydaci = grupy użytkowników per reguła + przecięcie ich uprawnień z `confidence >= min_confidence`.

**Założenia**: atrybuty HR są dobrą proksy ról biznesowych; uprawnienia są homogeniczne w grupie. **Tryby awarii**: (a) wszystkie atrybuty mają kardynalność ≈ n_users (każdy user unikalny) → 0 kandydatów; (b) >50% braków atrybutów → bias; (c) uprawnienia przyznane indywidualnie (nie grupowo) → niski lift.

**Wymagany rozmiar próby**: ≥ 20 użytkowników per wartość atrybutu dla statystycznej stabilności; dla n < 20 w grupie metryki podawane jako absolutne (bez p-values).

**Uzasadnienie włączenia**: jedyna metoda odkrywająca reguły ABAC (atrybut → zestaw uprawnień), które bezpośrednio mapują na birthright policies w organizacji z silną strukturą HR. Niezbędny dla SOX w środowiskach z wyraźną strukturą działową.

---

### 8.3.2 M2 — Frequent Closed Itemsets (Eclat na bitsetach)

**Typ**: bottom-up, associative. **Złożoność**: O(2^M) worst-case (ekspotential); w praktyce O(|frequent_sets| × n_users / 64) dzięki bitsetom.

**Algorytm** (Eclat — pionowe podejście):
1. Każde uprawnienie `e` ma bitset `tidset[e]` = bitmask użytkowników posiadających `e`.
2. Startuj od singletonów z `support(e) = popcount(tidset[e]) >= min_support_abs`.
3. Łącz pary: `tidset(A ∪ B) = tidset(A) & tidset(B)`; jeśli support ≥ min → rekurencja.
4. **Closed itemset**: zbiór `S` jest zamknięty gdy nie istnieje `S' ⊃ S` z tym samym `tidset`. Sprawdzanie: dla wszystkich superzbiorów w poprzednim poziomie.
5. Wynikowe zamknięte zbiory → kandydaci (zbiór uprawnień = S, zbiór użytkowników = tidset(S)).
6. Limit: max `max_candidates` zbiorów posortowanych malejąco po support.

**Założenia**: uprawnienia przydzielane razem (korelacja). **Tryby awarii**: (a) brak wzorców (każdy user unikalny → wszystkie singletons); (b) eksplozja kombinatoryczna przy gęstej macierzy → timeout (obsługa: próg support adaptywny, limit głębokości = 4).

**Wymagany rozmiar**: ≥ 10 użytkowników dla stabilnych wzorców. **Uzasadnienie**: jedyna metoda gwarantująca znalezienie **wszystkich** nietrywialnych wzorców do zadanego progu support bez redundancji (closed sets). Kluczowa dla odkrywania ról nieopartych na atrybutach HR.

---

### 8.3.3 M3 — Jaccard Hierarchical Clustering (aglomeracyjny)

**Typ**: bottom-up, similarity. **Złożoność**: O(n² × M/64) dla obliczenia macierzy podobieństwa; O(n² log n) dla hierarchii.

**Algorytm**:
1. Oblicz macierz Jaccard `J[i,j] = popcount(m[i] & m[j]) / popcount(m[i] | m[j])` dla wszystkich par (dla n > 500: próbkowanie k-medoids z `k = sqrt(n)` jako punkty centralne, przybliżona macierz).
2. Aglomeracja Average-Linkage lub Complete-Linkage (konfigurowalne).
3. Wytnij dendrogram przy progu `jaccard_threshold` (domyślnie 0.6).
4. Dla każdego klastra: rdzeń = `bitwise AND` wszystkich masek → kandydat; outliery = uprawnienia powyżej rdzenia.
5. Klastr z 1 użytkownikiem → kandydat EXCEPTION.

**Założenia**: użytkownicy pełniący podobne funkcje mają podobne zbiory uprawnień; Jaccard mierzy tę zbieżność. **Tryby awarii**: (a) wszyscy w jednym megaklastrze (za niski próg) → 1 kandydat; (b) n² problem dla dużych zbiorów → próbkowanie. **Wymagany rozmiar**: ≥ 20 par użytkowników. **Uzasadnienie**: naturalny do odkrywania ról z częściowymi nakładaniami (uprawnienia wspólne = rdzeń roli, dodatkowe = rozszerzenia).

---

### 8.3.4 M4 — Formal Concept Analysis (siatka pojęć)

**Typ**: dokładny, algebraiczny. **Złożoność**: O(2^min(n, M)) — dokładny, dla małych zbiorów wyczerpujący.

**Algorytm** (NextConcept / Close-by-One variant):
1. Kontekst formalny: `(U, E, I)` gdzie `(u, e) ∈ I ⟺ (matrix[u] >> e) & 1`.
2. Operator derywacji: `A' = {e : ∀u ∈ A, (u,e) ∈ I}` (uprawnienia wszystkich użytkowników w A); `B' = {u : ∀e ∈ B, (u,e) ∈ I}` (użytkownicy posiadający wszystkie uprawnienia z B).
3. Pojęcie formalne: para `(A, B)` gdzie `A'' = A` i `B'' = B` (domknięcia wzajemne).
4. Algorytm Close-by-One iteruje przez leksykograficznie minimalne reprezentacje pojęć bez duplikatów.
5. Każde pojęcie → kandydat (A = member_set, B = entitlement_set).
6. Kandydaci trywialnie duzi (1 pojęcie = cała macierz) lub z pustym przecięciem są pomijani.
7. Limit: `max_concepts` (domyślnie 500); po przekroczeniu limitu kontynuacja od największych pojęć.

**Założenia**: wszystkie maksymalne prostokąty binarnej macierzy są sensownymi rolami. **Tryby awarii**: (a) eksplozja liczby pojęć (O(2^min(n,M))) dla gęstych macierzy → przycięcie limitem; (b) dla n > 50 czas może przekroczyć timeout → M4 wyłączany automatycznie dla n > 100 (w profilerze). **Wymagany rozmiar**: optymalne dla 2–50 użytkowników; do 100 z limitem pojęć. **Uzasadnienie**: matematycznie wyczerpujący dla małych danych — gwarantuje znalezienie każdego możliwego kandydata; deterministyczny bez parametrów statystycznych, co ułatwia audyt.

---

### 8.3.5 M5 — Greedy Set Cover / δ-approximate RMP

**Typ**: optymalizacyjny. **Złożoność**: O(iterations × n_users × n_entitlements / 64).

**Algorytm** (δ-approximate Role Mining Problem):
1. Cel: znaleźć minimalny zbiór ról, który pokrywa ≥ (1-δ) ułamek wszystkich przypisań (user, entitlement).
2. Greedy: w każdej iteracji wybierz uprawnienie/zbiór uprawnień maksymalizujący pokrycie.
3. Dla δ > 0 (tolerancja błędów): po `max_roles` rolach zatrzymaj i oznacz resztę jako RESIDUAL.
4. Wariacja podstawowa: każde przypisanie `(u, e)` musi być pokryte dokładnie przez ≥ 1 rolę; minimalizuj liczbę ról.

**Założenia**: istniejące przypisania są dobrą reprezentacją rzeczywistych ról; minimalizacja liczby ról jest priorytetem. **Tryby awarii**: (a) lokalny optimum (greedy nie gwarantuje globalnego minimum); (b) za duże δ → za mało ról, za małe → zbyt wiele. **Wymagany rozmiar**: dowolny (algorytm działa dla każdej liczby użytkowników). **Uzasadnienie**: jedyna metoda z jawnym kryterium optymalizacji liczby ról — niezbędna gdy celem jest konsolidacja roli i redukcja direct assignments.

---

### 8.3.6 M6 — Entitlement Namespace Miner

**Typ**: strukturalny. **Złożoność**: O(n_entitlements × token_depth).

**Algorytm**:
1. Tokenizuj nazwy uprawnień po separatorze `:` (domyślnie), `/` lub `.`.
2. Buduj drzewo prefiksów (`namespace_tree`).
3. Każdy węzeł drzewa (prefix) → kandydat z: `entitlement_set = {wszystkie entitlements w poddrzewie}`, `member_set = {użytkownicy posiadający ≥ 1 uprawnienie z poddrzewa}`.
4. Przycinaj węzły z `|entitlement_set| < min_ns_size` (domyślnie 2) i `|member_set| < 1`.
5. Hierarchia: każdy węzeł jest `parent_candidate_id` swojego węzła dziecka.
6. Dla uprawnień bez wspólnego namespace: każde uprawnienie tworzy singletonowy kandydat-liść.

**Przykład**:
```
ERP:AP:APPROVE_INVOICE:LIMIT<10K → węzeł ERP, ERP:AP, ERP:AP:APPROVE_INVOICE
ERP:AP:CREATE_VENDOR              → węzeł ERP, ERP:AP
ERP:GL:POST_JOURNAL               → węzeł ERP, ERP:GL
```
Kandydat `ERP:AP` zawiera `{APPROVE_INVOICE:LIMIT<10K, CREATE_VENDOR}`.

**Założenia**: nazwy uprawnień mają znaczącą strukturę hierarchiczną. **Tryby awarii**: (a) brak struktury namespace (uprawnienia bezstrukturalne) → singletons; (b) zbyt głęboka hierarchia → wiele nakładających się kandydatów. **Wymagany rozmiar**: 1 tożsamość (nie potrzebuje wielu użytkowników — działa tylko na strukturze nazw). **Uzasadnienie**: jedyna metoda efektywna dla bardzo małych grup (< 5 osób) i dla uprawnień bez nakładania się zbiorów między użytkownikami; gwarantuje kandydatów nawet dla 1 tożsamości z 1 uprawnieniem.

---

### 8.3.7 M7 — Peer-Group / kNN Miner

**Typ**: podobieństwo, hybrydowy. **Złożoność**: O(n² × (attr_dim + M/64)) dla obliczeń podobieństwa.

**Algorytm**:
1. Oblicz odległość między użytkownikami jako kombinację: `dist(u,v) = α × (1 - Jaccard_ent(u,v)) + (1-α) × (1 - Jaccard_attr(u,v))` gdzie `Jaccard_attr` to udziały wspólnych wartości atrybutów.
2. Dla każdego użytkownika u: znajdź k najbliższych sąsiadów (k z params, domyślnie k=5).
3. Rola dla grupy rówieśniczej: `core_entitlements = intersection(matrix[u] for u in {u} ∪ neighbors(u))`.
4. Agregacja: kandydaci o tym samym `core_entitlements` są scalani.
5. Outliery użytkownika u = `matrix[u] XOR core_entitlements` (uprawnienia nadmiarowe i brakujące).

**Założenia**: użytkownicy o podobnych atrybutach i uprawnieniach pełnią podobne role; szum = uprawnienia indywidualne. **Tryby awarii**: (a) k za duże → za małe przecięcie → puste role; (b) atrybuty nieistotne (wysoka kardynalność) → degeneracja do Jaccard-only. **Wymagany rozmiar**: ≥ k+1 użytkowników. **Uzasadnienie**: efektywny dla danych mieszanych (atrybuty HR + uprawnienia) z szumem — uwzględnia kontekst organizacyjny, który M2/M3 ignorują.

---

### 8.3.8 M8 — Semantic Token Miner

**Typ**: tekstowy, NLP-lite. **Złożoność**: O(n_entitlements × avg_tokens + n_entitlements² × token_vocab).

**Algorytm** (TF-IDF na tokenach nazw uprawnień):
1. Tokenizuj nazwy uprawnień (`APPROVE_INVOICE` → `[approve, invoice]`, CamelCase split, snake_case split, namespace strip).
2. Oblicz TF-IDF per uprawnienie (dokument = tokeny nazwy + opcjonalne tokeny opisu).
3. Oblicz podobieństwo kosinusowe między wszystkimi parami uprawnień na macierzy TF-IDF.
4. Aglomeracyjne klastrowanie uprawnień o podobieństwie ≥ `semantic_threshold` (domyślnie 0.5).
5. Każdy klaster uprawnień → kandydat z: `member_set = {użytkownicy z ≥ 1 uprawnieniem z klastra}`.
6. Kandydaci semantyczni są oznaczani `confidence=MEDIUM` (bo opierają się na nazwie, nie na danych przydziałów).

**Założenia**: semantyka nazwy uprawnienia odzwierciedla jej funkcję biznesową; podobne nazwy = podobne uprawnienia. **Tryby awarii**: (a) nazwy kryptyczne/techniczne (np. `PRIV_0142`) → niskie podobieństwo, brak klasteryzacji → M8 nie produkuje kandydatów; (b) fałszywe podobieństwo semantyczne (np. `CREATE_VENDOR` i `CREATE_DOCUMENT` → token `create` wspólny). **Wymagany rozmiar**: działa bez żadnych użytkowników jeśli są same uprawnienia (pure semantics). **Uzasadnienie**: jedyna metoda odkrywająca relacje między uprawnieniami, które nigdy nie były przydzielone wspólnie — np. nowe uprawnienia, uprawnienia dla aplikacji manualnych z rzadkimi użytkownikami.

---

### 8.3.9 M9 — Boolean Matrix Factorization (Asso-like)

**Typ**: dekompozycja. **Złożoność**: O(iterations × n_users × n_roles × M/64).

**Algorytm** (heurystyka Asso):
1. Inicjalizuj `k` prototypowych ról (k z params) losowymi wierszami macierzy jako bitmask.
2. Faza asocjacji: dla każdej roli `r` znajdź kolumny (uprawnienia) z `P(e | r) ≥ τ_assoc` (domyślnie 0.8) — rola ma uprawnienie jeśli ≥ 80% jej członków je posiada.
3. Faza przypisania: dla każdego użytkownika `u` przypisz do roli `r` jeśli `Hamming(matrix[u], role[r]) ≤ δ` (tolerancja szumu δ z params).
4. Iteruj fazy 2-3 do konwergencji (max 50 iteracji).
5. Wynikowe role → kandydaci; wiersze nieskategoryzowane → RESIDUAL.
6. Wynik zależy od inicjalizacji → stałe ziarno; max 5 restartów z różnymi inicjalizacjami, najlepsza coverage.

**Założenia**: macierz ma ukryte wzorce ról z szumem (błędy przydziału, uprawnienia transgresyjne). **Tryby awarii**: (a) złe k → zbyt mało lub zbyt wiele ról; (b) zbieżność do lokalnego optimum; (c) wolna przy dużych macierzach. **Wymagany rozmiar**: ≥ 2×k użytkowników dla stabilności. **Uzasadnienie**: efektywny dla zaszumionych danych produkcyjnych, gdzie M2/M4 wymagają zbyt rygorystycznego dopasowania; tolerancja δ odpowiada za uprawnienia nadmiarowe (outliers).

---

### 8.3.10 M10 — Singleton / Per-Application Fallback

**Typ**: gwarancja. **Złożoność**: O(n_users × n_applications) — liniowy.

**Algorytm**:
1. Dla każdej aplikacji `app`: zbierz wszystkich użytkowników posiadających ≥ 1 uprawnienie z tej aplikacji.
2. Kandydat `APP_SCOPED[app]`: `entitlement_set = {wszystkie uprawnienia użytkowników w app}`, `member_set = {ci użytkownicy}`.
3. Dla każdego użytkownika z unikalnym zestawem uprawnień (Jaccard = 0 do wszystkich innych po agrupowaniu): osobny kandydat `EXCEPTION`.
4. Grupowanie per (aplikacja, dział): jeśli ≥ 2 użytkowników z tego samego działu ma te same uprawnienia w tej aplikacji → kandydat `APP_SCOPED` per (app, department).
5. Kandydaci `RESIDUAL` dla każdego niepokrytego przypisania.

**Założenia**: brak — algorytm deterministycznie produkuje wynik dla dowolnych danych. **Tryby awarii**: jedynym trybem awarii jest pusta macierz (brak przypisań) — obsługiwane jako NO_ASSIGNMENTS diagnoza, nie awaria M10. **Wymagany rozmiar**: 1 tożsamość z 1 uprawnieniem. **Uzasadnienie**: ostatnia deska ratunku; jego kandydaci mają `confidence=LOW` i są opatrzone rationale „wynik fallbacku M10 — wymaga przeglądu"; nie są automatycznie promowane.

## 8.4 Pipeline miningu — diagram sekwencji

```mermaid
sequenceDiagram
    autonumber
    actor UX as Użytkownik (UI)
    participant API as Governance API
    participant PR as DatasetProfiler
    participant SS as StrategySelector
    participant ME as MiningOrchestrator
    participant M1 as M1 Attribute-Lift
    participant M2 as M2 Closed Itemsets
    participant MX as M3..M9 (równolegle)
    participant M10 as M10 Fallback
    participant RC as RelaxationCascade
    participant EN as EnsembleConsolidator
    participant COV as CoverageGuard
    participant SOD as SoD Evaluator
    participant AU as Audit Service
    participant DB as PostgreSQL

    UX->>API: POST /mining/runs {scope, params, seed, auto_mode=true}
    API->>DB: INSERT mining_run(state=PROFILING, seed, data_hash)
    API->>DB: audit_event(mining_run.started)

    API->>PR: profile(matrix, user_attrs)
    PR-->>API: DatasetProfile (n_users, density, attr_completeness, ...)
    API->>SS: select_strategy(profile, user_overrides)
    SS-->>API: selected_methods: [M2, M3, M1, M6], rationale_per_method

    API->>DB: UPDATE run(state=MINING, selected_methods)
    API->>ME: orchestrate(matrix, user_attrs, methods, params, seed, timeout_per_miner)

    par Równoległe uruchomienie wybranych minerów
        ME->>M1: mine(matrix, params) within timeout
        ME->>M2: mine(matrix, params) within timeout
        ME->>MX: mine(matrix, params) within timeout
    end

    ME-->>ME: collect results; log per-miner: candidates_count, duration, errors

    ME->>RC: cascade_if_needed(results, min_required=params.min_candidates)
    Note over RC: Sprawdza czy wystarczająca liczba kandydatów
    alt Wystarczająca liczba kandydatów
        RC-->>ME: results unchanged, relaxation_level=0
    else Niewystarczająca
        RC->>RC: Szczebel 1: rozluźnij min_members 5→3, min_support -20%
        RC->>M2: rerun(relaxed_params)
        RC->>RC: Szczebel 2: rozluźnij do min_members=2; Jaccard_threshold × 0.8
        RC->>MX: rerun(more_relaxed_params)
        RC->>RC: Szczebel 3: aggregacja namespace (M6 z głębokością 1)
        RC->>M6: rerun(namespace_depth=1)
        RC->>RC: Szczebel 4: M10 fallback zawsze
        RC->>M10: mine(matrix, params)
        RC-->>ME: merged results with relaxation_trace
    end

    ME->>EN: consolidate(all_candidates, jaccard_threshold=0.85)
    Note over EN: Scal kandydatów z Jaccard≥0.85; oblicz consensus_score; deduplikacja; hierarchia parent/child
    EN-->>ME: consolidated_candidates, miner_comparison_matrix

    ME->>COV: check_coverage(consolidated_candidates, matrix)
    alt Brakujące przypisania
        COV->>COV: Utwórz RESIDUAL candidates per aplikacja / atrybut
        COV-->>ME: complete_candidates (100% coverage gwarantowane)
    end

    loop per candidate
        ME->>SOD: evaluate_batch(SIMULATION, candidate.entitlement_set)
        alt SoD naruszenie
            ME->>ME: Oznacz candidate.sod_violation_ids; propose auto-split
        end
    end

    ME->>ME: Oblicz metryki: coverage, noise, WSC, dept_entropy, outliers per user

    ME->>DB: INSERT role_candidate[], candidate_member[], mining_run(state=COMPLETED)
    ME->>DB: audit_event(mining_run.completed, candidates_count, coverage_pct, data_hash)
    API-->>UX: 200 {run_id, candidates_count, coverage_pct, relaxation_trace, miner_comparison}
```

## 8.5 Kaskada z rozluźnianiem

```python
class RelaxationCascade:
    """Uruchamia kolejne szczeble rozluźniania aż do uzyskania min_candidates."""

    LEVELS = [
        # (opis, modyfikator parametrów)
        (0, "Parametry użytkownika", lambda p: p),
        (1, "Rozluźnienie łagodne: min_members 5→3, min_support -20%",
            lambda p: p.replace(min_members=min(p.min_members, 3), min_support=p.min_support * 0.8)),
        (2, "Rozluźnienie umiarkowane: min_members 2, Jaccard×0.8",
            lambda p: p.replace(min_members=min(p.min_members, 2), jaccard_threshold=p.jaccard_threshold * 0.8)),
        (3, "Rozluźnienie silne: min_members 1, agregacja namespace głębokość 2",
            lambda p: p.replace(min_members=1, namespace_depth=2, use_namespace_aggregation=True)),
        (4, "Fallback M10: per-application, gwarantowany wynik",
            lambda p: p.replace(force_m10=True, min_members=1)),
    ]

    def run(self, miners, matrix, user_attrs, base_params, min_candidates, rng, timeout):
        for level, description, param_modifier in self.LEVELS:
            params = param_modifier(base_params)
            candidates = run_miners_with_timeout(miners, matrix, user_attrs, params, rng, timeout)
            if level == 4:  # M10 gwarantuje wynik
                candidates += M10Miner().mine(matrix, user_attrs, {}, params, timeout, 10000, rng)
            if len(candidates) >= min_candidates:
                log(f"Kaskada: szczebel {level} ({description}) → {len(candidates)} kandydatów")
                return candidates, RelaxationTrace(level=level, description=description, params=params)
        # Nigdy nie powinno być osiągnięte jeśli M10 zadziałał
        raise AssertionError("M10 nie wyprodukował żadnego kandydata — niemożliwe dla niepustej macierzy")
```

`relaxation_trace` jest wyświetlany w UI jako timeline: „Szczebel 0: 0 kandydatów → Szczebel 1: 2 kandydatów (wystarczy)".

## 8.6 Ensemble i konsolidacja

### 8.6.1 Scalanie kandydatów

```python
def consolidate(candidates: list[RoleCandidate], jaccard_threshold: float = 0.85) -> list[RoleCandidate]:
    """Scala kandydatów z różnych minerów, zachowując provenance."""
    groups: list[list[RoleCandidate]] = []
    for c in sorted(candidates, key=lambda x: -len(x.entitlement_set)):
        merged = False
        for group in groups:
            rep = group[0]  # reprezentant
            J = len(c.entitlement_set & rep.entitlement_set) / len(c.entitlement_set | rep.entitlement_set)
            if J >= jaccard_threshold:
                group.append(c)
                merged = True
                break
        if not merged:
            groups.append([c])
    
    result = []
    for group in groups:
        merged = merge_group(group)
        merged.found_by = sorted({c.miner_id for c in group})
        merged.consensus_score = sum(MINER_WEIGHT[c.miner_id] for c in group) / sum(MINER_WEIGHT.values())
        result.append(merged)
    return result
```

`MINER_WEIGHT` jest funkcją profilu danych: np. dla n < 20, waga M4 jest najwyższa; dla n ≥ 200, waga M1 i M2.

### 8.6.2 Hierarchia kandydatów

Po konsolidacji: jeśli `entitlement_set(A) ⊂ entitlement_set(B)` i `member_set(A) ⊇ member_set(B)`, to A jest `parent_candidate_id` B (rola podstawowa → rola rozszerzona). Zapobiega to duplikacji i umożliwia modelowanie RBAC hierarchicznego.

### 8.6.3 Widok porównawczy minerów

UI wyświetla macierz:

| Miner | Kandydatów | Pokrycie | Czas [s] | Nakładanie z M2 | Nakładanie z M3 |
|---|---|---|---|---|---|
| M1 | 15 | 82% | 0.4 | 0.71 | 0.58 |
| M2 | 22 | 94% | 1.2 | — | 0.63 |
| M3 | 18 | 88% | 2.1 | 0.63 | — |
| M10 | 8 | 100% | 0.1 | 0.25 | 0.31 |

Nakładanie = średni Jaccard `entitlement_set` między kandydatami obu minerów (pary o Jaccard ≥ 0.5).

## 8.7 Gwarancja pokrycia resztkowego

```python
def ensure_coverage(candidates: list[RoleCandidate], matrix: dict[int, int]) -> list[RoleCandidate]:
    """Gwarantuje 100% pokrycia przypisań (user, entitlement)."""
    covered: dict[int, int] = defaultdict(int)  # {user_idx: covered_bitmask}
    for c in candidates:
        for u in c.member_set:
            covered[u] |= reduce(or_, [1 << e for e in c.entitlement_set], 0)
    
    residuals_by_app: dict[int, tuple[frozenset, frozenset]] = defaultdict(lambda: (frozenset(), frozenset()))
    for u, bitmask in matrix.items():
        uncovered = bitmask & ~covered.get(u, 0)
        if uncovered:
            for e in iter_bits(uncovered):
                app_id = entitlement_meta[e].application_id
                ents, users = residuals_by_app[app_id]
                residuals_by_app[app_id] = (ents | {e}, users | {u})
    
    for app_id, (ents, users) in residuals_by_app.items():
        candidates.append(RoleCandidate(
            entitlement_set=ents, member_set=users,
            candidate_type="RESIDUAL", confidence="LOW",
            miner_id="COVERAGE_GUARD", relaxation_level=5,
            rationale=f"Rola resztkowa: {len(ents)} uprawnień z aplikacji {app_id} niepokrytych przez inne kandydaty",
        ))
    return candidates
```

## 8.8 SoD-Aware Mining i Auto-Split

Dla każdego kandydata silnik wywołuje `PolicyEvaluator.evaluate(SIMULATION, entitlement_set)`. Jeśli wynik = BLOCK lub REQUIRE_EXCEPTION:

1. Kandydat oznaczany `sod_violation_ids = [violation_id]`.
2. Propozycja auto-split: silnik szuka podziału `entitlement_set` na `A` i `B` takich, że `A ∩ SOD_rule.A = ∅` lub `B ∩ SOD_rule.B = ∅` i pokrycie jest maksymalne. Minimalizacja: liczba użytkowników tracących dostęp. Algorytm: zachłanny — przenoś uprawnienia z mniejszej strony konfliktu do podkandydata.
3. UI oferuje: (a) pozostaw z wyjątkiem SoD, (b) split na 2 kandydatów, (c) odrzuć.
4. Promocja kandydata z naruszeniem SoD bez wyjątku jest blokowana na poziomie API.

## 8.9 Metryki kandydata

```python
class CandidateMetrics(ContractModel):
    coverage_pct: float           # % przypisań (user, ent) pokrytych przez kandydata
    noise_pct: float              # % uprawnień w role, których > 20% członków nie posiada
    member_count: int
    entitlement_count: int
    dept_entropy: float           # entropia działu wśród członków (niska = spójne)
    wsc_contribution: float       # wkład do WSC całego zestawu ról
    direct_assignments_saved: int # liczba direct assignments zastąpionych przez rolę
    outliers_per_user: dict[int, tuple[frozenset, frozenset]]  # {user: (nadmiarowe, brakujące)}
    impact_users_gain: int        # użytkownicy zyskają uprawnienia po adopcji roli
    impact_users_lose: int        # użytkownicy stracą uprawnienia
    sod_violations: int           # naruszenia SoD wewnątrz kandydata (0 = czysta)
```

WSC (Weighted Structural Complexity) zestawu ról = `Σ_r (|entitlements_r| × |members_r|)` — niższe WSC = prostsza, bardziej zarządzalna struktura ról.

## 8.10 Lifecycle kandydata i audyt

| Stan | Opis | Przejście |
|---|---|---|
| `PROPOSED` | Wygenerowany przez miner | Opcjonalne akcje UI |
| `UNDER_REVIEW` | Wysłany do właściciela / Security Officer | Ręczna decyzja |
| `APPROVED` | Zatwierdzony do promocji | Promocja |
| `PROMOTED` | Rola biznesowa utworzona z kandydata | Finał |
| `REJECTED` | Odrzucony z uzasadnieniem | Finał; zapamiętywane w `RejectedCandidate` |
| `SUPERSEDED` | Scalony z innym kandydatem | Finał |

`RejectedCandidate` zawiera `entitlement_fingerprint = SHA-256(sorted(entitlement_ids))` — przy kolejnym przebiegu, jeśli miner wygeneruje kandydata z tym samym fingerprintem i uzasadnieniem odrzucenia, zostanie automatycznie oznaczony `PREVIOUSLY_REJECTED` (nie zablokowany, ale ostrzeżony).

Promocja kandydata do roli:
1. Governance API: `POST /mining/candidates/{id}/promote`.
2. Pre-check SoD dla `SIMULATION`.
3. Jeśli kandydat to `EXCEPTION` i `auto_promote_forbidden=true` → blokada + wymagany ręczny override z uzasadnieniem.
4. Transakcja: `INSERT role`, `INSERT role_revision`, `INSERT role_entitlement_assignment`, `INSERT entitlement_assignment` per member (lub provisioning_task dla aplikacji automatycznych), `INSERT outbox_event(role.promoted)`.
5. Dla aplikacji manualnych: `provisioning_task(ASSIGN_ROLE_MANUAL)` → Manual Fulfillment Connector.
6. Zdarzenie: `audit_event(role.promoted_from_mining, candidate_id, run_id, seed, data_hash)`.
