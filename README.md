# Johansen & Engle-Granger Cointegration Scanner

[![Python](https://img.shields.io/badge/Python-3.8+-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![Exchange](https://img.shields.io/badge/Exchange-KuCoin-23AF91?style=flat-square)](https://kucoin.com)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)
[![Statsmodels](https://img.shields.io/badge/Statsmodels-Latest-blue?style=flat-square)](https://www.statsmodels.org)

> **Scanner de cointegração para criptomoedas** — identifica pares com relação estatística de longo prazo usando os testes de Johansen e Engle-Granger, com dados em tempo real da KuCoin.

---

## 📖 O que é cointegração?

Dois ativos são **cointegrados** quando, apesar de cada um seguir um caminho aleatório individualmente (random walk), existe uma combinação linear entre eles que é estacionária — ou seja, a diferença de preços tende a retornar a uma média ao longo do tempo.

Isso tem implicações diretas para **estratégias de pairs trading**: quando o spread entre dois ativos cointegrados se afasta da média histórica, há uma probabilidade estatística de convergência, criando oportunidades de entrada.

---

## 🔬 Metodologia

O script implementa dois testes complementares, em sequência:

### 1. Teste de Engle-Granger (1987)
O teste clássico bivariado de cointegração:
- Estima a regressão de mínimos quadrados ordinários (OLS) entre os dois ativos
- Aplica o **Augmented Dickey-Fuller (ADF)** nos resíduos da regressão
- Se os resíduos forem estacionários (p-value < 0.05), os ativos são cointegrados

**Vantagens:** simples, rápido, amplamente aceito  
**Limitações:** assume apenas uma relação de cointegração (vetor único)

### 2. Teste de Johansen (1988)
O teste multivariado baseado em modelos VAR (Vector Autoregression):
- Utiliza o teste da **Razão de Verossimilhança com Traço** (`lr1`)
- Compara a estatística do traço com valores críticos a 5% (`cvt[:, 1]`)
- Permite detectar múltiplos vetores de cointegração

**Vantagens:** mais robusto, detecta estrutura completa de cointegração  
**Desvantagem:** requer amostras maiores e é computacionalmente mais pesado

### Critério de aprovação
Um par é considerado **cointegrado** apenas se **ambos** os testes forem aprovados simultaneamente:

```
p_value_Engle-Granger < 0.05
    AND
johansen_trace_stat > johansen_critical_value_5%
```

Isso reduz drasticamente os falsos positivos e garante pares estatisticamente robustos.

---

## ⚙️ Pipeline de execução

```
1. TRIAGEM
   └── Filtra ativos USDT com ≥ 2 anos de histórico na KuCoin

2. DOWNLOAD
   └── Baixa dados OHLCV (1h) dos últimos 2 anos via ccxt

3. PROCESSAMENTO PARALELO
   ├── Gera todas as combinações possíveis (C(n,2))
   ├── Executa Engle-Granger + Johansen em cada par
   └── Usa joblib (n_jobs=-1) para utilizar todos os núcleos da CPU

4. EXPORTAÇÃO
   └── Salva pares aprovados em `pares_elite_kucoin.csv`
```

---

## 🚀 Instalação

### Pré-requisitos
- Python 3.8+
- pip

### Instale as dependências

```bash
pip install ccxt pandas numpy statsmodels joblib
```

Ou usando um arquivo de requirements:

```bash
pip install -r requirements.txt
```

**requirements.txt:**
```
ccxt
pandas
numpy
statsmodels
joblib
```

---

## ▶️ Como usar

```bash
python CódigoPrincipal
```

O script irá:
1. Conectar à API pública da KuCoin (sem autenticação necessária)
2. Listar e filtrar pares USDT qualificados
3. Baixar os dados históricos (pode levar alguns minutos)
4. Processar todas as combinações em paralelo
5. Salvar os resultados em `pares_elite_kucoin.csv`

### Saída esperada

```
🔍 Iniciando triagem de pares com 2 anos de histórico...
📥 Baixando 84 ativos (OHLCV 1h)...
🚀 Processando 3486 combinações em paralelo...

=== TOP PARES COINTEGRADOS ===
              Par   Amostra  P-Value_EG  Johansen_Stat  Status
0  BTC/USDT vs ETH/USDT   17520      0.0023         18.42  Aprovado
1  BNB/USDT vs SOL/USDT   17520      0.0031         15.87  Aprovado
...
```

O arquivo CSV conterá:
| Coluna | Descrição |
|--------|-----------|
| `Par` | Os dois ativos avaliados |
| `Amostra` | Número de observações horárias compartilhadas |
| `P-Value_EG` | p-value do teste Engle-Granger (menor = melhor) |
| `Johansen_Stat` | Estatística do traço de Johansen |
| `Status` | `Aprovado` para pares cointegrados |

---

## 🔧 Configurações

Você pode ajustar os parâmetros diretamente no código:

```python
# Número de anos mínimos de histórico na triagem
realizar_triage(exchange, min_anos=2)

# Janela de dados baixados (em milissegundos)
since = exchange.milliseconds() - (730 * 24 * 60 * 60 * 1000)  # 2 anos

# Limite de ativos na triagem (remova [:150] para varrer todos os 900+ pares)
pares_usdt[:150]

# Amostra mínima por par (em candles de 1h)
if len(par_data) < 4000: return None  # ~166 dias

# Nível de significância Engle-Granger
if p_value_eg < 0.05 ...

# Número de núcleos (n_jobs=-1 = todos os núcleos disponíveis)
Parallel(n_jobs=-1, verbose=5)(...)
```

---

## 📊 Interpretação dos resultados

| P-Value EG | Interpretação |
|------------|---------------|
| < 0.01 | Cointegração muito forte — par de alta confiança |
| 0.01 – 0.05 | Cointegração significativa — par válido |
| > 0.05 | Sem evidência de cointegração — descartado |

> ⚠️ **Aviso:** Cointegração estatística não é garantia de lucro. Spreads podem divergir por longos períodos antes de convergir. Use gestão de risco adequada.

---

## 🧠 Base teórica

- **Engle, R.F. & Granger, C.W.J. (1987).** *Co-integration and error correction: Representation, estimation, and testing.* Econometrica, 55(2), 251–276.  
- **Johansen, S. (1988).** *Statistical analysis of cointegration vectors.* Journal of Economic Dynamics and Control, 12(2–3), 231–254.

---

## 📄 Licença

MIT — sinta-se livre para usar, modificar e distribuir.
