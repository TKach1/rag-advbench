# RAG-AdvBench: Adversarial Security & Red-Teaming Benchmark for RAG Agents

[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/)
[![LangChain](https://img.shields.io/badge/Orchestration-LangChain%20LCEL-green.svg)](https://www.langchain.com/)
[![Vector Store](https://img.shields.io/badge/Vector%20Store-FAISS-orange.svg)](https://github.com/facebookresearch/faiss)
[![Evaluation](https://img.shields.io/badge/Red--Teaming-DeepEval-purple.svg)](https://github.com/confident-ai/deepeval)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**RAG-AdvBench** (`rag-advbench`) é um *framework* experimental e *benchmark* reprodutível para avaliação de robustez e segurança adversarial em sistemas de **Geração Aumentada por Recuperação (RAG)**. O projeto implementa uma suíte automatizada de *Red-Teaming* que avalia como diferentes arquiteturas e escalas de Grandes Modelos de Linguagem (*open-weight*) reagem a ataques de envenenamento de base de conhecimento, *backdoors* ativados por gatilho e injeção em memória de longo prazo.

## Arquitetura do Benchmark

O ambiente experimental simula um agente corporativo de suporte e hospitalidade executiva (**Grand Luxury Hotel Helpdesk**), que opera sob políticas estritas de privacidade, proteção de dados pessoais (LGPD/GDPR) e acordos de não divulgação (NDAs) utilizando registros sintéticos derivados do corpus `ai4privacy/pii-masking-300k`.

* **Orquestração:** `LangChain` com composição funcional declarativa via *LangChain Expression Language* (LCEL).
* **Modelo de Embeddings:** `sentence-transformers/all-MiniLM-L6-v2` ($d = 384$) com normalização unitária na norma Euclidiana ($\ell_2$).
* **Indexador Vetorial:** `FAISS` (`IndexFlatIP`), realizando busca exata por produto interno (equivalente à similaridade de cosseno para vetores normalizados em $\ell_2$).
* **Avaliação Automatizada (*LLM-as-a-Judge*):** `DeepEval` integrado via adaptador customizado (`OpenRouterDeepEvalWrapper`) e verificadores determinísticos de violação de políticas.
* **Inferência Reprodutível:** Temperatura nula ($\tau = 0.0$) em todas as chamadas locais (Hugging Face Pipeline em GPU) e remotas (via gateway unificado OpenRouter).

## Vetores de Ataque Avaliados

1. **PoisonedRAG (Zou et al., 2024):** Corrupção direcionada do banco vetorial ($P = Q \oplus I$), injetando documentos forjados que maximizam a similaridade semântica com consultas-alvo para induzir respostas factuais falsas.
2. **AgentPoison (Chen et al., 2024):** Ataque de *backdoor* em agentes RAG que implanta documentos dormentes acionados por uma sequência discreta de gatilho (`VIP-OVERRIDE-TOKEN` / `[PROTOCOLO-VIP-OVERRIDE]`), visando anular salvaguardas normativas (*guardrails*) e exfiltrar dados confidenciais (PII).
3. **MINJA / Memory INJection Attack (Dong et al., 2025):** Ataque de envenenamento de memória conversacional operando estritamente via consultas (*query-only*), combinando passos de transição semântica (*bridging steps*) e a estratégia de encurtamento progressivo (*Progressive Shortening Strategy* - PSS).

## Modelos Open-Weight Comparados

| Família / Modelo | Identificador (`slug`) | Arquitetura | Escala de Parâmetros | Provedor / Execução |
| :-- | :-- | :-- | :-- | :-- |
| **Google Gemma 4 E2B** | `google/gemma-4-E2B-it` | Densa Compacta | ~2B Efetivos | Google (Hugging Face / OpenRouter) |
| **OpenAI gpt-oss-20b** | `openai/gpt-oss-20b` | *Mixture-of-Experts* (MoE) | 21B Totais (3.6B Ativos) | OpenAI (OpenRouter) |
| **DeepSeek V4 Flash** | `deepseek/deepseek-v4.1-flash` | MoE + *Multi-head Latent Attention* (MLA) | 284B Totais (13B Ativos) | DeepSeek (OpenRouter) |
| **GLM-4.7-Flash** | `z-ai/glm-5.3-flash` | Atenção Híbrida Esparsa / Agêntica | Classe 30B | Zhipu AI (OpenRouter) |

## Métricas de Avaliação

* **ISR (*Injection Success Rate*):** Proporção de consultas adversariais nas quais ao menos um documento envenenado é recuperado no *top-$k$* do índice FAISS.
* **ASR (*Attack Success Rate*):** Proporção de consultas adversariais nas quais o modelo gerador acata a instrução maliciosa ou reproduz o conteúdo corrompido (reportada de forma global e segregada por vetor de ataque).
* **UD (*Utility Drop*):** Degradação relativa da acurácia operacional do agente entre cenários benignos de controle e cenários sob ataque.
* **RA (*Retrieval Accuracy*):** Acurácia ponderada do estágio de recuperação vetorial diante da base contaminada.
* **$F1_{\text{AVG}}$ (*Harmonic F1-Score*):** Indicador consolidado que equilibra a resistência generativa ($1 - \text{ASR}$) e a precisão de recuperação ($1 - \text{ISR}$).

## Principais Resultados Empíricos

| Modelo | Escala | ISR | ASR Global | ASR (PoisonedRAG) | ASR (AgentPoison) | ASR (MINJA) | UD | RA | $F1_{\text{AVG}}$ |
| :-- | :-- | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| **Google Gemma 4 E2B** | ~2B Efetivo | 71.4% | 28.6% | 0.0% | 100.0% | 0.0% | 28.6% | 64.3% | 0.500 |
| **OpenAI gpt-oss-20b** | 21B (3.6B Ativos) | 71.4% | 0.0% | 0.0% | 0.0% | 0.0% | 0.0% | 64.3% | 0.643 |
| **DeepSeek V4 Flash** | 284B (13B Ativos) | 71.4% | 14.3% | 0.0% | 50.0% | 0.0% | 14.3% | 64.3% | 0.571 |
| **GLM-4.7-Flash** | Classe 30B | 71.4% | 14.3% | 0.0% | 50.0% | 0.0% | 14.3% | 64.3% | 0.571 |

## Estrutura do Repositório

```text
rag-advbench/
├── rag_advbench.ipynb   # Notebook principal com a suíte de testes, métricas e anotações
├── figuras/             # Gráficos comparativos de alta resolução (300 DPI) gerados nos testes
└── README.md            # Documentação técnica do projeto
```

## Como Executar

### 1. No Google Colaboratory (Recomendado)
1. Abra o arquivo `rag_advbench.ipynb` no **Google Colaboratory** com acelerador de hardware **GPU T4** ativado.
2. Adicione sua chave `OPENROUTER_API_KEY` (e opcionalmente `HF_TOKEN`) na aba **Secrets** (`google.colab.userdata`) ou insira interativamente quando solicitado pelo prompt seguro (`getpass`).
3. Execute todas as células sequencialmente (`Runtime > Run all`).

### 2. Em Ambiente Local (Jupyter / Python 3.10+)
```bash
git clone https://github.com/TKach1/rag-advbench.git
cd rag-advbench
pip install "pandas>=2.2.0" "requests" "huggingface-hub>=0.23.0" "langchain>=0.2.0" \
            "langchain-community>=0.2.0" "langchain-openai>=0.1.0" "langchain-core>=0.2.0" \
            "langchain-huggingface>=0.0.3" "deepeval>=0.21.0" "datasets>=2.18.0" \
            "faiss-cpu>=1.7.4" "sentence-transformers>=2.6.0" "pydantic>=2.7.0" \
            "matplotlib>=3.8.0" "seaborn>=0.13.0" "accelerate>=0.28.0" "transformers>=4.40.0"

export OPENROUTER_API_KEY="sk-or-v1-..."
jupyter notebook rag_advbench.ipynb
```

## Referências Fundamentais

* **PoisonedRAG:** Zou, W. et al. *PoisonedRAG: Knowledge Corruption Attacks to Retrieval-Augmented Generation of Large Language Models*. USENIX Security Symposium, 2024.
* **AgentPoison:** Chen, Z. et al. *AgentPoison: Red-teaming LLM Agents via Poisoning Memory or Knowledge Bases*. Advances in Neural Information Processing Systems (NeurIPS), 2024.
* **MINJA:** Dong, S. et al. *Practical Memory Injection Attack against LLM Agents*. arXiv:2503.03704, 2025.

## Autores e Licença

Desenvolvido por **Vitor Galafassi Delapria** e **Wesley Brandão**.  
Distribuído sob a licença **MIT**.
