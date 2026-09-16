# Hermes Execution Bundle — World Pulse Post-Stable Upgrade

Bu dosya Hermes'in uygulayacağı **icra planıdır**. Amaç konuşma notlarını production-safe, test edilebilir ve geri alınabilir entegrasyonlara çevirmektir.

## Kural 0 — kör kurulum yok

Hermes bu dosyayı okuduğunda önce mevcut host durumunu doğrular. Önceki konuşmalardan phase/state tahmin etmez.

Aşağıdaki baseline kapısı PASS olmadan hiçbir üçüncü-party repo production Hermes içine kurulmaz:

- Hermes mevcut görevlerini sağlıklı yürütüyor.
- n8n sağlıklı ve restart sonrası workflow state korunuyor.
- PostgreSQL sağlıklı; public listener yok; backup+restore testi mevcut.
- Agent Office yalnız observability; Hermes kapalı/Agent Office kapalı senaryolarında ana iş akışı etkilenmiyor.
- beklenmeyen public port yok.
- secret canary testi PASS.
- CPU/RAM/disk baseline kaydedildi.
- rollback snapshot/config backup hazır.

Baseline PASS değilse: **STOP, düzelt ve tekrar test et.**

---

# Wave 1 — Repo Engineer oluştur

## Amaç

Kullanıcı bir GitHub repo URL'si verdiğinde çalışan:

1. canonical repo + commit SHA tespit eder,
2. kodu çalıştırmadan static audit yapar,
3. repo architecture/domain modelini çıkarır,
4. World Pulse capability graph ile karşılaştırır,
5. INSTALL / ADAPT / IDEA_ONLY / REJECT kararı verir,
6. yalnız izole worktree/branch içinde kod yazar,
7. test ve benchmark yapar,
8. production'a dokunmadan PR/patch hazırlar.

## Runtime identity

Ayrı Unix kullanıcısı önerisi:

```text
repoengineer
```

Yetkiler:

```text
Internet/public GitHub         ALLOW
External repo clone            ALLOW
World Pulse source             READ
Dedicated worktree             WRITE
Integration branch             WRITE
Test DB/test fixtures           ALLOW

sudo/root                       DENY
production DB credentials       DENY
n8n credentials                 DENY
Hermes provider secrets         DENY
~/.ssh private keys             DENY
Docker socket                   DENY by default
production systemd              DENY
main branch direct write        DENY
PR merge                         DENY
```

## Repo Engineer skill

Kaynak: `hermes/skills/repo-engineer/SKILL.md`

Bu skill production Hermes genel rolüne eklenmez; Repo Engineer session/worker'a scoped yüklenir.

---

# Wave 2 — Egonex Understand Anything: Repo Engineer'ın semantic repo map motoru

Canonical repo:

```text
https://github.com/Egonex-AI/Understand-Anything
```

## Kurulum politikası

- production Hermes kullanıcısına global `curl | bash` YAPMA.
- önce exact commit SHA pinle.
- installer ve package scripts audit et.
- ayrı `repoengineer` HOME altında kur.
- external repo analizi sandbox/worktree içinde yapılır.
- external repo'nun install scriptleri otomatik çalıştırılmaz.
- `.ua/` / knowledge graph çıktıları çalışma verisidir; production secret path'leri input yapılmaz.

## Kullanım

```text
External repo
  -> static audit
  -> Egonex / understand analysis
  -> architecture + dependency + domain graph
  -> Hermes Repo Engineer reasoning
```

İlk analiz token açısından pahalı olabilir. Sonraki analizlerin incremental olması beklenir. Her repo için initial_cost ve incremental_cost ölçülür.

## Acceptance

- test repo üzerinde graph üretimi PASS
- repo dışındaki dosyalara write yok
- production HOME/secrets erişimi yok
- tekrar analizde yalnız değişikliklere odaklanıyor
- generated graph Repo Engineer tarafından sorgulanabiliyor

---

# Wave 3 — Agent Reach: X/Twitter read-only ingestion

Canonical repo:

```text
https://github.com/Panniantong/Agent-Reach
```

## Amaç

World Pulse'ın X sinyal katmanını açmak.

## Mimari

```text
X
 -> Agent Reach / dedicated X adapter
 -> n8n
 -> normalize
 -> cheap prefilter
 -> retrieval/dedupe
 -> Hermes
```

## Güvenlik

- ayrı service user
- read/search only
- tweet/DM/like/follow/post capability açma
- X auth/cookie bilgilerini n8n workflow JSON'una koyma
- Hermes promptuna credential koyma
- Agent Office event feed'e credential koyma
- arbitrary shell endpoint açma

Tercih edilen dar adapter API:

```text
POST /v1/x/search
GET  /v1/x/timeline
GET  /health
```

İlk canary: tek düşük hacimli watch query.

Ölç:
- success rate
- rate limits
- duplicate ratio
- items/min
- n8n execution latency
- Hermes'e ulaşan item oranı
- token/item

---

# Wave 4 — World Pulse Retrieval + Deduplication

Bu üçüncü-party installer değildir; World Pulse core özelliğidir.

## Akış

```text
incoming item
 -> exact id/url hash
 -> normalized fingerprint
 -> actor/country/type/time filter
 -> vector/semantic candidate retrieval
 -> Top-K events
 -> Hermes: same_event | continuation | new_event | uncertain
```

PostgreSQL source-of-truth kalır. n8n database ile World Pulse domain database'i ayrılır.

Önerilen domain tabloları:

```text
raw_items
articles
social_posts
events
event_sources
entities
claims
evidence
ingestion_runs
```

Gerekirse `pgvector` eklenir.

Raw haber/tweet destructive compression ile değiştirilmez.

---

# Wave 5 — World Pulse native behavior skills

`Arete-Consortium/ai-skills` yalnız tasarım referansıdır. Global bundle install etme.

Hermes-native küçük skill'ler oluştur:

```text
scout
analyst
validator
worker
synthesizer
```

Her skill için:
- dar rol
- typed input
- typed output
- explicit NEVER rules
- test fixtures
- version

Özellikle analyst output:

```json
{
  "decision": "same_event|continuation|new_event|uncertain",
  "event_id": null,
  "confidence": 0.0,
  "evidence_refs": []
}
```

---

# Wave 6 — Graphify: system graph / provenance graph deneyi

Canonical repo:

```text
https://github.com/Graphify-Labs/graphify
```

Graphify production source-of-truth DEĞİLDİR. PostgreSQL canonical truth kalır.

İki logical graph tanımla:

```text
WORLD PULSE GRAPH
Event / Claim / Source / Entity / Country / Company / Technology

SYSTEM GRAPH
Service / Agent / Skill / Workflow / Repository / Module / API / Database / Provider
```

Provenance ile bağlanabilir:

```text
Source -> collected_by -> Collector
Collector -> orchestrated_by -> n8n Workflow
Workflow -> analyzed_by -> Hermes Worker
Hermes Worker -> produces -> Claim/Event
```

Graph inputlarından secret/config credential dosyalarını hariç tut.

İlk kullanım production news graph değil; **World Pulse source/docs üzerinde system graph prototype**.

Acceptance:
- graph yeniden üretilebilir
- EXTRACTED/INFERRED ayrımı korunur
- PostgreSQL/veri kaybı olmadan graph silinip yeniden build edilebilir
- Hermes yalnız scoped subgraph alır; tüm graph her prompta taşınmaz

---

# Wave 7 — OmniRoute: yalnız non-critical worker canary

Canonical repo:

```text
https://github.com/diegosouzapw/OmniRoute
```

Ana Hermes provider yolunu ilk aşamada değiştirme.

```text
Main Hermes -> current provider

Non-critical worker -> OmniRoute -> primary/fallback
```

Kurallar:
- internal/loopback bind
- ayrı provider secrets
- Docker socket verme
- production raw news compression yapma
- kill switch ile direct provider'a anında dönülebilmeli

Compression allowlist:
- terminal logs
- repetitive tool output
- build/test output
- boilerplate metadata

Compression denylist:
- raw articles
- X post text
- quotes
- dates/numbers
- source identity/URL
- epistemic attribution (`claims`, `denies`, `reportedly`, vb.)

A/B ölç:
- p50/p95 latency
- tokens/event
- cost/event
- schema error
- provider error
- fallback count
- semantic difference

---

# Wave 8 — Codebase Memory MCP: development plane only

Canonical repo:

```text
https://github.com/DeusData/codebase-memory-mcp
```

Egonex'in yerine değil, gerekirse structural code intelligence tamamlayıcısı olarak eklenir.

Trigger condition:
- World Pulse repo büyüdü
- grep/read döngüleri aşırı arttı
- cross-service impact analysis zorlaştı
- coding token maliyeti belirginleşti

İlk deneme:
- exact release/commit pin
- installer audit
- mümkünse `--skip-config`
- manuel MCP wiring
- yalnız source repo index
- secrets/prod data hariç
- UI varsa localhost bind

Kazanç benchmark ile kanıtlanmadan standart tool yapma.

---

# Ohm / Nero politikası

## Ohm
Maintenance-only. Read-only inventory. Cleanup script otomatik çalıştırılmaz.

## Nero OSS
Production'a kurulmaz. Knowledge-graph memory ve background-project design fikirleri incelenebilir; agent runtime/queue/UI/host-runner stack alınmaz.

---

# Değişiklik protokolü

Her entegrasyon aynı sıradan geçer:

```text
AUDIT
 -> STAGE
 -> CANARY
 -> MEASURE
 -> FAILURE TEST
 -> ROLLBACK TEST
 -> PROMOTE
```

Asla iki yeni kritik bileşeni aynı anda production'a sokma.

Her adım sonunda `WORLD_PULSE_DEPLOYMENT_STATE.md` veya aktif state dosyasına:
- timestamp
- exact repo/commit
- config değişiklikleri
- ports/listeners
- credentials touched (secret değerleri olmadan)
- test result
- rollback command/path
- PASS/FAIL
kaydet.

---

# Nihai hedef

```text
Sources
  |- RSS/API
  |- X -> Agent Reach
  `- Web
       |
       v
      n8n
       |
 normalize/prefilter
       |
       v
World Pulse PostgreSQL + retrieval
       |
   Top-K context
       |
       v
     Hermes
  /   |    \
Scout Analyst Validator -> Workers
       |
       v
 events/claims/evidence
       |
       v
 WORLD PULSE

Side planes:
- Agent Office = observability only
- Repo Engineer + Egonex = integration/development plane
- Graphify = derived graph/index plane
- OmniRoute = optional model-gateway canary
- Codebase Memory = optional development code intelligence
```
