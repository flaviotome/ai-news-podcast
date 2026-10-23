# Plano de implantação

> Base: [podcast-ai-engineering-plan.md](podcast-ai-engineering-plan.md). Última atualização: 2026-10-23.

## Status atual

| Fase | Situação |
|---|---|
| 0. Fundação | Quase pronta. Falta anotar as cotas do Gemini e ouvir as vozes |
| 1. Infraestrutura da VM | Infra pronta. Faltam as credenciais no n8n e o workflow de teste |
| 2. Roteiro e áudio | Não iniciada (os prompts podem ser escritos antes de a Fase 1 fechar) |
| 3–5 | Não iniciadas |

**Próximos passos**

1. Terminar a credencial do GitHub no n8n (User = `flaviotome` + token `n8n-podai`). Pausado pelo Flávio.
2. Criar a chave do Gemini. A opção "Get API key" não apareceu no AI Studio; tentar `https://aistudio.google.com/apikey` ou criar pelo Cloud Console (Generative Language API → Credentials → API key).
3. Criar o bot do Telegram (@BotFather), pegar o chat ID e cadastrar a credencial.
4. Rodar o workflow de teste: Manual Trigger → Execute Command (`/files/bin/ffmpeg -hide_banner -version`) → Telegram.
5. Começar a Fase 2 pelos prompts do Lucas e da Sarah e pelo pool de temas.

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
│   └── daily-episode.json # export do workflow do n8n (fonte da verdade)
└── prompts/
    ├── system-news.md     # personas + regras, dia com notícias
    ├── system-slow-day.md # personas + regras, dia parado
    └── topics.json        # pool de ~30 temas leves

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
| Áudio | ffmpeg: PCM 24 kHz do Gemini → `loudnorm` (-16 LUFS) → MP3 64 kbps mono | Volume consistente entre episódios, ~1,4 MB por episódio |
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
- [ ] Spike de 30 minutos no AI Studio: confirmar cotas atuais do Gemini (texto e TTS) e ouvir pares de vozes. Depende da chave do Gemini (ver Fase 1).

**Pronto quando:** `https://flaviotome.github.io/ai-news-podcast/` responde 200 e as cotas do free tier estão anotadas.

### Fase 1: infraestrutura da VM

- [x] `ffmpeg` e `ffprobe` 7.1 estáticos em `/local-files/bin`.
- [x] Pasta de trabalho `/local-files/podcast`, com dono = usuário do container do n8n.
- [x] [`infra/docker-compose.override.yml`](../infra/docker-compose.override.yml) em `/docker/n8n`: fixa a imagem em 2.14.2, libera o Execute Command (`NODES_EXCLUDE=[]`) e permite arquivos em `/files` (`N8N_RESTRICT_FILE_ACCESS_TO`).
- [ ] Credencial "GitHub API" no n8n: Server `https://api.github.com`, User `flaviotome`, token fine-grained `n8n-podai` (só `ai-news-podcast`, Contents: read/write, validade de 1 ano). Em andamento.
- [ ] Credencial "Google Gemini (PaLM) API" no n8n. A chave ainda não foi criada.
- [ ] Credencial "Telegram API" no n8n (bot via @BotFather) e chat ID anotado.
- [ ] Workflow de teste: Manual Trigger → Execute Command → Telegram.

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

- [ ] Coleta: RSS Read para cada fonte, merge.
- [ ] Tratamento (Code): janela de 24h, limpeza de HTML, dedup por URL/título, top ~8.
- [ ] Roteiro: prompt com personas, saída JSON `[{speaker, text}]`.
- [ ] Validação: schema + 350–450 palavras, retry até 2x.
- [ ] TTS multi-speaker → WAV → MP3 em disco.
- [ ] Rodar manualmente por alguns dias e ouvir. Ajustar prompt e vozes.

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
