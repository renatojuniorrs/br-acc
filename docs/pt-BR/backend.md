# Backend — Como Funciona

Idioma: [English](../backend.md) | **Português (Brasil)**

O backend do BR/ACC é uma aplicação [FastAPI](https://fastapi.tiangolo.com/) escrita em Python 3.12+.  
Ele expõe uma API REST com [Neo4j 5](https://neo4j.com/) como banco de dados em grafo e roda como um serviço ASGI assíncrono.

## Estrutura de Diretórios

```
api/
├── src/bracc/
│   ├── main.py          # Fábrica da aplicação e configuração de middlewares
│   ├── config.py        # Configurações carregadas de variáveis de ambiente
│   ├── constants.py     # Constantes de domínio compartilhadas (ex.: cargos PEP)
│   ├── dependencies.py  # Injeção de dependências FastAPI (driver DB, auth)
│   ├── middleware/      # Cabeçalhos de segurança, rate limiting, mascaramento de CPF
│   ├── models/          # Schemas Pydantic de requisição/resposta
│   ├── queries/         # Arquivos de consulta Cypher nomeados (.cypher)
│   ├── routers/         # Handlers de rota agrupados por domínio
│   ├── services/        # Lógica de negócio e helpers do Neo4j
│   ├── i18n/            # Strings de internacionalização
│   └── templates/       # Templates Jinja2 (exportação PDF)
├── tests/               # Suite de testes Pytest
├── pyproject.toml       # Dependências e configuração de ferramentas
└── Dockerfile           # Imagem de container para produção
```

## Inicialização da Aplicação

O `main.py` cria a aplicação FastAPI e integra todos os componentes:

1. **Hook de lifespan** — na inicialização, um driver async do Neo4j é criado via `init_driver()` e armazenado em `app.state`.  
   `ensure_schema()` executa o `schema_init.cypher` para que todos os índices e constraints existam antes da primeira requisição.
2. **Stack de middlewares** (mais externo → mais interno):
   | Middleware | Função |
   |---|---|
   | `SlowAPIMiddleware` | Aplica rate limits por cliente |
   | `CORSMiddleware` | Define origens permitidas a partir da variável `CORS_ORIGINS` |
   | `SecurityHeadersMiddleware` | Adiciona `X-Frame-Options`, `CSP`, `HSTS` (prod), etc. |
   | `CPFMaskingMiddleware` | Mascara CPFs brasileiros nas respostas JSON |
3. **Routers** são registrados para cada domínio (veja [Routers](#routers) abaixo).

## Configuração

Todas as configurações são carregadas pelo `config.py` usando [pydantic-settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/).  
As variáveis de ambiente mapeiam 1-para-1 com os nomes das configurações (sem prefixo).  
Configurações principais:

| Variável | Padrão | Descrição |
|---|---|---|
| `NEO4J_URI` | `bolt://localhost:7687` | URI de conexão com o Neo4j |
| `NEO4J_USER` | `neo4j` | Usuário do banco |
| `NEO4J_PASSWORD` | `changeme` | Senha do banco — **deve ser alterada** |
| `JWT_SECRET_KEY` | `change-me-in-production` | Segredo HMAC para tokens JWT — **deve ter ≥ 32 chars** |
| `JWT_EXPIRE_MINUTES` | `1440` | Tempo de vida do token (24 h) |
| `RATE_LIMIT_ANON` | `60/minute` | Rate limit padrão para requisições não autenticadas |
| `RATE_LIMIT_AUTH` | `300/minute` | Rate limit para requisições autenticadas |
| `PRODUCT_TIER` | `community` | Nível de funcionalidades |
| `PUBLIC_MODE` | `false` | Habilita restrições de API no modo público |
| `PUBLIC_ALLOW_PERSON` | `false` | Permite consultas a entidades pessoais no modo público |
| `PUBLIC_ALLOW_ENTITY_LOOKUP` | `false` | Permite endpoints de lookup de entidade no modo público |
| `PUBLIC_ALLOW_INVESTIGATIONS` | `false` | Permite endpoints de investigação no modo público |
| `PATTERNS_ENABLED` | `false` | Habilita o engine de patterns |
| `CORS_ORIGINS` | `http://localhost:3000` | Origens CORS permitidas (separadas por vírgula) |

## Acesso ao Banco de Dados

Toda interação com o banco passa por `services/neo4j_service.py`:

- **`CypherLoader`** lê arquivos `.cypher` de `queries/` e mantém o texto em cache em memória.
- **`execute_query(session, query_name, params)`** executa uma consulta nomeada e retorna todos os registros.
- **`execute_query_single(session, query_name, params)`** retorna um único registro (ou `None`).
- **`sanitize_props(props)`** converte tipos temporais, listas e dicts do Neo4j em escalares seguros para JSON.
- **`ensure_schema(driver)`** executa `schema_init.cypher` na inicialização para criar constraints e índices de forma idempotente.

Nenhuma string Cypher é embutida diretamente no código Python; cada consulta fica em um arquivo `.cypher` dedicado.

## Routers

| Módulo do router | Prefixo | Auth obrigatória | Descrição |
|---|---|---|---|
| `meta.py` | `/api/v1/meta` | Não | Health check do banco, estatísticas agregadas, registro de fontes |
| `public.py` | `/api/v1/public` | Não | Subgrafo público de empresa, patterns (503 quando desabilitado), meta |
| `auth.py` | `/api/v1/auth` | Não (registro/login) | Cadastro de usuário (com código de convite), login, `/me` |
| `entity.py` | `/api/v1/entity` | Sim | Lookup de entidade por CPF/CNPJ, timeline, conexões, exposure |
| `search.py` | `/api/v1` | Não | Busca textual de entidades (`/search?q=…`) |
| `graph.py` | `/api/v1/graph` | Sim | Expansão de ego-grafo com filtragem de labels via APOC |
| `patterns.py` | `/api/v1/patterns` | Sim | Endpoints do engine de patterns (desabilitado por padrão) |
| `baseline.py` | `/api/v1/baseline` | Sim | Estatísticas de baseline regional e setorial |
| `investigation.py` | `/api/v1/investigations` | Sim | CRUD de investigações, anotações, tags; exportação PDF/JSON; links de compartilhamento |

Um endpoint não autenticado `GET /health` existe diretamente na aplicação raiz para health checks de containers.

## Autenticação

O serviço `auth` (`services/auth_service.py`) implementa autenticação por e-mail e senha:

1. **Cadastro** — senhas são hasheadas com `bcrypt`. Um código de convite opcional é validado contra a variável `INVITE_CODE`. O nó de usuário é armazenado no Neo4j via `user_create.cypher`.
2. **Login** — as credenciais são verificadas com `bcrypt.checkpw`. Um JWT assinado (`HS256`) é retornado.
3. **Validação do token** — `decode_access_token()` verifica a assinatura e a expiração do JWT. `get_current_user()` (injetado via dependência `CurrentUser`) carrega o usuário do Neo4j em cada requisição autenticada.
4. **Rate limiting** — login e cadastro são limitados a 10 requisições/minuto por IP, independentemente dos limites globais.

## Detalhes dos Middlewares

### Mascaramento de CPF (`middleware/cpf_masking.py`)

Após a geração da resposta, o middleware lê o corpo JSON completo e substitui os padrões de CPF:

- CPF formatado (`123.456.789-00`) → `***.***.789-00`
- CPF cru com 11 dígitos (`12345678900`) → `*******8900`

CPFs de **Pessoas Expostas Politicamente (PEPs)** são mantidos visíveis.  
O middleware percorre a árvore JSON para coletar CPFs de PEPs (identificados por `is_pep: true` ou palavras-chave políticas nos campos `role`/`cargo`) antes de aplicar as máscaras.

### Rate Limiting (`middleware/rate_limit.py`)

É usado o [SlowAPI](https://slowapi.readthedocs.io/) (wrapper Starlette para controle de taxa).  
A função de chave prefere o ID do usuário autenticado (extraído do cabeçalho `Authorization: Bearer …`) em vez do IP do cliente, de modo que usuários autenticados têm uma cota maior.

### Cabeçalhos de Segurança (`middleware/security_headers.py`)

Toda resposta HTTP recebe:

- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Referrer-Policy: no-referrer`
- `Permissions-Policy` — desabilita APIs do navegador (câmera, geolocalização, etc.)
- `Content-Security-Policy` — `default-src 'none'` restrito para rotas de API
- `Strict-Transport-Security` (apenas produção + HTTPS)

## Modo Público e Controle de Acesso

Quando `PUBLIC_MODE=true`, o `services/public_guard.py` aplica controles de acesso em camadas:

| Função guard | O que ela garante |
|---|---|
| `enforce_entity_lookup_enabled()` | Bloqueia `/entity/*` a menos que `PUBLIC_ALLOW_ENTITY_LOOKUP=true` |
| `enforce_entity_lookup_policy(identifier)` | Bloqueia lookup por CPF a menos que `PUBLIC_ALLOW_PERSON=true` |
| `enforce_person_access_policy(labels)` | Retorna 403 se um nó tem labels Person/Partner e o acesso a pessoas está desabilitado |
| `ensure_investigations_enabled()` | Bloqueia `/investigations/*` a menos que `PUBLIC_ALLOW_INVESTIGATIONS=true` |
| `sanitize_public_properties(props)` | Remove CPF e campos sensíveis das propriedades de nó no modo público |
| `infer_exposure_tier(labels)` | Retorna `public_safe`, `restricted` ou `internal_only` |

## Rodando Localmente

```bash
cd api
uv sync --dev

# inicie o Neo4j primeiro (veja docker-compose em infra/)
uvicorn bracc.main:app --reload
```

- API: `http://localhost:8000`
- Docs interativos: `http://localhost:8000/docs`

## Testes

```bash
cd api
uv run pytest
```

A suite usa `pytest-asyncio`. Testes marcados como `integration` exigem uma instância do Neo4j em execução e são excluídos por padrão (`-m 'not integration'`).
