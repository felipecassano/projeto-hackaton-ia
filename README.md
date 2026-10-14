# Energy Start

> Inteligência regulatória para a transição energética. O Energy Start lê diariamente as publicações da ANEEL, do MME e do CNPE no Diário Oficial da União, resume o que mudou, cruza as regras novas com o perfil da usina e avisa a empresa quando algo afeta a sua operação.

Grupo 2 — Hackathon IA 2026.

### 💻 Minha Contribuição no Projeto

Projeto desenvolvido durante o **Hackathon IA COPPE 2026** (Grupo 02). Atuei como desenvolvedor principal na arquitetura e implementação ponta a ponta da solução:

- **Backend (`api/`):** Arquitetura e desenvolvimento da API REST em **NestJS + TypeORM** com **PostgreSQL 16**, modelagem de normas/trechos, motor de cruzamento de limites regulatórios com o perfil da usina e sistema de disparo de alertas.
- **Frontend (`web/`):** Interface em **React + Vite + Tailwind CSS**, integrando o painel de monitoramento, central de alertas, feed de notícias do setor e interface de chat do copiloto com citações.
- **Integração de IA & Infra (`ai/` e `infra/`):** Integração do fluxo de ingestão do DOU e RAG (`/ask`) via API, conteinerização com **Docker Compose** e configuração para deploy na AWS.

## Como funciona

```
ai/ (Python)                    api/ (NestJS)                          web/ (React)
coleta do DOU ──▶ classificador ──▶ resumidor ──POST /interno/ingestao──▶ normas, trechos, limites ──▶ painel, resumos
                  (área/subárea)    (mudanças,                            motor de alertas ──────────▶ central de alertas
                                     prazos)                              (norma nova na área,          + e-mail
                                                                           limite × perfil da usina)
                                    RAG (/ask) ◀──── copiloto ◀───────────────────────────────────── chat com citações
```

- **Ingestão** ([ai/](ai/)): coleta as publicações do dia no DOU, classifica cada uma por área e subárea (Solar, Eólica, Armazenamento…) e como ato geral ou individual, e extrai, com trechos conferidos contra o texto original, o que mudou, os prazos e quem é afetado. Ato individual (despacho que libera, transfere ou multa uma usina — a maior parte do DOU) só segue se citar um cliente pelo CNPJ ou pelo CEG da usina.
- **API** ([api/](api/)): recebe esse resultado, guarda as normas quebradas em trechos por artigo e dispara os alertas: norma nova numa área que a usina monitora, publicação que cita a empresa, ou limite regulatório que os dados da usina não cumprem. Os e-mails saem na hora ou em resumo diário ou semanal, conforme o perfil. Os endpoints estão em [api/ENDPOINTS.md](api/ENDPOINTS.md).
- **Web** ([web/](web/)): painel, resumos (com PDF e link para o documento original), central de alertas, notícias do setor (coletadas pela API a cada 3 horas de ABSOLAR, ABEEólica, PV Magazine Brasil, MegaWhat, ANEEL e MME, filtradas pelas áreas da empresa) e o copiloto.

## Tecnologias utilizadas

- Linguagens: TypeScript e Python
- Frameworks: NestJS + TypeORM (API), React + Vite + Tailwind CSS (web), pandas (pipeline)
- Banco de dados: PostgreSQL 16
- APIs / serviços externos: Diário Oficial da União (in.gov.br) e modelos de linguagem via API da NVIDIA
- Deploy: AWS Fargate via CDK ([infra/](infra/))

## Como rodar o projeto

```bash
git clone https://github.com/felipecassano/projeto-hackaton-ia.git
cd projeto-hackaton-ia
cp .env.example .env          # preencha DB_*, JWT_SECRET e INTERNAL_API_KEY

docker compose up             # Postgres + API (http://localhost:3333) + web (http://localhost:5173) + serviço de IA
```

Sem Docker, rode cada parte:

```bash
cd api && npm install && npm run start:dev    # precisa de um Postgres no DB_HOST/DB_PORT do .env
cd web && npm install && npm run dev          # VITE_USE_MOCK=true roda o front sem backend
cd ai  && pip install -r requirements.txt && python main.py
```

Testes da API: `cd api && npx jest`.

### Copiloto

O copiloto (`ai/servidor.py`) responde com base nas normas da base — as **canônicas** (normas-base em vigor) e as **novidades** que o pipeline traz do DOU — e, quando a base não responde, busca no próprio DOU. Cada afirmação sai com a fonte [n], o trecho e o link oficial.

```bash
# no docker-compose (o serviço "copiloto" já sobe com o compose)
docker compose run --rm ai python carregar_canonicas.py     # normas-base (uma vez; rodar de novo atualiza)
docker compose run --rm ai python main.py 25-09-2026        # novidades de um dia do DOU (sem data = hoje)

# sem Docker
cd ai && python carregar_canonicas.py && uvicorn servidor:app --port 8000
```

O pipeline é rodado à mão (não há agendamento). Por padrão, os modelos (classificador, resumidor, copiloto e embeddings) são os da NVIDIA. Com `BEDROCK_MODEL_ID`, o classificador, o resumidor e o copiloto passam a usar o Claude no Amazon Bedrock, que é o que o deploy da AWS faz ([infra/README.md](infra/README.md)); com `BEDROCK_EMBEDDING_MODEL_ID`, os embeddings passam ao Titan (ainda desligado no deploy).

As normas-base vêm do Planalto (Lei 14.300, Lei 15.097 — lista em `ai/functions/canonicas.py`) e dos PDFs de resoluções da ANEEL colocados em `ai/data/canonicas/` (o site da ANEEL bloqueia download automático). Na API, `AI_SERVICE_URL` aponta para o serviço; sem ele, o chat cai numa busca por palavra-chave.

### Variáveis de ambiente principais

| Variável                            | Para quê                                                                                           |
| ----------------------------------- | -------------------------------------------------------------------------------------------------- |
| `DB_USER`, `DB_PASSWORD`, `DB_NAME` | Postgres                                                                                           |
| `JWT_SECRET`                        | login                                                                                              |
| `INTERNAL_API_KEY`                  | chave que o serviço Python manda em `x-internal-key` para gravar normas (`POST /interno/ingestao`) |
| `AI_SERVICE_URL`                    | opcional: URL do serviço de RAG do copiloto; sem ela, o copiloto busca por palavra-chave           |
| `SMTP_*`                            | opcional: sem SMTP, os e-mails de alerta e de senha só aparecem no log                             |
| `WEB_URL`                           | link do front usado nos e-mails                                                                    |

## Pré-requisitos

- Docker e Docker Compose, **ou** Node 24, Python 3.10+ e PostgreSQL 16
- Uma chave da API da NVIDIA **ou** acesso ao Amazon Bedrock para os modelos do pipeline em `ai/`

## Licença

Este projeto está sob a licença MIT — veja o arquivo [LICENSE](./LICENSE) para mais detalhes.
