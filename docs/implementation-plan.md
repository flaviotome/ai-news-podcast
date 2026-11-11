# Plano de implantação

> Base: [podcast-ai-engineering-plan.md](podcast-ai-engineering-plan.md). Última atualização: 2026-11-11.

## Status atual

| Fase | Situação |
|---|---|
| 0. Fundação | Quase pronta. Falta anotar as cotas do Gemini |
| 1. Infraestrutura da VM | Pronta, exceto o Telegram. GitHub, Gemini (texto e TTS) e ffmpeg validados num workflow de teste em 2026-10-23 |
| 2. Roteiro e áudio | Em andamento. Workflow [`daily-episode.json`](../workflows/daily-episode.json) e prompts prontos e testados localmente; falta importar no n8n e rodar |
| 3–5 | Não iniciadas |

**Próximos passos**

1. Push do `main` (o workflow baixa os prompts do GitHub) e importação do workflow no n8n.
2. Rodar o workflow manualmente por alguns dias, ouvir os episódios e ajustar prompts e vozes.
3. Adiados a pedido do Flávio: bot do Telegram e cotas do free tier do Gemini. Sem as cotas, o risco é estourar o limite do TTS em dias de muitos testes.
4. Decidir o destino do workflow vazio "AI News Podcast v1" (`SOVnvLkWqicYtaIK`): apagar, já que o diário é um workflow novo.

## 1. Ponto de partida

| Item | Estado em 2026-10-23 |
|---|---|
| Repositório | [flaviotome/ai-news-podcast](https://github.com/flaviotome/ai-news-podcast), público, com `main` e `gh-pages` |
| GitHub Pages | No ar, servindo a `gh-pages` |
| Pasta local | Clone do repositório, branch `main` |
| URL do feed (definitiva) | `https://flaviotome.github.io/ai-news-podcast/feed.xml` |
| VM | Hostinger, Ubuntu 24.04, 3,8 GB de RAM, 43 GB livres. Docker 29.3, Compose v5.1 |
| n8n | 2.14.2, fixada pelo override (no compose da Hostinger a imagem não tinha tag). Atrás de Traefik. Compose em `/docker/n8n/docker-compose.yml` (template da Hostinger) |
| Imagem do n8n | "Docker Hardened Images (Alpine)": sem `apk`/`apt-get` e sem ffmpeg próprio; tem `git`. ffmpeg vem de `/files/bin` |
| Pastas | Volume `n8n_data` em `/home/node/.n8n`; host `/local-files` montado em `/files` |
| Fuso global | `GENERIC_TIMEZONE=Europe/Berlin` (usado por outros workflows; não mexer) |
| Banco do n8n | SQLite em `n8n_data/database.sqlite`. Execuções antigas são apagadas pelo n8n (em 2026-11-11 não havia nenhuma do teste de 23/10) |
| Outros workflows | ~100 templates importados em 2025, todos inativos (fora do escopo) |
| Outros containers | `whatsapp-mcp` em `/opt/whatsapp-mcp` (fora do escopo) |
| Acesso à VM | Sem SSH para o Claude: todo comando na VM é entregue para o Flávio copiar e colar |

A URL do feed fica fixa a partir daqui. Se um domínio próprio entrar depois, o GitHub redireciona com 301.

### Identidade do programa

| Item | Valor |
|---|---|
| Nome | Pod.AI |
| Hosts | Lucas (speaker A, voz masculina) e Sarah (speaker B, voz feminina) |
| Dono do feed (`itunes:owner`) | Flávio Tomé, flaviotome5@gmail.com |

O e-mail do `itunes:owner` fica público dentro do `feed.xml`. Um alias como `flaviotome5+podai@gmail.com` chega na mesma caixa e facilita filtrar spam.

### Como mexer na VM

Como não há SSH a partir daqui, cada mudança na VM segue o mesmo roteiro:

1. Comandos de leitura primeiro; a saída volta para cá com segredos mascarados.
2. Backup do arquivo antes de qualquer edição (`cp arquivo arquivo.bak-AAAAMMDD`).
3. Mudança em um bloco de comandos, seguida de um comando de verificação.
4. Comando de rollback documentado junto.

## 2. Organização do repositório

Um repositório, duas branches com papéis separados:

```text
main                       # código e documentação (versionado por mim)
├── docs/                  # discovery + este plano
├── infra/
│   └── docker-compose.override.yml # o que mudamos no compose da Hostinger
├── workflows/
│   ├── test-infra.json    # workflow de teste da Fase 1
│   └── daily-episode.json # export do workflow do n8n (fonte da verdade)
└── prompts/               # baixados pelo workflow a cada execução (branch main)
    ├── hosts.md           # personas, estilo, regras de fatos e formato de saída
    ├── system-news.md     # tarefa do dia com notícias
    ├── system-slow-day.md # tarefa do dia parado
    ├── show.json          # intro e outro fixos
    └── topics.json        # pool de 30 temas leves

gh-pages (órfã)            # site público, escrito só pelo n8n
├── .nojekyll
├── index.html             # página mínima com link para o feed
├── cover.jpg              # 3000x3000
├── feed.xml               # gerado a cada execução a partir do state.json
├── state.json             # episódios, URLs cobertas, temas usados
└── episodes/
    └── 2026-11-03.mp3
```

Por que separar:

- Os commits diários do n8n não poluem o histórico do código.
- O MP3 fica só na `gh-pages`, que pode ter o histórico "achatado" (ver 4.4) sem tocar no `main`.
- O `state.json` substitui o static data do n8n: fica versionado, visível e sobrevive a uma reinstalação da VM.

## 3. Decisões técnicas complementares

Detalhes que o discovery deixou em aberto, com a escolha proposta:

| Tema | Proposta | Motivo |
|---|---|---|
| Como publicar | Git Data API (blobs → tree → commit → ref) via nó HTTP Request | Um único commit atômico com MP3 + `state.json` + `feed.xml`. O feed nunca aponta para um MP3 que ainda não existe. Sem `git` na imagem e sem token em variável de ambiente |
| Fonte da verdade do feed | `state.json`; o `feed.xml` é sempre regenerado inteiro | A retenção de 90 dias vira só um filtro na lista de episódios |
| Idempotência | `guid` = data do episódio. Se o `state.json` já tem o episódio do dia, o fluxo para | Reexecução manual ou dupla não publica duas vezes |
| Áudio | ffmpeg: PCM 24 kHz do Gemini → `loudnorm` (-16 LUFS) → MP3 44,1 kHz, 64 kbps mono | Volume consistente entre episódios, ~1,2 MB por episódio de 2,5 min. Validado no teste de infra |
| Crescimento do repositório | Mensalmente, recriar a `gh-pages` como um commit sem pais (mesma tree) e forçar o ref | MP3 apagados pela retenção somem do histórico; o repositório não cresce ~500 MB/ano |
| Alerta de "não rodou" | Opcional: ping no healthchecks.io (free) ao final de cada execução | O Error Trigger não dispara se a VM ou o n8n estiverem fora do ar |
| ffmpeg | Binários estáticos (`mwader/static-ffmpeg`) em `/local-files/bin`, vistos pelo n8n como `/files/bin/ffmpeg` | A imagem hardened não tem gerenciador de pacotes. Sem imagem customizada para manter e sem rebuild a cada upgrade do n8n |
| Mudanças no compose | `docker-compose.override.yml` ao lado do arquivo da Hostinger | O arquivo original fica intacto; rollback é apagar o override |
| Fuso do podcast | Definido nas configurações do próprio workflow (America/Sao_Paulo) | O fuso global (Berlin) atende outros workflows. A janela de 24h usa timestamps absolutos, então o fuso só afeta o horário do cron e a data do episódio |

## 4. Fases

### Fase 0: fundação

- [x] `git init`, mover o discovery para `docs/`, primeiro push no `main`.
- [x] Criar a branch órfã `gh-pages` com `.nojekyll` e `index.html`.
- [x] Ligar o Pages na `gh-pages` (Settings → Pages → Deploy from a branch → `gh-pages` / root).
- [x] Levantar a VM com comandos só de leitura: SO, versão do n8n, onde está o compose, imagem, se já há ffmpeg, variáveis relevantes e fuso.
- [x] Primeiro par de vozes: Puck (Lucas) e Aoede (Sarah), usado no teste de 2026-10-23. Pode ser trocado na Fase 2.
- [ ] Anotar as cotas atuais do free tier do Gemini (texto e TTS) para os modelos candidatos (ver "Modelos disponíveis" na Fase 1).

**Pronto quando:** `https://flaviotome.github.io/ai-news-podcast/` responde 200 e as cotas do free tier estão anotadas.

### Fase 1: infraestrutura da VM

- [x] `ffmpeg` e `ffprobe` 7.1 estáticos em `/local-files/bin`.
- [x] Pasta de trabalho `/local-files/podcast`, com dono = usuário do container do n8n.
- [x] [`infra/docker-compose.override.yml`](../infra/docker-compose.override.yml) em `/docker/n8n`: fixa a imagem em 2.14.2, libera o Execute Command (`NODES_EXCLUDE=[]`) e permite arquivos em `/files` (`N8N_RESTRICT_FILE_ACCESS_TO`).
- [x] Credencial `githubApi` "GitHub account" no n8n: Server `https://api.github.com`, User `flaviotome`, token fine-grained `n8n-podai` (só `ai-news-podcast`, Contents: read/write). Criada em 2026-10-22; o token vence por volta de 2027-10-22.
- [x] Credencial `googlePalmApi` "Google Gemini(PaLM) Api account 2" no n8n. Criada em 2026-10-22.
- [ ] Credencial "Telegram API" no n8n (bot via @BotFather) e chat ID anotado.
- [x] Workflow de teste [`workflows/test-infra.json`](../workflows/test-infra.json) ("Pod.AI - teste de infra", `PodAiTestInfra01`), rodado em 2026-10-23. Falta o nó do Telegram.

Resultado do teste (gravado em `/files/podcast/test-result.json`):

| Verificação | Resultado |
|---|---|
| ffmpeg / ffprobe pelo Execute Command | 7.1, e escrita em `/files/podcast` OK |
| GitHub: `POST /git/blobs` com a credencial | Blob criado (`cd5fb79`), então o token tem Contents: write. O blob é solto e o GitHub descarta |
| Gemini texto (`gemini-2.5-flash`) | Respondeu "Pod.AI text OK" |
| Gemini TTS multi-speaker (`gemini-2.5-flash-preview-tts`) | Devolveu `audio/L16;codec=pcm;rate=24000` |
| PCM → `loudnorm` → MP3 | `test.mp3`: 17,1 s, mono, 44,1 kHz, 64 kbps, 137 KB |

Modelos disponíveis para a chave em 2026-10-23:

| Uso | Modelos |
|---|---|
| Texto (flash) | `gemini-2.5-flash`, `gemini-2.5-flash-lite`, `gemini-3-flash-preview`, `gemini-3.1-flash-lite`, `gemini-3.5-flash`, `gemini-3.5-flash-lite`, `gemini-3.6-flash`, `gemini-3.7-flash`, `gemini-3.8-flash`, `gemini-flash-latest`, `gemini-flash-lite-latest` e outros |
| TTS | `gemini-2.5-flash-preview-tts`, `gemini-2.5-pro-preview-tts`, `gemini-3.1-flash-tts-preview`, `gemini-3.8-flash-tts`, `gemini-3.8-flash-lite-tts` |

O teste usou os modelos 2.5. Vale comparar com a geração 3.8 na Fase 2 (qualidade e cota).

Alterações feitas na VM em 2026-10-22, todas verificadas:

| O quê | Onde |
|---|---|
| Imagem `mwader/static-ffmpeg:7.1` baixada (só para extrair os binários) | Docker local |
| `ffmpeg` e `ffprobe` 7.1 (~135 MB cada), `root:root`, 755 | `/local-files/bin` → `/files/bin` no container |
| Pasta de trabalho do áudio, dono `ubuntu:ubuntu` (mesmo UID do usuário do n8n) | `/local-files/podcast` → `/files/podcast` |
| Override com imagem 2.14.2, `NODES_EXCLUDE=[]` e `N8N_RESTRICT_FILE_ACCESS_TO=/files` | `/docker/n8n/docker-compose.override.yml` |
| Container `n8n-n8n-1` recriado com o override | `docker compose up -d n8n` |

Rollback: `rm /docker/n8n/docker-compose.override.yml` e `docker compose up -d n8n` em `/docker/n8n`.

Com o Execute Command ligado, quem entra no n8n consegue rodar comandos no container. Como o n8n está exposto na internet, a conta precisa de senha forte e, se possível, 2FA.

**Pronto quando:** um workflow de teste roda `/files/bin/ffmpeg -version` num Execute Command e manda uma mensagem no Telegram.

### Fase 2: MVP do roteiro e do áudio (sem publicar)

- [x] Coleta: Hacker News pela API do Algolia (histórias com 40+ pontos nas últimas 24h, filtradas por palavras-chave de AI) + um nó RSS para 11 feeds.
- [x] Tratamento (Code): janela de 24h, limpeza de HTML, dedup por URL e título (a mesma notícia em duas fontes vira um item só), peso e limite por fonte, top 10.
- [x] Roteiro: personas e regras em `prompts/`, saída JSON com `title`, `summary`, `covered` e `lines` (schema forçado no Gemini).
- [x] Validação: falantes, mínimo de 8 falas, 350–450 palavras no total. Até 3 tentativas, devolvendo ao modelo o motivo da reprovação.
- [x] Dia parado (< 3 itens): tema do `topics.json` escolhido pelo número do dia, então não se repete em menos de 30 dias. Sem estado.
- [x] TTS multi-speaker → PCM → MP3 em disco, com o relatório do episódio em `.json` ao lado.
- [ ] Importar no n8n e rodar a primeira vez.
- [ ] Rodar manualmente por alguns dias e ouvir. Ajustar prompt e vozes.

Fontes RSS (nó "Fontes"): Simon Willison, Hugging Face, OpenAI, Google AI, Google for Developers, GitHub (AI & ML), Ollama, Latent Space, Interconnects, Sebastian Raschka e r/LocalLLaMA. Ficaram de fora: Google DeepMind (o feed vem comprimido de um jeito que o nó RSS não lê), Anthropic e LangChain (sem feed público). A busca do hnrss.org estava dando 502, por isso o Hacker News vem do Algolia.

Limitação conhecida: itens do Hacker News e do Hugging Face chegam só com o título, e o modelo é instruído a não inventar detalhes. Se os roteiros ficarem rasos, o próximo passo é buscar a descrição (meta tags) das páginas escolhidas.

Saída de cada execução em `/files/podcast/AAAA-MM-DD-HHMM.mp3` e `.json` (roteiro, fontes cobertas, estatísticas da coleta e avisos). O sufixo de hora evita sobrescrever episódios de teste do mesmo dia.

**Pronto quando:** 3 episódios seguidos saem bons sem ajuste manual.

### Fase 3: publicação

- [x] Nome do podcast, hosts e e-mail do dono definidos (ver seção 1).
- [ ] Capa 3000x3000.
- [ ] Ler `state.json` da `gh-pages` no início do fluxo; usar para o dedup histórico.
- [ ] Gerar `feed.xml` (RSS 2.0 + tags iTunes, `itunes:duration`, `enclosure length` em bytes, show notes com as fontes e aviso de conteúdo gerado por IA).
- [ ] Commit atômico na `gh-pages`.
- [ ] Validar o feed num validador de podcast (Cast Feed Validator ou similar).

**Pronto quando:** o feed valida sem erros e o MP3 toca a partir da URL pública.

### Fase 4: robustez

- [ ] Caminho de dia parado: IF < 3 itens, pool de temas, sem repetir em 30 dias.
- [ ] `ffprobe`: duração entre 1,5 e 4 min, senão aborta.
- [ ] Fallback de TTS (Google Cloud TTS, duas vozes, chamadas por fala e concatenação no ffmpeg).
- [ ] Error Trigger → Telegram. Nunca publicar episódio quebrado.
- [ ] Retenção de 90 dias e o achatamento mensal da `gh-pages`.

**Pronto quando:** forçar cada falha (LLM, TTS, GitHub) gera alerta e nada é publicado.

### Fase 5: go-live

- [ ] Ligar o cron no horário escolhido.
- [ ] Rodar 5–7 dias publicando no feed, ainda sem cadastrar nas plataformas.
- [ ] Cadastrar o feed no Spotify for Creators, Apple Podcasts Connect e YouTube Studio (ingestão de RSS); marcar como conteúdo gerado por IA onde houver opção.
- [ ] Exportar o workflow final para `workflows/daily-episode.json`.

## 5. Decisões pendentes e quando bloqueiam

| Decisão | Bloqueia | Sugestão |
|---|---|---|
| ~~Dados da VM (versão do n8n, como roda)~~ | — | Levantado: ver seção 1 |
| ~~Nomes dos hosts~~ | — | Decidido: Lucas e Sarah |
| ~~Nome do podcast~~ | — | Decidido: Pod.AI |
| Capa 3000x3000 | Fase 3 | Pode ser gerada por IA |
| ~~E-mail do `itunes:owner`~~ | — | Decidido: Flávio Tomé, flaviotome5@gmail.com |
| Horário e fuso | Fase 5 | Rodar ~06:00 America/Sao_Paulo, com janela de 24h até esse horário |
| Categoria / explicit | Fase 3 | Technology, `explicit = false` |
| Domínio próprio | Nada | Decidir depois |
