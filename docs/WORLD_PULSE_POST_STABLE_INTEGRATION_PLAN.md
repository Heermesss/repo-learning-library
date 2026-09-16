# World Pulse — Post-Stable Integration Plan

Bu belge, mevcut **Hermes + n8n + PostgreSQL + Agent Office** control-plane kurulumu tamamen stabil olduktan sonra World Pulse sistemine hangi repo/fikirlerin hangi sırayla ve hangi güvenlik sınırlarıyla eklenmesi gerektiğini tanımlar.

> Temel ilke: Bir bileşen yalnızca somut bir problemi çözüyor ve mevcut trust boundary'yi bozmuyorsa production yoluna alınır. Her entegrasyon ayrı canary, ölçüm ve rollback ile yapılır.

---

## 0. Mevcut mimariyi dondur ve baseline al

Yeni repo eklemeden önce mevcut sistem için bir **known-good baseline** oluştur:

- Hermes mevcut model/provider ayarı değişmeden çalışıyor.
- n8n yalnızca orchestration katmanı; host üzerinde genel amaçlı root shell değil.
- PostgreSQL public port açmıyor.
- Agent Office yalnızca observability; Hermes'i kontrol etmiyor.
- Agent Office loopback/internal bind kullanıyor ve sanitized event feed dışında Hermes secrets görmüyor.
- n8n, Agent Office ve PostgreSQL restart sonrası geri geliyor.
- Backup/rollback dokümante edilmiş ve restore testi yapılmış.
- Unexpected listener yok.
- Secret canary testleri geçiyor.
- Hermes regression testi geçiyor.
- CPU/RAM/disk için normal çalışma baseline'ı kaydedilmiş.

### Stabil kabul kapısı

Aşağıdaki maddeler geçmeden yeni entegrasyon ekleme:

1. Tüm servisler clean restart sonrası sağlıklı.
2. Bir tam ingestion/analysis test akışı hata vermeden tamamlanıyor.
3. Secret leak testi PASS.
4. Hostta beklenmeyen public listener yok.
5. Hermes'in mevcut işleri Agent Office kapalıyken de devam ediyor.
6. n8n restartında workflow state bozulmuyor.
7. PostgreSQL backup + restore doğrulandı.
8. Her servisin rollback yolu yazılı.
9. Baseline token/latency/error ölçümü mevcut.

Bu noktada baseline commit/config snapshot alınır. Bundan sonraki her entegrasyon tek başına eklenir; iki yeni sistemi aynı anda production'a sokma.

---

# 1. Öncelik 1 — Agent Reach: X/Twitter ingestion

Repository: `Panniantong/Agent-Reach`

## Neden ilk sırada?

World Pulse için X erken sinyal kaynağıdır. Agent Reach; X, RSS, web, Reddit, YouTube gibi kaynaklara erişim sağlayabilir. Bizim ilk kullanımımız **yalnızca X read/search ingestion** olacaktır.

## Yanlış mimari

```text
Hermes -> Agent Reach -> X
```

Bu yapı Hermes'i doğrudan internet collector haline getirir, tool kullanımını genişletir ve her aramada reasoning tokenı yakar.

## Hedef mimari

```text
X/Twitter
   |
   v
Agent Reach / X adapter
   |
   v
n8n collector workflow
   |
   v
normalize -> prefilter -> dedupe/retrieval
   |
   v
Hermes analysis
```

## Deployment sınırı

Agent Reach'i Hermes process'inin içine kurma. Ayrı bir düşük yetkili servis/sidecar olarak çalıştır.

Önerilen sınır:

```text
user: worldpulse-x
sudo: none
Docker socket: none
Hermes home: denied
SSH home: denied
n8n secret store: denied
World Pulse DB credential: yalnızca gerekiyorsa read/write için dar yetki
```

### X credential storage

X oturum/auth bilgileri hassastır. Örnek protected location:

```text
/etc/worldpulse-secrets/agent-reach/
  twitter.env
```

- directory `0700`
- file `0600`
- owner yalnızca X collector service user
- n8n workflow JSON'una cookie/token gömülmez
- Hermes prompt/context içine credential girmez
- Agent Office event feed'e credential alanları çıkmaz

## n8n ile bağlantı

n8n'in host shell'e genel erişimini açmak yerine dar bir adapter kullan:

```text
POST /v1/x/search
POST /v1/x/user-timeline
GET  /health
```

Adapter sadece gerekli parametreleri kabul eder:

```json
{
  "query": "...",
  "since_id": "...",
  "limit": 50
}
```

Shell command, arbitrary URL, filesystem path veya raw command kabul etmemeli.

## X ingestion data contract

World Pulse'a X'ten yalnızca normalize edilmiş alanlar girsin:

```json
{
  "platform": "x",
  "post_id": "...",
  "author_id": "...",
  "author_handle": "...",
  "created_at": "...",
  "text": "...",
  "canonical_url": "...",
  "reply_to_id": null,
  "quote_of_id": null,
  "repost_of_id": null,
  "metrics": {},
  "collector_received_at": "..."
}
```

Credential/cookie/header/raw browser state hiçbir zaman payload'a girmez.

## Operational rules

- İlk sürüm READ ONLY: post atma, DM, like, follow, delete yok.
- Her watch query için cursor/since_id tut.
- Retry exponential backoff kullan.
- Rate-limit cevabını başarısız haber diye değil collector state olarak işle.
- Aynı post_id idempotent olmalı.
- Retweet/repost ayrı haber gibi Hermes'e gitmemeli.
- X içeriği **signal** kabul edilir; doğrulanmış fact değildir.
- Kritik claim için daha sonra ikinci kaynak/evidence validation gerekir.

## Canary testi

Önce tek bir düşük hacimli watch query ile çalıştır. Ölç:

- çağrı başarı oranı
- rate-limit sıklığı
- post/minute
- duplicate oranı
- n8n execution süresi
- Hermes'e kadar ulaşan post yüzdesi
- post başına toplam token maliyeti

Başarılıysa watchlist sayısı kademeli artırılır.

---

# 2. Agent Reach'ten hemen sonra — Retrieval / Deduplication katmanı

Bu bir üçüncü-party repo değil; World Pulse'ın kendi çekirdek bileşeni olmalı.

Amaç: Hermes'in geçmiş arşivin tamamını okumamasını sağlamak.

```text
incoming item
   |
   +-> exact URL/post ID hash
   +-> normalized title/text fingerprint
   +-> metadata filter (actor/country/type/time)
   +-> vector similarity
   |
   v
Top-K candidate events
   |
   v
Hermes: same event / continuation / new event
```

## Database ayrımı

n8n'in kendi PostgreSQL tablolarına World Pulse domain verisini karıştırma.

Aynı PostgreSQL instance kullanılabilir ama ayrı database/user önerilir:

```text
postgres
  |- n8n database
  `- worldpulse database
       |- raw_items
       |- articles
       |- social_posts
       |- events
       |- event_sources
       |- entities
       |- claims
       |- evidence
       `- ingestion_runs
```

World Pulse için gerektiğinde `pgvector` kullan.

## Raw data kuralı

Raw article/post asla destructive compression ile değiştirilmez. Normalize edilmiş veya özetlenmiş kopya ayrı alanda tutulur.

---

# 3. Öncelik 2 — Behavior Skills fikri: World Pulse-native skills

Repository referansı: `Arete-Consortium/ai-skills`

Bu repo doğrudan production'a kurulmayacak. Claude Code odaklı installer yerine davranış tasarım modelini alacağız.

## Oluşturulacak World Pulse skill seti

```text
skills/
  scout/
    SKILL.md
    schema.json
  analyst/
    SKILL.md
    schema.json
  validator/
    SKILL.md
    schema.json
  worker/
    SKILL.md
    schema.json
  synthesizer/
    SKILL.md
    schema.json
```

### Scout

Görev:
- yeni signal bul
- source metadata koru
- yorum/sonuç üretme
- duplicate olabilecek öğeleri işaretle

Çıktı kesin JSON schema.

### Analyst

Görev:
- candidate events arasından eşleşme kararı
- `same_event`, `continuation`, `new_event`, `uncertain`
- confidence
- gerekçe için kısa evidence refs

### Validator

Görev:
- claim-source ayrımı
- kaynak çelişkisi
- bağımsız ikinci doğrulama
- `confirmed / supported / disputed / unverified`

### Worker

Görev:
- yalnızca atanmış event scope içinde derin araştırma
- scope creep yok
- yeni iddia bulursa claim olarak ayrı çıkar

### Synthesizer

Görev:
- doğrulanmış event/claim kayıtlarından kullanıcıya okunabilir çıktı üretme
- raw unsupported claim eklememe

## Neden skill?

Her çağrıda 2-5K token rol tarifi tekrar vermek yerine kısa, versioned ve test edilmiş davranış sözleşmesi kullanılır.

## Skill kalite kapısı

Her skill için fixture seti oluştur:

```text
tests/skills/scout/
tests/skills/analyst/
tests/skills/validator/
```

Ölç:
- schema validity
- false merge
- false split
- unsupported claim rate
- average input tokens
- average output tokens

Skill değişikliği prompt değişikliği gibi değil, kod değişikliği gibi versionlanmalı.

---

# 4. Öncelik 3 — OmniRoute: model gateway, ancak canary ile

Repository: `diegosouzapw/OmniRoute`

OmniRoute model çağrı yoluna girdiği için Agent Reach'ten daha yüksek blast radius taşır.

## İlk aşamada tüm Hermes'in önüne koyma

Yanlış:

```text
Hermes -> OmniRoute -> bütün production model çağrıları
```

İlk canary:

```text
non-critical worker
     |
     v
OmniRoute
     |
     +-> primary provider
     `-> fallback provider
```

Ana Hermes doğrudan mevcut provider yolunda kalır.

## İzole deployment

- loopback veya internal Docker network bind
- public dashboard/port yok
- provider secrets ayrı protected env/secret path
- n8n credentials ile paylaşma
- Hermes home mount etme
- Docker socket verme

## Routing stratejisi

World Pulse görevlerini sınıflandır:

```text
T0 deterministic       -> LLM yok
T1 cheap classification -> küçük/ucuz model
T2 event matching       -> orta reasoning
T3 evidence analysis    -> güçlü model
T4 final synthesis      -> güçlü model
```

OmniRoute ancak bu policy ölçüldükten sonra routing yapmalı.

## Compression sınırı

Compression için allowlist yaklaşımı kullan.

Compression uygulanabilir:
- terminal logs
- repetitive tool output
- Docker/build/test output
- verbose metadata
- known boilerplate

Compression uygulanmamalı:
- raw news article
- X post text
- direct quote
- evidence snippet
- URL/source identity
- dates/numbers
- attribution words (`claims`, `denies`, `reportedly`, vb.)

İstihbarat verisinde birkaç kelime epistemik anlamı değiştirebilir.

## Canary metrikleri

Direct provider ile OmniRoute worker'ı A/B karşılaştır:

- success rate
- p50/p95 latency
- input/output tokens
- cost/event
- routing/fallback count
- semantic output difference
- schema error rate
- provider error rate

## Kill switch

Tek config değişikliğiyle OmniRoute bypass edilebilmeli:

```text
worker -> direct provider
```

Gateway arızası World Pulse'ın tamamını durdurmamalı.

---

# 5. Öncelik 4 — Codebase Memory MCP: sadece development plane

Repository: `DeusData/codebase-memory-mcp`

Bu araç haber hafızası değildir. World Pulse kod tabanı büyüdüğünde coding agentin repo keşif tokenlarını azaltmak için kullanılır.

## Nerede çalışmalı?

Tercihen:

```text
local/dev machine
        |
        v
World Pulse source repo
        |
        v
Codebase Memory MCP
        |
        v
coding Hermes / development agent
```

Production runtime path'ine sokma.

## Güvenli ilk deneme

- release/commit pinle
- installer audit et
- otomatik client config istemiyorsan `--skip-config`
- MCP bağlantısını manuel yap
- yalnızca World Pulse source dizinini indexle
- UI kullanılacaksa `127.0.0.1` bind
- prod secrets/config dizinini indexleme
- global hooks/config modifications kapalı başla

## Ne zaman değerli hale gelir?

Aşağıdaki belirtiler olduğunda:

- agent aynı dosyaları tekrar tekrar grep/read ediyor
- repo onlarca/yüzlerce modüle çıktı
- cross-service call tracing zorlaştı
- code exploration tokenı feature implementasyonundan daha pahalı hale geldi
- impact analysis manuel zorlaştı

Önce baseline al, sonra MCP açıkken aynı coding benchmark'ını çalıştır. Kazanç ölçülmeden production/dev standardı yapma.

---

# 6. Ohm: bakım ve envanter; runtime değil

Repository: `derKosi/Ohm`

Ohm World Pulse data plane'e bağlanmaz.

Kullanım:

```text
host -> ohm scan -> inventory report -> human review
```

Kurallar:
- daemon yapma
- n8n workflow'a bağlama
- cleanup scripti otomatik çalıştırma
- production deploy sırasında kullanma
- bakım penceresinde read-only scan
- önce/sonra disk ve config inventory karşılaştırması

Yararı: zaman içinde VPS'te unutulmuş MCP, cache, agent config ve AI araçlarını bulmak.

---

# 7. Nero OSS: production'a kurma; mimari araştırma kaynağı

Repository: `pompeii-labs/nero-oss`

Nero'nun agent, MCP, background jobs, persistence, queues, UI ve host-runner bileşenleri Hermes+n8n mimarisiyle çakışır.

Bu nedenle:

```text
DO NOT DEPLOY INTO WORLD PULSE PRODUCTION
```

İncelenecek fikirler:
- knowledge-graph memory
- entity/event node modeli
- edge activation
- background project lifecycle

Alınmayacak şeyler:
- Nero host-runner
- Nero agent runtime
- Nero queue/persistence stack
- Nero UI/runtime

World Pulse kendi event/claim/evidence modelini kurmalı.

---

# 8. Mevcut katalogdaki JCode / Graphify / Ponytail için karar

Bunlar doğrudan bu entegrasyon dalgasına eklenmemeli.

## JCode

Önce PostgreSQL + pgvector + event retrieval ile gerçek ihtiyacı ölç. Eksik kalan uzun dönem agent memory problemi varsa yeniden değerlendir.

## Graphify

Event/entity veri modeli oturduktan sonra knowledge graph için değerlendir. Veri modeli stabil olmadan graph katmanı eklemek migration maliyeti doğurur.

## Ponytail

Runtime haber pipeline bileşeni değil. Coding/development agent davranışında ölçülebilir verim sorunu varsa ayrı benchmark ile değerlendir.

---

# 9. Uygulama sırası

```text
CURRENT CONTROL PLANE
Hermes + n8n + PostgreSQL + Agent Office
            |
            v
[BASELINE / SECURITY / ROLLBACK PASS]
            |
            v
1. Agent Reach — X read-only canary
            |
            v
2. World Pulse retrieval + dedupe
            |
            v
3. World Pulse native behavior skills
            |
            v
4. OmniRoute — one non-critical worker canary
            |
            v
5. Codebase Memory MCP — development plane only
            |
            +----> Ohm — maintenance-only
            |
            `----> Nero — research-only, no deployment
```

---

# 10. Her entegrasyon için zorunlu change protocol

Her yeni repo/bileşen için aynı prosedürü kullan:

## A. Audit
- canonical repository
- exact commit SHA/tag
- license
- installer behavior
- network listeners
- outbound requests
- filesystem writes
- credential access
- auto-update behavior
- telemetry
- subprocess/shell behavior
- Docker socket ihtiyacı
- root ihtiyacı

## B. Staging
- dedicated user/container
- minimal filesystem mounts
- minimal network
- loopback/internal binding
- test credentials
- synthetic data

## C. Canary
Sadece tek workflow/worker/source üzerinde aç.

## D. Measure
- latency
- CPU/RAM
- error rate
- token count
- cost
- duplicate rate
- quality metrics

## E. Failure test
- process kill
- network loss
- provider 429
- malformed payload
- DB unavailable
- restart/reboot

## F. Rollback
Bir komut/config değişikliğiyle eski baseline'a dön.

## G. Promote
Ancak canary ve failure testleri geçerse kapsamı artır.

---

# 11. Son hedef mimari

```text
                            +-------------------+
                            |   Agent Office    |
                            | observability only|
                            +---------^---------+
                                      |
                               sanitized events
                                      |
Sources                               |
  |                                   |
  +-- RSS/API ------------------+      |
  +-- X -> Agent Reach ---------+      |
  +-- Web/other collectors -----+      |
                               v      |
                         +-----------+ |
                         |    n8n    | |
                         |orchestrator| |
                         +-----+-----+ |
                               |       |
                  normalize/prefilter  |
                               |       |
                               v       |
                    +-------------------+
                    | World Pulse DB    |
                    | raw + event index |
                    | pgvector/retrieval|
                    +---------+---------+
                              |
                       Top-K candidates
                              |
                              v
                         +----------+
                         |  Hermes  |
                         | reasoning|
                         +----+-----+
                              |
                  +-----------+-----------+
                  |                       |
          native WP skills         worker canary
          Scout/Analyst/...              |
                  |                      v
                  |                 OmniRoute
                  |                (optional)
                  |                      |
                  +-----------+----------+
                              |
                              v
                    events/claims/evidence
                              |
                              v
                         WORLD PULSE

Development plane only:
World Pulse source -> Codebase Memory MCP -> coding agent

Maintenance only:
Host -> Ohm scan -> human review

Research only:
Nero OSS knowledge-graph concepts
```

Bu yapı, collector/orchestration/reasoning/observability/model-routing/development-memory rollerini birbirinden ayırır. Bir katmanın bozulması diğer katmanların yetki alanını genişletmez.
