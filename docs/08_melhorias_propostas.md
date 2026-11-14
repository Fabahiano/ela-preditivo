# 08 — Melhorias propostas e métricas de avaliação adicionais
 

## 1. Melhorias no código

### M1 — Interface interativa de apoio à decisão (P1)

- **O que fazer:**
  1. Streamlit (ou Dash) com upload dos dois VCFs, sexo, idade atual e laudo de repetições do C9orf72.
  2. Exibir P(portador), risco vitalício com IC 95 %, F(t) com banda, risco em 10 e 20 anos e a tabela de qualidade da ingestão.
  3. Manter o aviso de uso (instrumento experimental) visível na tela e em qualquer exportação.
  4. Não gravar os VCFs enviados em disco além do necessário (LGPD, TCC 3.4).
- **Aceite:** a interface reproduz os números de `ela-preditivo executar` para os cenários 1 a 5 com a mesma semente; teste automatizado sem navegador chamando a função de pipeline.

### M2 — Cenário 4 no fluxo oficial (P1)

- **O que fazer:** quando um lócus esperado não existe na base real, usar a tabela curada como fallback e registrar a fonte em `fonte_anotacao`; no relatório, listar explicitamente os cenários pulados e o motivo.
- **Aceite:** `ela-preditivo tudo` produz os 5 cenários nos resultados; teste cobrindo o fallback.

### M3 — Modelo de Cox como comparação (P1)

- **O que fazer:** ajustar `CoxPHFitter` (Lifelines) na coorte sintética com gene e sexo como covariáveis e comparar com a cura mista (F(t), C-index, AIC onde aplicável). Registrar em `04_decisoes.md` que a cura mista continua sendo o modelo principal, pois o Cox não representa a fração que nunca manifesta.
- **Aceite:** tabela comparativa nos resultados; teste de recuperação do razão de risco entre sexos no C9orf72 (δ = ±1 ano).

### M4 — Transmissão do C9orf72 pelo número de repetições (P3)

- **Situação:** a expansão entra como dosagem 0/1/2 a partir do laudo.
- **Proposta:** modelar limiar de repetições e probabilidade de expansão ou contração na transmissão, com parâmetros no YAML e citação. Só implementar com evidência publicada; registrar como decisão em `04_decisoes.md`.
- **Módulos:** `genotipo.py`, `monte_carlo.py`, `config/parametros_literatura.yaml`.


### M7 — Fatores adicionais e limites do modelo (P3)

- O modelo assume independência entre genes e não usa ancestralidade, histórico familiar nem idade dos pais. Isso limita o "personalizado" da pergunta-problema.
- Antes de implementar, declarar explicitamente a limitação na seção 5 do TCC. Se houver evidência publicada, incluir fatores modificadores como parâmetros opcionais, nunca por padrão.

---

## 2. Métricas de avaliação

### Já existentes (`avaliacao.py`, tabelas 5 a 9)

| Métrica | Onde |
|---|---|
| Razão DP/EP do Monte Carlo (esperado ≈ 1) | Tabela 5 |
| Erro do MC contra o valor exato | Tabela 5 |
| F(∞) exato contra Kaplan-Meier; mediana exata contra Weibull (Lifelines); parâmetros recuperados | Tabela 7 |
| Tempo de execução (PyMC contra NumPy) | Tabela 8 |
| Sensibilidade aos parâmetros | Tabela 9 |

### A adicionar

| # | Métrica | Como calcular | Critério sugerido | Prio. |
|---|---|---|---|---|
| A1 | **Cobertura do IC 95 %** | Simular K famílias sintéticas (π verdadeiro sorteado da Beta); medir a fração em que o valor verdadeiro cai no intervalo de credibilidade | 0,93 a 0,97 | P2 |
| A2 | **Brier score** e curva de calibração | Sobre coorte sintética com desfecho conhecido, nos horizontes de 10 e 20 anos | Brier abaixo do modelo nulo (taxa média) | P2 |
| A3 | **C-index (Harrell)** | `lifelines.utils.concordance_index` na coorte sintética | Reportar com IC por bootstrap | P2 |
| A4 | **AUC dependente do tempo** | Em 10 e 20 anos (ex. `scikit-survival`, se aceita como dependência opcional) | Reportar por gene | P3 |
| A5 | **MAE da idade de início** | Erro absoluto médio entre idade prevista (mediana) e idade simulada, por gene | Reportar por gene e sexo | P2 |
| A6 | **Distância entre curvas** | Kolmogorov–Smirnov e Wasserstein entre F(t) estimada e F(t) teórica | Reportar por cenário | P3 |
| A7 | **Convergência do MCMC** | R-hat (< 1,01) e ESS, se o posterior (e não só o preditivo a priori) for amostrado | R-hat < 1,01; ESS > 400 | P3 |
| A8 | **Robustez a erros de entrada** | Injetar falsos negativos e positivos em uma fração dos sítios do VCF e medir a variação do risco | Variação máxima documentada | P2 |
| A9 | **Sensibilidade tipo tornado** | Variar cada parâmetro de `parametros_literatura.yaml` dentro do IC e ordenar pelo efeito no risco vitalício | Gráfico ordenado | P3 |
| A10 | **Utilidade percebida** | Questionário SUS e concordância com aconselhadores geneticistas, em pequena escala, com aprovação ética | Fora do escopo do código; registrar como etapa do TCC | P3 |

### Onde implementar

- Novas funções em `avaliacao.py` (uma por métrica), com saída em `resultados/avaliacao/tabelas/tabela_10_*.csv` em diante (a tabela 10 já existe como Kaplan-Meier em `executar`; usar numeração nova, ex. `tabela_11` em diante, para evitar colisão).
- Subcomando ou flag em `cli.py` (`avaliar --metricas completas`), mantendo o comportamento atual como padrão.
- Testes em `tests/` com sementes fixas e tolerâncias explícitas.

---
