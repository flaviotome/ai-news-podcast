# Pod.AI

Podcast diário e 100% automático sobre as novidades de AI Engineering das últimas 24 horas. Dois hosts, Lucas e Sarah, conversam em inglês por 2 a 3 minutos.

- Feed: `https://flaviotome.github.io/ai-news-podcast/feed.xml`
- Discovery: [docs/podcast-ai-engineering-plan.md](docs/podcast-ai-engineering-plan.md)
- Plano de implantação: [docs/implementation-plan.md](docs/implementation-plan.md)

## Branches

| Branch | Conteúdo |
|---|---|
| `main` | Código e documentação |
| `gh-pages` | Site público (feed, episódios, estado). Escrita só pelo n8n |

## Estrutura

| Pasta | Conteúdo |
|---|---|
| `docs/` | Discovery e plano |
| `infra/` | Dockerfile e compose do n8n |
| `workflows/` | Export do workflow do n8n |
| `prompts/` | Prompts dos hosts e pool de temas |
