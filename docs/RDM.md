---
pdf_options:
  format: A4
  margin: 9mm 11mm
  printBackground: true
css: |-
  body { font-family: "Segoe UI", Arial, sans-serif; font-size: 8.3pt; line-height: 1.25; color: #222; }
  h1 { font-size: 14pt; margin: 0 0 2px; border-bottom: 2px solid #c2381b; padding-bottom: 3px; }
  h2 { font-size: 9.5pt; margin: 6px 0 2px; color: #c2381b; }
  table { border-collapse: collapse; width: 100%; margin: 0 !important; }
  th, td { border: 1px solid #bbb; padding: 2px 5px !important; vertical-align: top; text-align: left; }
  th { background: #f3ece0; }
  code { font-family: Consolas, monospace; font-size: 7.6pt !important; background: #f5f5f5; padding: 0 2px !important; }
  p, ul { margin: 2px 0; } ul { padding-left: 16px; }
---

# Ficha Simplificada de Registro de Mudança (RDM) — EstuLanches

| Campo | Registro |
|---|---|
| **Sistema** | EstuLanches — cardápio online e fila de pedidos da cantina escolar |
| **Responsável** | Henrique Sasaki — Técnico de Implantação |
| **Tipo de mudança** | Normal — implantação inicial (nova aplicação em produção via Docker Compose) |
| **Versão implantada** | v1.0.0 · imagem `estulanches-app:1.0.0` (ID `sha256:2c79e2270b9a`) · commit `3f82318` |
| **Início / fim** | 22/11/2026 · 12:53 – 12:56 (horário de Brasília) |
| **Ambiente** | Windows 11 Pro + Docker Desktop (WSL2, linux/amd64) · Docker Engine 29.9.3 · Docker Compose v5.7.0 · Node 22.23.5 · PostgreSQL 16.16 |
| **Serviços** | `app` (Fastify + front React, porta 3000) e `db` (postgres:16-alpine, sem porta publicada, volume `pgdata`) |

## Parâmetros configurados

| Variável | Valor | Variável | Valor |
|---|---|---|---|
| `NODE_ENV` | production | `JWT_EXPIRES_IN` | 8h |
| `APP_PORT` | 3000 | `CORS_ORIGIN` | http://localhost:3000 |
| `POSTGRES_DB` / `POSTGRES_USER` | estulanches | `RATE_LIMIT_LOGIN` | 5 tentativas/min por IP |
| `POSTGRES_PASSWORD`, `DATABASE_URL` | definido via .env | `SEED` | true |
| `JWT_SECRET` | definido via .env | `SEED_ADMIN_PASSWORD`, `SEED_ALUNO_PASSWORD` | definido via .env |

Limites: `app` 512 MB / 1 CPU · `db` 512 MB · `restart: unless-stopped` · contêiner da app roda como usuário `node` (não root).

## Testes de validação (evidências em `docs/evidencias/`)

| # | Comando | Esperado | Obtido |
|---|---|---|---|
| 1 | `docker compose build` | imagem gerada sem erro | OK — `estulanches-app:1.0.0` construída |
| 2 | `docker compose up -d` + `ps` | app e db *healthy* | OK — ambos *healthy*; app em ~9 s |
| 3 | `docker compose logs db/app --tail=100` | migrations + seed + servidor no ar | OK — 2 migrations aplicadas, seed concluído, ouvindo em :3000 |
| 4 | `grep -i -E "error\|fatal\|panic"` nos logs | 0 ocorrências | OK — 0 no db, 0 na app |
| 5 | `curl -s localhost:3000/api/health` | `{"status":"ok","db":"up"}` | OK — `{"status":"ok","db":"up"}` |
| 6 | `docker compose --profile test run --rm tests` | todos passam; cobertura ≥ 70% | OK — 64/64 testes (28 unit., 36 integr.); cobertura 97,19% stmts · 93,33% branches · 100% funções |
| 7 | curl: login aluno → pedido → login admin → avançar | 201, total do servidor, 200 | OK — pedido #1 total R$ 18,00 (preço falso ignorado), estoque 40→38, RECEBIDO→EM_PREPARO→PRONTO; inválida = 422 |
| 8 | `docker compose down` (sem `-v`) + `up -d` | pedido continua no banco | OK — pedido #1 (PRONTO, R$ 18,00) preservado; seed não duplicou dados |
| 9 | `docker image ls estulanches-app` | tag 1.0.0 presente | OK — 1.0.0, 159 MB (conteúdo) / 661 MB em disco |

## Plano de rollback

1. **Antes de cada nova versão**, backup do banco: `docker compose exec -T db sh -c 'pg_dump -U $POSTGRES_USER $POSTGRES_DB' > backup_AAAAMMDD.sql`.
2. **Voltar a imagem:** em `docker-compose.yml`, trocar `image:` para a tag anterior (ex.: `estulanches-app:1.0.0`) e executar `docker compose up -d app` — sem rebuild, a tag antiga continua na máquina.
3. **Se houve migration incompatível:** `docker compose down`, recriar o banco e restaurar: `docker compose exec -T db sh -c 'psql -U $POSTGRES_USER $POSTGRES_DB' < backup_AAAAMMDD.sql`.
4. **Validar:** `docker compose ps` (*healthy*) e `curl localhost:3000/api/health`. Por ser a implantação inicial, o rollback desta v1.0.0 é `docker compose down`.

## Status final

**Concluída com sucesso** — todos os 9 testes de validação aprovados, sem erros nos logs; sistema disponível em http://localhost:3000.
