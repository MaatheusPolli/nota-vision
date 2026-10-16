# NotaVision

Extração estruturada de cupons fiscais, NF-e e recibos a partir de foto ou PDF. Devolve JSON validado por schema, com **confiança por campo** e **revisão humana assistida**.

> O LLM extraiu tudo certo — menos o total.
> Por isso a aritmética é conferida em código, e não pelo modelo.

## Como funciona

```
upload → pré-processamento (sharp/pdfjs) → extração (Claude visão + tool use)
       → schema (Zod) → regras de consistência (funções puras) → confiança por campo
       → pronto  |  fila de revisão → UI lado a lado → correção vira dado de avaliação
```

- **Schema único.** O mesmo Zod gera a tool definition enviada ao Claude e valida a resposta. Se a resposta violar o schema, os erros voltam ao modelo como `tool_result` e ele tenta de novo. Nunca sai "quase JSON".
- **O LLM não tem a última palavra.** Regras determinísticas conferem Σ itens = subtotal, subtotal − desconto = total, qtd × unitário = total do item, DV do CNPJ (inclusive o alfanumérico), data plausível e DV/AAMM/CNPJ da chave de acesso.
- **Confiança por campo.** `high` = uma regra conferiu. `medium` = nenhuma regra verifica o campo. `low` = falhou numa regra ou o modelo declarou incerteza. Só vai para a fila de revisão o documento que tem algum campo `low`.
- **Correção é telemetria.** Cada campo editado na revisão grava o valor do modelo e o valor correto (`/metricas` → `correcoesPorCampo`).
- **Falha parcial não perde trabalho.** Se a extração falhar, as páginas processadas e o custo já gasto ficam registrados (`status: partial`).

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
      "message": "subtotal − desconto = 94.03, mas total = 99.03" }
  ],
  "confianca": { "total": "low", "subtotal": "high", "emitente.cnpj": "high", "chaveAcesso": "high",
                 "itens[1].valorTotal": "high", "emitente.nome": "medium", "...": "..." },
  "custoUsd": 0.0189
}
```

Na UI de revisão, a imagem fica à esquerda e o formulário à direita. O campo `total` aparece em vermelho e a conferência (Σ itens, subtotal − desconto) é recalculada enquanto o revisor digita. `Ctrl+Enter` confirma e abre o próximo documento da fila.

<!-- TODO: GIF da revisão (docs/revisao.gif) -->

## Avaliação

`pnpm eval` passa o dataset sintético (30 cupons e recibos renderizados, alguns rotacionados ou com ruído) pelo pipeline completo e escreve [`report/eval.md`](report/eval.md). A CI falha se a acurácia ficar abaixo de 90% ou se mais de 10% dos documentos saírem como `pronto` com algum erro.

> ⚠️ Os números abaixo usam **respostas simuladas** do modelo (gabarito + erros típicos injetados com seed fixa). Eles validam o pipeline e as métricas, **não** a acurácia do Claude. Para números reais, rode `pnpm eval --real` (tem custo; grava as respostas) e depois `pnpm eval --gravadas`.

| Métrica | Simulado |
|---|---|
| Acurácia por campo | 97,9% |
| Taxa de revisão | 26,7% |
| Erros que foram para revisão | 94,4% |
| Docs `pronto` com erro silencioso | 1 de 30 |

O erro que passou foi numa descrição de item: nenhuma regra verifica texto livre, e é por isso que esses campos nunca passam de `medium`.

## Rodando

Pré-requisitos: Node 20.19+ e pnpm 9.

```bash
pnpm install
```

**Demo sem chave de API** (LLM falso + cupons sintéticos, tudo em memória):

```bash
pnpm demo
```

```bash
pnpm dev:web
```

Abra http://localhost:5173.

**Com o Claude de verdade e persistência:**

```bash
docker compose up -d
```

```bash
cp .env.example .env
```

Preencha `ANTHROPIC_API_KEY` no `.env` (a API lê o arquivo ao subir) e rode `pnpm dev:api` e `pnpm dev:web`. Sem `DATABASE_URL`/`S3_ENDPOINT`, a API usa armazenamento em memória. O modelo padrão é `claude-opus-5`; troque com `ANTHROPIC_MODEL`.

**Testes:**

```bash
pnpm test:coverage
```

A CI nunca chama o LLM: o pipeline roda com respostas gravadas (`tests/fixtures/llm/`).

## API

| Método | Rota | |
|---|---|---|
| `POST` | `/documentos?aguardar=true&webhook=URL` | upload multipart (`arquivo`); JPG, PNG, HEIC, PDF até 10 MB. `202` assíncrono ou `201` com `aguardar=true` |
| `GET` | `/documentos?status=` | listagem |
| `GET` | `/documentos/:id` | documento, issues, confiança, custo |
| `GET` | `/documentos/:id/paginas/:n` | página pré-processada (JPEG) |
| `GET` | `/documentos/:id/export.json` · `.csv` | exportação |
| `GET` | `/export.csv?status=pronto` | exportação em lote |
| `GET` | `/revisao` | fila de revisão |
| `PUT` | `/revisao/:id` | `{ documento }` confirma a revisão e registra as correções |
| `GET` | `/metricas` | custo total/médio, taxa de revisão, correções por campo |

O webhook recebe `documento.processado` e `documento.revisado` (3 tentativas com backoff). Arquivos e registros são apagados depois de `RETENCAO_DIAS` (padrão 30); o bucket MinIO também tem regra de expiração.

## Estrutura

```
packages/core   schema Zod, pré-processamento, extração (porta LlmVisionClient), regras, confiança, pipeline, export
apps/api        Fastify, repositório Postgres/memória, storage S3/memória, webhook, retenção, modo demo
apps/web        React + Vite + Tailwind: lista, fila, upload, revisão lado a lado
eval            gerador de dataset sintético, métricas, runner
tests           unit (regras com 100% de cobertura), integração (pipeline e API com LLM gravado)
```

## Stack

TypeScript · Claude (visão + tool use, `@anthropic-ai/sdk`) · Zod · sharp · pdfjs-dist · Fastify · PostgreSQL · S3/MinIO · React · Vite · Tailwind · Vitest

## Status

Etapas 1 a 6 concluídas (schema, pré-processamento, extração, pipeline/API e UI de revisão); 12 arquivos de teste passando.

Pendências conhecidas:

- etapa 7: dataset de avaliação e `report/eval.md` com acurácia por campo
- GIF da revisão no README

## Licença

Este projeto usa **licença dupla**. O texto completo está em [LICENSE](LICENSE).

- **Uso pessoal, educacional e não comercial:** livre, nos termos da licença MIT (mantenha o aviso de copyright).
- **Uso comercial:** exige licença comercial. Isso inclui usar em produto ou serviço comercial, vender acesso ou derivados, usar em atividades com fins lucrativos e implantar em empresas.

Contato para licença comercial: ma.polli@hotmail.com
