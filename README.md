# Grupo G6 — Economia de energia (intenção simulada)

**Projeto integrador** — Pós-graduação

| | |
|---|---|
| **Disciplina** | Análise de Dados aplicada a Redes de Telecomunicações |
| **Professor** | Jonas Augusto Kunzler |
| **Grupo** | G6 |
| **Integrantes** | Alessandro Rocha · Erik Ferreira · Alexandre Barbosa |

## Tema e pergunta (Proposta)

**Tema:** G6 — Economia de energia (intenção simulada).

**Pergunta:** em que trechos do experimento a carga da célula está baixa o suficiente para, em tese, justificar uma política de economia de energia — sem desligar nada de verdade?

**Como os dados são usados:** cruzamos `RRU.PrbTotUl` (utilização de PRB no uplink) com `DRB.UEThpUl` (vazão do UE no uplink) para identificar janelas de baixa carga, e verificamos `DRB.RlcSduDelayDl` (atraso RLC no downlink) nessas janelas para confirmar que a qualidade permanece aceitável. 
O laboratório **não controla potência de RU**: qualquer política é uma **intenção simulada** (dry-run), nunca atuação física na RAN.

## Origem dos dados

- Fonte: `repo/data/code/datasets/kpm-ue-tp-sample/` (trilha offline oficial da disciplina).
- Lab de origem: `oai-cn-gnb-nonrt-nearrt`, caso UE-TP / load-anomaly.
- Experimento: `run_id = ue-tp-20260804-174422`, regenerado em 2026-08-04.
- Formato: `kpm.sqlite` (tabelas `runs` e `kpm_samples`, com `payload_json` por amostra) e espelho `kpm.jsonl`.
- Fases: `baseline` (20 amostras), `stress` (60 amostras), `recovery` (20 amostras) — total 100.
- Licença/ética: telemetria **sintética** de laboratório RFSIM (OpenAirInterface); sem dados
  pessoais; uso exclusivamente acadêmico neste módulo.

## Como reproduzir

```bash
# a partir da raiz do projeto (Modulo9/)
python3 -m venv .venv
source .venv/bin/activate
pip install pandas numpy matplotlib jupyter nbconvert ipython

cd analise-de-dados-aplicada-a-redes-de-telecomunicacoes/notebooks
jupyter nbconvert --to notebook --execute --inplace 01_etl_kpm.ipynb
jupyter nbconvert --to notebook --execute --inplace 02_eda_kpm.ipynb
```

- `01_etl_kpm.ipynb` lê `kpm.sqlite`, tipa e faz QC das amostras, calcula a coluna de negócio `baixa_carga` e exporta `derived/kpm_features.csv` + `derived/etl_qc.json`.
- `02_eda_kpm.ipynb` consome o CSV, roda as 2 consultas, calcula os indicadores preliminares e exporta os 2 plots em `figures/`.
- Kernel: Python 3 (testado com Python 3.14, pandas 3.0.5, matplotlib recentes — ver `requirements.txt` de referência em `repo/data/code/notebooks/requirements.txt`).

## Timezone e qualidade dos dados (QC)

- `ingested_at` está em **UTC** (ISO 8601, sufixo `+00:00`).
- 100 linhas, sem payload vazio, **sem duplicatas** por `(run_id, phase, sample_index)`.
- `DRB.UEThpDl` está **100% nulo** nesta amostra — gap de coleta, não usado nos indicadores do grupo (registrado como limitação).
- `DRB.UEThpUl`, `DRB.RlcSduDelayDl`, `RRU.PrbTotUl`: sem nulos.
- Apenas **1 `run_id`** disponível: agregações "por fase" descrevem este experimento específico, não um comportamento estatístico geral da rede.

## Resumo dos achados (TL;DR)

A `baseline` e quase toda a fase `recovery` operam em baixa carga (PRB UL ≈ 2%) sem degradação de delay, enquanto `stress` mantém PRB ~97–99% o tempo todo — um contraste nítido que sustenta a hipótese de economia de energia nessas janelas. O limiar de 10% de PRB separa as fases de forma quase perfeita (FBC de 100%, 95% e 1,7%, respectivamente).

## Indicadores (KPI/KQI)

### KPI 1 — Fração de tempo em baixa carga, por fase

```
FBC(fase) = (nº de amostras com RRU.PrbTotUl <= 10%) / (nº total de amostras na fase)
```

- **Unidade:** % (fração × 100).
- **Granularidade:** por fase (`baseline` / `stress` / `recovery`), dentro do `run_id` único disponível.
- **Fonte:** `kpm_samples.payload_json → RRU.PrbTotUl`, materializada em `derived/kpm_features.csv` (coluna `baixa_carga`).
- **Limiar (10% de PRB UL):** escolhido por ser bem acima do platô observado no baseline (mediana e MAD = 0, valor constante ≈ 2%) e bem abaixo do platô observado no stress (≈ 97–99%), evitando classificar erroneamente qualquer amostra de stress como baixa carga.
- **Interpretação:** quanto maior a FBC de uma fase, mais amostras dessa fase operam com utilização de rádio muito baixa — candidatas a uma janela de economia de energia.
- **Limite de validade:** é um indicador de *oportunidade* baseado em uso de PRB, não uma medição de consumo de energia; vale apenas para o `run_id` único desta amostra (não generaliza estatisticamente para outras redes/cargas).
- **Gráfico dedicado:** `figures/01_kpi1_prb_janela_temporal.png`.

**Resultado observado nesta amostra:**

| Fase | FBC |
|---|---|
| baseline | 100,0% |
| stress | 1,7% (1 de 60 amostras — outlier pontual) |
| recovery | 95,0% |

### KQI 2 — Vazão UL média e delay médio dentro das janelas de baixa carga

```
ThpUL_bc(fase) = média(DRB.UEThpUl | baixa_carga = True)
Delay_bc(fase) = média(DRB.RlcSduDelayDl | baixa_carga = True)
```

- **Unidade:** Mbps (vazão, conforme unidade do lab) e ms (delay).
- **Granularidade:** por fase, restrito às amostras com `baixa_carga = True`.
- **Fonte:** mesmas colunas de `payload_json`, mesma tabela derivada.
- **Interpretação:** vazão e delay estáveis dentro das janelas de baixa carga indicam que
  reduzir o uso de rádio nesses trechos não compromete a qualidade percebida — se caíssem
  junto com o PRB, a baixa carga seria sinal de degradação, não de oportunidade.
- **Limite de validade:** amostra pequena (100 pontos, poucos UEs); a célula
  `stress | baixa_carga=True` tem apenas 1 ponto e não deve ser lida como tendência.
- **Gráfico dedicado:** `figures/02_kqi2_qualidade_janela_temporal.png`.

**Resultado observado nesta amostra:**

| Fase | n amostras em baixa carga | Vazão UL média (Mbps) | Delay médio (ms) |
|---|---|---|---|
| baseline | 20 | 3,72 | 55,25 |
| stress | 1 | 15,16 | 134,79 |
| recovery | 19 | 3,68 | 81,21 |

### O que os indicadores NÃO provam

- FBC é um **proxy de oportunidade** de economia de energia baseado em uso de PRB — **não mede consumo de energia real**; o lab RFSIM não instrumenta potência de RU.
- ThpUL_bc/Delay_bc descrevem qualidade média nas janelas identificadas, mas com amostra pequena (100 pontos, 1 único `run_id`, poucos UEs) — **não têm poder estatístico** para generalizar a tráfego real de campo.
- A amostra de `stress | baixa_carga = True` tem **apenas 1 ponto** — não é uma tendência, é um outlier pontual dentro da fase de carga; não deve ser interpretada como "stress também tem baixa carga".
- Não há atuação física na RAN: qualquer "política de economia" derivada destes indicadores é uma **intenção simulada** (dry-run), nunca `AI_POLICY_COMMIT=1` fora de demonstração docente.

## Discussão rápida (CP2)

**Que decisão o indicador habilita?** Quando a FBC de uma fase é alta (recovery ≈ 95%,
baseline = 100%) e o KQI 2 confirma vazão/delay estáveis nessas mesmas janelas, o indicador
habilita abrir uma **política A1 candidata de economia de energia em dry-run** (ex.: reduzir
a potência de referência simulada do RU) para essas janelas — nunca uma atuação real na RAN.
Se o KQI 2 mostrasse degradação de delay dentro da baixa carga, a decisão correta seria
**não** acionar a política, pois a baixa carga estaria mascarando um problema, não uma folga.

**O limiar de 10% é um SLA didático?** Sim. O valor não vem de um SLA de operadora nem de uma
especificação 3GPP — foi escolhido de forma didática, a partir da separação natural entre os
platôs observados no baseline (~2%) e no stress (~97–99%) **desta amostra específica**. Ele
serve para demonstrar o raciocínio de limiarização de forma reprodutível, não para ser usado
como parâmetro operacional real de uma rede em produção.

## Visualizações

Um gráfico dedicado por indicador, com título, eixos com unidade e uma **janela temporal
contínua** (índice cronológico do experimento, `baseline → stress → recovery`, sem reiniciar
a cada fase):

1. **`figures/01_kpi1_prb_janela_temporal.png`** (KPI 1) — `RRU.PrbTotUl` (%) ao longo de todo
   o experimento, com as amostras `baixa_carga=True` marcadas, a linha do limiar (10%) e o
   valor de FBC anotado diretamente sobre cada fase.
   *Insight:* a fase `stress` mantém PRB ~97–99% e praticamente não gera janelas de baixa
   carga (FBC=1,7%); `baseline` (FBC=100%) e a quase totalidade de `recovery` (FBC=95%) ficam
   abaixo do limiar.
2. **`figures/02_kqi2_qualidade_janela_temporal.png`** (KQI 2) — vazão UL (círculos, eixo
   esquerdo, Mbps) e delay (cruzes, eixo direito, ms), restritos às amostras `baixa_carga=True`,
   ao longo do mesmo eixo cronológico.
   *Insight:* dentro das janelas de baixa carga, vazão e delay permanecem estáveis tanto no
   início (`baseline`) quanto no fim (`recovery`) do experimento, sem indício de degradação
   associada à baixa utilização de PRB — reforçando que essas janelas são seguras para uma
   política de economia simulada.

## Estrutura da pasta

```
analise-de-dados-aplicada-a-redes-de-telecomunicacoes/
  README.md
  notebooks/
    01_etl_kpm.ipynb     # Extract-Transform-Load + QC + coluna baixa_carga
    02_eda_kpm.ipynb     # 2 consultas, indicadores preliminares/formais, 1 gráfico por KPI/KQI
  figures/
    01_kpi1_prb_janela_temporal.png       # gráfico dedicado ao KPI 1
    02_kqi2_qualidade_janela_temporal.png # gráfico dedicado ao KQI 2
  derived/
    kpm_features.csv     # tabela de features com coluna baixa_carga
    etl_qc.json           # relatório de qualidade do ETL
```
