# InvestFacil — Pipeline de Dados B3 → App Web

## Visão Geral

O InvestFacil é um pipeline de dados que coleta cotações e fundamentos de ações da B3 (bolsa brasileira) diariamente, processa os dados em 3 camadas (Bronze, Silver, Gold) e exporta arquivos JSON para alimentar um aplicativo web público hospedado no Netlify.

**Qualquer pessoa pode acessar o app pelo link público — sem login, sem Databricks.**

---

## Arquitetura do Pipeline

```
Yahoo Finance API
       ↓
  ┌─────────────────────────────────────────────┐
  │           Databricks Job (Diário 19h)        │
  │                                              │
  │  01_Bronze_Ingestao_B3                       │
  │  → Coleta cotações e fundamentos (JSON bruto)│
  │  → Salva em: investfacil_catalog.bronze      │
  │       ↓                                      │
  │  02_Silver_Transformacao                     │
  │  → Parse do JSON → colunas estruturadas      │
  │  → Mapeia nomes para português (70+ empresas)│
  │  → Traduz setores (Energy → Petróleo e Gás)  │
  │  → Remove duplicatas                         │
  │  → Salva em: investfacil_catalog.silver     │
  │       ↓                                      │
  │  03_Gold_Indicadores                         │
  │  → Junta cotações + fundamentos              │
  │  → Calcula: Dividend Yield, P/L, ROE,       │
  │    Volatilidade, Momentum 30d, Liquidez     │
  │  → Salva em: investfacil_catalog.gold       │
  │       ↓                                      │
  │  04_Export_JSON_S3                          │
  │  → Converte dados Gold para JSON             │
  │  → Gera: indicadores.json, historico.json,  │
  │    metadata.json                             │
  │  → Exporta para esta pasta no GitHub         │
  └─────────────────────────────────────────────┘
       ↓
  GitHub (InvestFacilWeb/Import-Json-InvestFacil/)
       ↓
  Netlify (deploy automático)
       ↓
  Link público → qualquer pessoa acessa
```

---

## Passo a Passo — Como Funciona

### Passo 1: Ingestão (Bronze)

O notebook `01_Bronze_Ingestao_B3` conecta à API do Yahoo Finance e coleta:
- **Cotações**: preço, abertura, máxima, mínima, volume
- **Fundamentos**: market cap, P/L, dividend yield, beta, setor

Os dados chegam como JSON bruto (`raw_payload`) e são salvos em:
- `investfacil_catalog.bronze.raw_quotes` (743 registros)
- `investfacil_catalog.bronze.raw_fundamentals` (743 registros)

### Passo 2: Transformação (Silver)

O notebook `02_Silver_Transformacao` faz o parse do JSON bruto e aplica:
- **Mapeamento de nomes**: 70+ tickers traduzidos (ex: PETR4 → "Petrobras PN")
- **Tradução de setores**: "Energy" → "Petróleo e Gás", "Financial Services" → "Financeiro"
- **Limpeza**: remove duplicatas com `dropDuplicates(["ticker"])`
- **Tipagem**: converte strings para DOUBLE, LONG, TIMESTAMP

Salva em:
- `investfacil_catalog.silver.cotacoes` (4.458 registros, 17 colunas)
- `investfacil_catalog.silver.fundamentos` (19 colunas)

### Passo 3: Indicadores (Gold)

O notebook `03_Gold_Indicadores` junta cotações + fundamentos e calcula:
- **Dividend Yield** (%) — retorno em dividendos
- **P/L** (Preço/Lucro) — valuation da empresa
- **ROE** (Retorno sobre Patrimônio)
- **Volatilidade** — desvio padrão dos retornos
- **Momentum 30d** — performance últimos 30 dias
- **Liquidez** — volume médio negociado
- **Beta** — sensibilidade ao mercado

Salva em:
- `investfacil_catalog.gold.indicadores_completos` (41 registros, 27 colunas)
- `investfacil_catalog.gold.cotacoes_historico`

### Passo 4: Exportação (JSON)

O notebook `04_Export_JSON_S3` converte os dados Gold em 3 arquivos JSON:

| Arquivo | Tamanho | Conteúdo |
|---------|---------|----------|
| `indicadores.json` | ~19 KB | 41 ações com todos os indicadores |
| `historico.json` | ~2 bytes | Histórico de cotações agrupado por ticker |
| `metadata.json` | ~0.6 KB | Metadados do pipeline (data, fonte, total) |

Os arquivos são exportados para esta pasta (`Import-Json-InvestFacil/`) no repositório GitHub.

### Passo 5: Deploy no Netlify

1. Acesse [app.netlify.com](https://app.netlify.com)
2. Conecte o repositório `flrmedeiros78/InvestFacilWeb`
3. Configurações de build:
   - **Build command**: (deixar vazio — site estático)
   - **Publish directory**: `/` (raiz do repo)
4. Clique em **Deploy**
5. O Netlify gera um link público (ex: `investfacil.netlify.app`)

### Passo 6: Compartilhar o Link

O link público gerado pelo Netlify pode ser compartilhado com qualquer pessoa:
- Funciona em celular, tablet e desktop
- Sem login necessário
- Sem instalação
- Dados atualizados automaticamente a cada execução do pipeline

---

## Estrutura dos Arquivos JSON

### indicadores.json

```json
[
  {
    "ticker": "PETR4",
    "nome_curto": "Petrobras PN",
    "nome_completo": "Petróleo Brasileiro S.A. - Petrobras",
    "setor": "Petróleo e Gás",
    "preco_atual": 49.0,
    "dividend_yield": 8.85,
    "preco_lucro_pl": 4.85,
    "volatilidade": null,
    "momentum_30d": null,
    "beta": -0.213
  }
]
```

### metadata.json

```json
{
  "projeto": "InvestFacil",
  "versao": "1.0",
  "ultima_atualizacao": "2026-10-07 15:30:00 BRT",
  "fonte_dados": "brapi.dev",
  "total_acoes": 41,
  "indicadores_disponiveis": [
    "dividend_yield",
    "preco_lucro_pl",
    "retorno_patrimonio_roe",
    "volatilidade",
    "momentum_30d",
    "volume_medio"
  ]
}
```

---

## Repositórios

| Repositório | Função |
|-------------|--------|
| [flrmedeiros78/Databricks](https://github.com/flrmedeiros78/Databricks) | Pipeline de dados (notebooks, pipeline SDP, job) |
| [flrmedeiros78/InvestFacilWeb](https://github.com/flrmedeiros78/InvestFacilWeb) | App web + JSONs exportados (este repositório) |

---

## Job no Databricks

| Item | Valor |
|------|-------|
| **Nome** | InvestFacil_Pipeline_Diario |
| **Job ID** | 679490622792902 |
| **Schedule** | Diariamente às 19h (America/Sao_Paulo) |
| **Duração** | ~5 minutos |
| **Notificação** | Email em caso de falha |

---

## Catálogo Unity (Databricks)

| Camada | Tabela | Registros |
|--------|--------|-----------|
| Bronze | `investfacil_catalog.bronze.raw_quotes` | 743 |
| Bronze | `investfacil_catalog.bronze.raw_fundamentals` | 743 |
| Silver | `investfacil_catalog.silver.cotacoes` | 4.458 |
| Silver | `investfacil_catalog.silver.fundamentos` | — |
| Gold | `investfacil_catalog.gold.indicadores_completos` | 41 |
| Gold | `investfacil_catalog.gold.cotacoes_historico` | — |

---

## Tecnologias

- **Databricks** — processamento de dados (PySpark)
- **Unity Catalog** — governança e armazenamento
- **Yahoo Finance API** — fonte de dados da B3
- **GitHub** — versionamento e hospedagem dos JSONs
- **Netlify** — hospedagem gratuita do app web
- **HTML/CSS/JavaScript** — interface do app
