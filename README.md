# openaudit-core-contracts

**Source of truth** (fonte única de verdade) para os contratos do ecossistema OpenAudit Brasil.

Este repositório define, valida e versiona:

- **OpenAPI** dos serviços (ex.: `openaudit-search-api`)
- **JSON Schemas** (mensagens, artefatos, auditoria)
- **Contratos de observabilidade** (logs estruturados e convenções de telemetria)
- **Exemplos canônicos** validados por CI
- **Geração de tipos/modelos** para **TypeScript (Node)** e **Python (CLI/pipelines)**

> Se um serviço/UI não segue estes contratos, ele está fora do padrão do projeto.  
> "Plug-and-play" só existe quando os contratos são executáveis e testados.

---

## Índice
- [Objetivo](#objetivo)
- [O que este repositório é (e não é)](#o-que-este-repositório-é-e-não-é)
- [Melhor stack (tooling)](#melhor-stack-tooling)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Contratos incluídos no MVP (Fase 1)](#contratos-incluídos-no-mvp-fase-1)
- [Exemplos de contrato](#exemplos-de-contrato)
  - [OpenAPI: `GET /search`](#openapi-get-search)
  - [JSON Schema: `SearchResponse`](#json-schema-searchresponse)
  - [JSON Schema: `catalog_meta.json`](#json-schema-catalog_metajson)
  - [JSON Schema: `LogEvent` (observabilidade)](#json-schema-logevent-observabilidade)
- [Como executar](#como-executar)
- [Geração de tipos (TS e Python)](#geração-de-tipos-ts-e-python)
- [Cenários de testes](#cenários-de-testes)
- [Diretrizes de desenvolvimento](#diretrizes-de-desenvolvimento)
  - [Como adicionar/alterar um contrato](#como-adicionaralterar-um-contrato)
  - [Política de versionamento (SemVer)](#política-de-versionamento-semver)
  - [Compatibilidade e breaking changes](#compatibilidade-e-breaking-changes)
- [Como consumir nos outros repositórios](#como-consumir-nos-outros-repositórios)
- [Links oficiais](#links-oficiais)
- [Licença](#licença)

---

## Objetivo

Garantir que o ecossistema seja:

- **Modular**: serviços e ferramentas se encaixam sem acoplamento implícito
- **Auditável**: contratos forçam presença de metadados essenciais (versões, hashes, proveniência)
- **Seguro**: contratos impedem/evitam PII por padrão e linguagem acusatória em campos críticos
- **Evolutivo**: versionamento formal (SemVer) com exemplos e CI impedindo divergência

---

## O que este repositório é (e não é)

### ✅ É
- Contrato executável (OpenAPI + JSON Schema)
- Biblioteca de modelos gerados (TS/Python) para evitar "JSON de mão"
- Catálogo de convenções (observabilidade: logs/telemetria) com validação
- Base para "plug-ins" futuros (connectors/transformers/IA) via contratos bem definidos

### ❌ Não é
- Serviço HTTP
- Implementação de regras/indicadores
- Camada de persistência
- Observabilidade "rodando" (isso é o repo `openaudit-observability`, que implementa libs/middlewares)

---

## Melhor stack (tooling)

Como `core-contracts` é essencialmente **validação + geração**, a stack ideal é tooling em Node:

- **Node.js 22+**
- **pnpm** (ou npm/yarn)
- **TypeScript** (para scripts/geração)
- **AJV** para validar JSON Schema (runtime e testes)
- **OpenAPI linter** (ex.: `@redocly/cli`) para validar OpenAPI
- **openapi-typescript** para gerar tipos TS a partir do OpenAPI
- **datamodel-code-generator** (Python) para gerar modelos Pydantic a partir de JSON Schema/OpenAPI

> Observação: os repositórios consumidores podem ser Node, Python, Rust etc.  
> O "source of truth" aqui é OpenAPI/JSON Schema. As linguagens consomem via geração.

---

## Estrutura do repositório

Estrutura:

```txt
.
├─ openapi/
│  └─ search-api.yaml
├─ schemas/
│  ├─ common/
│  │  ├─ confidence.enum.json
│  │  ├─ severity.enum.json
│  │  ├─ stage.enum.json
│  │  └─ ids.schema.json
│  ├─ search/
│  │  ├─ candidate.schema.json
│  │  └─ search_response.schema.json
│  ├─ catalog/
│  │  └─ catalog_meta.schema.json
│  └─ observability/
│     ├─ log_event.schema.json
│     └─ error_event.schema.json
├─ observability/
│  ├─ metrics_registry.yaml
│  └─ span_names.yaml
├─ examples/
│  ├─ search/
│  │  └─ search_response.json
│  ├─ catalog/
│  │  └─ catalog_meta.json
│  └─ logs/
│     └─ search_log_event.json
├─ packages/
│  ├─ typescript/            # gerado (não editar à mão)
│  └─ python/                # gerado (não editar à mão)
├─ scripts/
│  ├─ validate_schemas.ts
│  ├─ validate_openapi.ts
│  ├─ validate_examples.ts
│  └─ generate.ts
├─ CHANGELOG.md
├─ package.json
└─ README.md
````

---

## Contratos incluídos no MVP (Fase 1)

### Search

* OpenAPI: `GET /health`, `GET /search`
* JSON Schema: `SearchResponse`, `Candidate`, enums comuns

### Entity Catalog

* JSON Schema do `catalog_meta.json` (proveniência e versão do catálogo)

### Observabilidade (contratos)

* JSON Schema: `LogEvent` e `ErrorEvent`
* Registry: nomes de métricas e spans (catálogo de convenções)

---

## Exemplos de contrato

### OpenAPI: `GET /search`

Trecho ilustrativo:

```yaml
openapi: 3.0.3
info:
  title: openaudit-search-api
  version: 0.1.0
paths:
  /search:
    get:
      summary: Search candidates in entity catalog
      parameters:
        - in: query
          name: q
          required: true
          schema: { type: string, minLength: 1 }
        - in: query
          name: limit
          schema: { type: integer, minimum: 1, maximum: 25, default: 10 }
        - in: query
          name: offset
          schema: { type: integer, minimum: 0, default: 0 }
      responses:
        "200":
          description: OK
          headers:
            X-Request-Id:
              schema: { type: string }
            X-Catalog-Version:
              schema: { type: string }
            X-Cache:
              schema: { type: string, enum: [HIT, MISS] }
          content:
            application/json:
              schema:
                $ref: "../schemas/search/search_response.schema.json"
```

---

### JSON Schema: `SearchResponse`

Trecho ilustrativo (resumo):

```json
{
  "$id": "schemas/search/search_response.schema.json",
  "type": "object",
  "required": ["query", "catalog", "candidates"],
  "properties": {
    "query": {
      "type": "object",
      "required": ["normalized", "hash"],
      "properties": {
        "normalized": { "type": "string", "minLength": 1 },
        "hash": { "type": "string", "pattern": "^sha256:" }
      }
    },
    "catalog": {
      "type": "object",
      "required": ["name", "version", "snapshot_id", "source"],
      "properties": {
        "name": { "type": "string" },
        "version": { "type": "string" },
        "snapshot_id": { "type": "string" },
        "source": { "type": "string" }
      }
    },
    "candidates": {
      "type": "array",
      "items": { "$ref": "./candidate.schema.json" }
    }
  }
}
```

---

### JSON Schema: `catalog_meta.json`

Esse arquivo é produzido pelo `openaudit-entity-catalog` (Python CLI) e consumido pelo `search-api`.

Exemplo:

```json
{
  "name": "openaudit-entity-catalog",
  "catalog_version": "2026-02-28",
  "snapshot_id": "cat:2026-02-28T03:00:00Z",
  "schema_version": "1.0.0",
  "build_version": "0.1.0",
  "collected_at": "2026-02-28T03:00:00Z",
  "sources": [
    {
      "name": "TSE",
      "dataset": "candidaturas",
      "source_url": "https://dadosabertos.tse.jus.br/",
      "collected_at": "2026-02-28T02:40:00Z",
      "hash": "sha256:..."
    }
  ],
  "artifacts": [
    { "file": "catalog.sqlite", "hash": "sha256:...", "rows": 123456 }
  ],
  "qa": {
    "report_file": "qa_report.json",
    "passed": true,
    "checks_failed": 0
  }
}
```

---

### JSON Schema: `LogEvent` (observabilidade)

**Por que aqui?** Porque observabilidade também é contrato. Se cada serviço loga de um jeito, você perde rastreabilidade.

Regras essenciais:

* **não** logar `q` bruto
* **não** logar CPF/PII por padrão
* campos para correlação (`request_id`, `run_id`) obrigatórios conforme contexto

Exemplo de evento:

```json
{
  "timestamp": "2026-02-28T20:10:00Z",
  "level": "info",
  "service": { "name": "openaudit-search-api", "version": "0.1.0" },
  "env": "dev",
  "event": { "name": "catalog.search" },
  "correlation": { "request_id": "req_123", "run_id": null },
  "data": {
    "query_hash": "sha256:...",
    "catalog_version": "2026-02-28",
    "limit": 10,
    "offset": 0,
    "result_count": 8,
    "cache_hit": false
  },
  "duration_ms": 21,
  "status": "success"
}
```

> Implementação para emitir isso fica no repo `openaudit-observability`.
> Aqui, a gente só define e valida o contrato.

---

## Como executar

### Pré-requisitos

* Node.js 22+
* pnpm (recomendado) ou npm

### Instalação

```bash
pnpm install
```

### Validar tudo

```bash
pnpm validate
```

### Gerar modelos/tipos

```bash
pnpm generate
```

### Rodar testes

```bash
pnpm test
```

> Os comandos acima são o "contrato operacional" do repo.

---

## Geração de tipos (TS e Python)

### TypeScript (Node)

Gera tipos TS para consumo nos serviços Node/UI:

* saída: `packages/typescript/`

Exemplos de geradores comuns:

* `openapi-typescript` (OpenAPI → TS)
* `json-schema-to-typescript` (JSON Schema → TS)

### Python (CLI/pipelines)

Gera modelos Pydantic para consumo em CLIs/pipelines:

* saída: `packages/python/`

Gerador recomendado:

* `datamodel-code-generator` (JSON Schema/OpenAPI → Pydantic)

> Regra: `packages/*` é gerado. Não editar manualmente.

---

## Cenários de testes

O CI deve cobrir, no mínimo:

1. **Schema lint/parse**

   * todos os JSON Schemas devem ser válidos
2. **OpenAPI lint/parse**

   * OpenAPI válido e consistente
3. **Exemplos validados**

   * todo JSON em `examples/` deve validar contra seu schema correspondente
4. **Geração reprodutível**

   * `pnpm generate` não pode gerar diffs não commitados (enforced)
5. **Breaking change guard (recomendado)**

   * mudanças incompatíveis exigem bump de versão MAJOR e changelog

---

## Diretrizes de desenvolvimento

### Como adicionar/alterar um contrato

Checklist obrigatório:

1. Atualize o arquivo em `openapi/` ou `schemas/`
2. Atualize/adicione exemplo em `examples/`
3. Rode:

   * `pnpm validate`
   * `pnpm test`
   * `pnpm generate`
4. Atualize `CHANGELOG.md`
5. Se for breaking change, atualize versão (SemVer)

### Política de versionamento (SemVer)

* **MAJOR**: breaking changes (remove/muda tipo/muda semântica/torna obrigatório)
* **MINOR**: adiciona campo opcional, novo endpoint compatível
* **PATCH**: correção de docs/descrições/examples sem alterar contrato

### Compatibilidade e breaking changes

Consideramos breaking change:

* remover campo
* mudar tipo
* mudar enum permitido
* mudar significado semântico (mesmo mantendo tipo)
* mudar contrato de observabilidade (campos obrigatórios), sem bump major

---

## Como consumir nos outros repositórios

### Node (search-api, ui)

* Dependência: `@openaudit/core-contracts` (quando publicado) ou via git/tag
* Use os tipos gerados + validação runtime opcional (AJV)

### Python (entity-catalog CLI)

* Dependência: `openaudit_core_contracts` (quando publicado) ou via git/tag
* Use os modelos Pydantic para validar `catalog_meta.json` e outros artefatos

> Mesmo antes de publicar em registries, vocês podem versionar por Git tags e usar `git+https`.

---

## Links oficiais

Documentação normativa do projeto (fonte de verdade):

* [https://github.com/OpenAudit-Brasil/About](https://github.com/OpenAudit-Brasil/About)

Roadmap:

* [https://github.com/OpenAudit-Brasil/About/blob/main/ROADMAP.md](https://github.com/OpenAudit-Brasil/About/blob/main/ROADMAP.md)

---

## Licença

* Licença oficial: [https://github.com/OpenAudit-Brasil/About/blob/main/LICENSE](https://github.com/OpenAudit-Brasil/About/blob/main/LICENSE)
