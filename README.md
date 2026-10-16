# NotaVision

ExtraÃ§Ã£o estruturada de cupons fiscais, NF-e e recibos a partir de foto ou PDF. Devolve JSON validado por schema, com **confianÃ§a por campo** e **revisÃ£o humana assistida**.

> O LLM extraiu tudo certo â€” menos o total.
> Por isso a aritmÃ©tica Ã© conferida em cÃ³digo, e nÃ£o pelo modelo.

## Como funciona

```
upload â†’ prÃ©-processamento (sharp/pdfjs) â†’ extraÃ§Ã£o (Claude visÃ£o + tool use)
       â†’ schema (Zod) â†’ regras de consistÃªncia (funÃ§Ãµes puras) â†’ confianÃ§a por campo
       â†’ pronto  |  fila de revisÃ£o â†’ UI lado a lado â†’ correÃ§Ã£o vira dado de avaliaÃ§Ã£o
```

- **Schema Ãºnico.** O mesmo Zod gera a tool definition enviada ao Claude e valida a resposta. Se a resposta violar o schema, os erros voltam ao modelo como `tool_result` e ele tenta de novo. Nunca sai "quase JSON".
- **O LLM nÃ£o tem a Ãºltima palavra.** Regras determinÃ­sticas conferem Î£ itens = subtotal, subtotal âˆ’ desconto = total, qtd Ã— unitÃ¡rio = total do item, DV do CNPJ (inclusive o alfanumÃ©rico), data plausÃ­vel e DV/AAMM/CNPJ da chave de acesso.
- **ConfianÃ§a por campo.** `high` = uma regra conferiu. `medium` = nenhuma regra verifica o campo. `low` = falhou numa regra ou o modelo declarou incerteza. SÃ³ vai para a fila de revisÃ£o o documento que tem algum campo `low`.
- **CorreÃ§Ã£o Ã© telemetria.** Cada campo editado na revisÃ£o grava o valor do modelo e o valor correto (`/metricas` â†’ `correcoesPorCampo`).
- **Falha parcial nÃ£o perde trabalho.** Se a extraÃ§Ã£o falhar, as pÃ¡ginas processadas e o custo jÃ¡ gasto ficam registrados (`status: partial`).

## Exemplo

Entrada: foto de um cupom (`eval/dataset/imagens/doc-002.jpg`) em que o total foi lido errado.

```jsonc
// GET /documentos/:id
{
  "status": "revisao",
  "dados": {
    "tipo": "cupom",
    "emitente": { "nome": "FARMACIA SAUDE TOTAL", "cnpj": "47.123.961/0001-57" },
    "emissao": "2024-09-26T09:10:00-03:00",
    "itens": [
      { "descricao": "CADERNO 96 FLS",   "quantidade": 3,     "valorUnitario": 14.9, "valorTotal": 44.7 },
      { "descricao": "PAO FRANCES KG",   "quantidade": 2.589, "valorUnitario": 16.9, "valorTotal": 43.75 },
      { "descricao": "DETERGENTE 500ML", "quantidade": 2,     "valorUnitario": 2.79, "valorTotal": 5.58 }
    ],
    "subtotal": 94.03, "desconto": 0, "total": 99.03,
    "formaPagamento": "CARTAO DEBITO",
    "chaveAcesso": "35240947123961000157650010000000021872504519"
  },
  "issues": [
    { "field": "total", "code": "TOTAL_DIVERGENTE", "severity": "error",
      "message": "subtotal âˆ’ desconto = 94.03, mas total = 99.03" }
  ],
  "confianca": { "total": "low", "subtotal": "high", "emitente.cnpj": "high", "chaveAcesso": "high",
                 "itens[1].valorTotal": "high", "emitente.nome": "medium", "...": "..." },
  "custoUsd": 0.0189
}
```

Na UI de revisÃ£o, a imagem fica Ã  esquerda e o formulÃ¡rio Ã  direita. O campo `total` aparece em vermelho e a conferÃªncia (Î£ itens, subtotal âˆ’ desconto) Ã© recalculada enquanto o revisor digita. `Ctrl+Enter` confirma e abre o prÃ³ximo documento da fila.

<!-- TODO: GIF da revisÃ£o (docs/revisao.gif) -->

## AvaliaÃ§Ã£o

`pnpm eval` passa o dataset sintÃ©tico (30 cupons e recibos renderizados, alguns rotacionados ou com ruÃ­do) pelo pipeline completo e escreve [`report/eval.md`](report/eval.md). A CI falha se a acurÃ¡cia ficar abaixo de 90% ou se mais de 10% dos documentos saÃ­rem como `pronto` com algum erro.

> âš ï¸ Os nÃºmeros abaixo usam **respostas simuladas** do modelo (gabarito + erros tÃ­picos injetados com seed fixa). Eles validam o pipeline e as mÃ©tricas, **nÃ£o** a acurÃ¡cia do Claude. Para nÃºmeros reais, rode `pnpm eval --real` (tem custo; grava as respostas) e depois `pnpm eval --gravadas`.

| MÃ©trica | Simulado |
|---|---|
| AcurÃ¡cia por campo | 97,9% |
| Taxa de revisÃ£o | 26,7% |
| Erros que foram para revisÃ£o | 94,4% |
| Docs `pronto` com erro silencioso | 1 de 30 |

O erro que passou foi numa descriÃ§Ã£o de item: nenhuma regra verifica texto livre, e Ã© por isso que esses campos nunca passam de `medium`.

## Rodando

PrÃ©-requisitos: Node 20.19+ e pnpm 9.

```bash
pnpm install
```

**Demo sem chave de API** (LLM falso + cupons sintÃ©ticos, tudo em memÃ³ria):

```bash
pnpm demo
```

```bash
pnpm dev:web
```

Abra http://localhost:5173.

**Com o Claude de verdade e persistÃªncia:**

```bash
docker compose up -d
```

```bash
cp .env.example .env
```

Preencha `ANTHROPIC_API_KEY` no `.env` (a API lÃª o arquivo ao subir) e rode `pnpm dev:api` e `pnpm dev:web`. Sem `DATABASE_URL`/`S3_ENDPOINT`, a API usa armazenamento em memÃ³ria. O modelo padrÃ£o Ã© `claude-opus-5`; troque com `ANTHROPIC_MODEL`.

**Testes:**

```bash
pnpm test:coverage
```

A CI nunca chama o LLM: o pipeline roda com respostas gravadas (`tests/fixtures/llm/`).

## API

| MÃ©todo | Rota | |
|---|---|---|
| `POST` | `/documentos?aguardar=true&webhook=URL` | upload multipart (`arquivo`); JPG, PNG, HEIC, PDF atÃ© 10 MB. `202` assÃ­ncrono ou `201` com `aguardar=true` |
| `GET` | `/documentos?status=` | listagem |
| `GET` | `/documentos/:id` | documento, issues, confianÃ§a, custo |
| `GET` | `/documentos/:id/paginas/:n` | pÃ¡gina prÃ©-processada (JPEG) |
| `GET` | `/documentos/:id/export.json` Â· `.csv` | exportaÃ§Ã£o |
| `GET` | `/export.csv?status=pronto` | exportaÃ§Ã£o em lote |
| `GET` | `/revisao` | fila de revisÃ£o |
| `PUT` | `/revisao/:id` | `{ documento }` confirma a revisÃ£o e registra as correÃ§Ãµes |
| `GET` | `/metricas` | custo total/mÃ©dio, taxa de revisÃ£o, correÃ§Ãµes por campo |

O webhook recebe `documento.processado` e `documento.revisado` (3 tentativas com backoff). Arquivos e registros sÃ£o apagados depois de `RETENCAO_DIAS` (padrÃ£o 30); o bucket MinIO tambÃ©m tem regra de expiraÃ§Ã£o.

## Estrutura

```
packages/core   schema Zod, prÃ©-processamento, extraÃ§Ã£o (porta LlmVisionClient), regras, confianÃ§a, pipeline, export
apps/api        Fastify, repositÃ³rio Postgres/memÃ³ria, storage S3/memÃ³ria, webhook, retenÃ§Ã£o, modo demo
apps/web        React + Vite + Tailwind: lista, fila, upload, revisÃ£o lado a lado
eval            gerador de dataset sintÃ©tico, mÃ©tricas, runner
tests           unit (regras com 100% de cobertura), integraÃ§Ã£o (pipeline e API com LLM gravado)
```

## Stack

TypeScript Â· Claude (visÃ£o + tool use, `@anthropic-ai/sdk`) Â· Zod Â· sharp Â· pdfjs-dist Â· Fastify Â· PostgreSQL Â· S3/MinIO Â· React Â· Vite Â· Tailwind Â· Vitest


## Licença

### Uso Pessoal
Este projeto é licenciado sob a **Licença MIT** para uso pessoal, educacional e não comercial.

### Uso Comercial
Se você deseja usar este projeto comercialmente, deve obter uma licença comercial do autor.

**Uso comercial inclui:**
- Incorporar em um produto comercial
- Oferecer como serviço
- Usar em atividades com fins lucrativos
- Implantação empresarial

**Para obter uma licença comercial:**
Email: ma.polli@hotmail.com
