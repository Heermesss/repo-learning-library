# AI Repo Learning Library

Bu repository, "Repo Öğrenimi" sohbetinde incelediğimiz AI/agent araçlarını tek yerde tutar.

## Çalışma kuralı

1. Bir GitHub reposunu sohbette inceleriz.
2. Kullanıcı **"ekle"** dediğinde ilgili repo bu kütüphaneye eklenir.
3. `data/repos.json` katalog için resmi veri kaynağıdır.
4. `index.html` bu kataloğu tarayıcıda kartlar halinde gösterir.

## World Pulse entegrasyon planı

World Pulse'ın mevcut **Hermes + n8n + PostgreSQL + Agent Office** kurulumu stabil olduktan sonra hangi araçların hangi sırayla, hangi güvenlik sınırlarıyla ve hangi canary/rollback prosedürüyle uygulanacağı burada tutulur:

- [`docs/WORLD_PULSE_POST_STABLE_INTEGRATION_PLAN.md`](docs/WORLD_PULSE_POST_STABLE_INTEGRATION_PLAN.md)

Planın ana sırası: **Agent Reach (X read-only) → retrieval/dedupe → World Pulse-native behavior skills → OmniRoute canary → Codebase Memory MCP (development only)**. Ohm maintenance-only, Nero OSS ise research-only tutulur.

## Başlangıç kayıtları

- Agent Office — observability
- JCode — memory / retrieval
- Graphify — knowledge graph
- Ponytail — agent efficiency

## Sonradan eklenen World Pulse adayları

- Codebase Memory MCP — code intelligence / token efficiency
- Agent Reach — X/Twitter ve internet source access
- AI Skills / Behavior Skills — agent behavior tasarım referansı
- Ohm — offline maintenance / inventory
- Nero OSS — knowledge-graph memory araştırma referansı
- OmniRoute — model gateway / fallback / compression canary

Bu repository'ye API anahtarı, parola, token, cookie veya başka bir secret eklenmemelidir.
