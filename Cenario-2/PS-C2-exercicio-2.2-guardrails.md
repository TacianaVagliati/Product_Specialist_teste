# Exercício 2.2 — Guardrails Formalizados como Artefato de Produto
# Guardrails do NovaTech Assistant

> **Papel responsável:** Product Specialist
> **Exercício:** Cenário 2 — Ex. 2.2 (Product Specialist)
> **Status:** Aprovado para consumo por agentes de IA — referenciado em `AGENTS.md § Product Rules & Guardrails`
> **Fonte de verdade:** `docs/novatech/` (POL-001, PROC-042 v1/v2, SLA-2024, FAQ-atendimento) + guardrails informais do Cenário 1

## Contexto

No Cenário 1, o time definiu 4 guardrails informais: (1) sempre citar fonte, (2) nunca inventar prazos ou valores, (3) quando não encontrar resposta, dizer explicitamente, (4) responder em português formal. Esses guardrails eram descrições de intenção — não artefatos que um agente consegue consumir de forma determinística.

Três incidentes registrados em testes internos mostraram onde os guardrails informais falharam:

| ID | Incidente |
|----|-----------|
| **INC-01** | O assistente respondeu que o prazo de devolução para carga perigosa é 7 dias. Na realidade, cargas perigosas (classes 1-6 ANTT) **não** são elegíveis para devolução pelo processo padrão (POL-001 § 3.2). |
| **INC-02** | O assistente citou "PROC-042, seção 2" com os multiplicadores da v1 (desatualizada) em vez da v2 (vigente desde 10/11/2023). |
| **INC-03** | O assistente respondeu "Não encontrei informação sobre isso" para uma pergunta sobre SLA Gold, mas o documento SLA-2024 estava indexado e continha a resposta. |

Este documento formaliza os guardrails como artefato estruturado, com três propriedades que os guardrails informais não tinham: (a) categorização em DEVE / NÃO DEVE / QUANDO EM DÚVIDA, (b) classificação explícita de enforcement — via prompt (probabilístico) ou via código (determinístico) —, e (c) rastreabilidade a um risco concreto (um dos 3 incidentes).

**Nota sobre enforcement:** um guardrail só via prompt depende do modelo "obedecer" a cada chamada — é probabilístico. Guardrails que protegem contra risco alto (segurança de carga perigosa, valores financeiros) devem ter uma camada de verificação em código, não apenas instrução textual. No repositório, essa camada é `src/services/response-validator.ts` ("Validação determinística de respostas (harness)") — os guardrails marcados como **Código** ou **Híbrido** abaixo devem ter uma regra correspondente ali antes de irem para produção.

---

## DEVE

| ID | Guardrail | Enforcement | Justificativa | Incidente(s) |
|----|-----------|-------------|----------------|--------------|
| D-01 | Toda resposta que contenha informação normativa deve citar a fonte com identificador do documento e seção (ex: `POL-001 § 3.2`), incluindo o campo `source_document` no JSON de retorno, mesmo com confiança baixa. | **Híbrido** (prompt instrui a citar; código valida que `source_document` não está vazio antes de devolver a resposta ao atendente) | Citar é uma instrução de estilo (probabilística), mas a *presença do campo* é verificável de forma binária — por isso a validação final deve ser em código. | INC-02 |
| D-02 | Quando duas versões de um documento coexistirem (ex: PROC-042 v1 e v2), a resposta deve usar os valores da versão vigente (mais recente por metadado de data/vigência) e informar explicitamente que existe uma versão anterior. | **Híbrido** (pipeline de ingestão anexa metadado de vigência — determinístico; prompt instrui o modelo a priorizar o chunk marcado como vigente — probabilístico) | A extração do metadado de vigência é feita no pipeline (código), mas qual chunk o modelo efetivamente usa na resposta ainda depende do prompt. Os dois devem atuar juntos. | INC-02 |
| D-03 | Antes de responder qualquer pergunta que combine "carga perigosa" (classes 1-6 ANTT) com devolução, frete expresso ou qualquer benefício padrão, o assistente deve verificar a exceção documentada em POL-001 § 3.2 e responder de forma explícita sobre a restrição. | **Código** (regra determinística no `response-validator`: se a pergunta contém os termos "carga perigosa"/classes ANTT + "devolução", a resposta é bloqueada/reescrita caso não contenha a negativa) | Risco de segurança/compliance alto (transporte de produtos regulados) — não pode depender só do modelo lembrar da regra a cada chamada. | INC-01 |
| D-04 | Antes de declarar "não encontrei informação", o assistente deve ter consultado o índice de busca e confirmado que os documentos relevantes não retornaram chunk com score acima do threshold mínimo de confiança. | **Código** (o texto de "não encontrado" só pode ser emitido pelo `response-builder` quando o resultado da busca vier abaixo do threshold configurado) | O modelo, isolado, não tem como saber que "não encontrei" só é uma resposta verdadeira se a busca de fato falhou — isso é uma condição objetiva sobre o retrieval, verificável em código. | INC-03 |
| D-05 | Responder sempre em português formal, com tom institucional adequado ao atendimento ao cliente. | **Prompt** | É uma questão de estilo/tom, difícil (e desnecessário) de verificar deterministicamente. | Guardrail geral (Cenário 1) |
| D-06 | Usar apenas os tiers de cliente existentes (Gold, Silver, Standard) e, se o cliente mencionar um tier diferente (ex: "Platinum"), esclarecer que esse tier não existe na NovaTech. | **Híbrido** (prompt instrui a corrigir; código valida que o campo `customer_tier`, se presente na resposta, é um dos 3 valores do enum) | Corrigir o cliente é uma questão de tom (prompt), mas o dado estruturado retornado à interface nunca pode conter um valor fora do enum — isso é validável em código. | Risco correlato a INC-01/INC-03 (dados incorretos passando sem verificação) |

---

## NÃO DEVE

| ID | Guardrail | Enforcement | Justificativa | Incidente(s) |
|----|-----------|-------------|----------------|--------------|
| N-01 | Nunca gerar valores numéricos (prazos, multiplicadores, percentuais, SLAs) que não estejam literalmente presentes nos chunks recuperados para aquela resposta. | **Código** (harness compara todo número presente na resposta gerada contra os números existentes nos chunks recuperados; números "órfãos" reprovam a resposta) | Alucinação numérica é o risco mais caro do domínio (frete, prazos, reembolsos envolvem dinheiro e compliance) — inaceitável depender só de instrução de prompt. | INC-01, INC-02 |
| N-02 | Nunca afirmar que carga perigosa (classes 1-6 ANTT) pode ser devolvida pelo processo padrão de devolução. | **Código** (regra hardcoded — nenhuma resposta pode combinar "carga perigosa" + confirmação de devolução padrão; a resposta correta está fixada, não gerada) | Este é o guardrail de maior severidade do domínio — a única forma responsável de garantir 0% de falha é travar a resposta no código, com o prompt apenas reforçando a mesma regra como redundância. | INC-01 |
| N-03 | Nunca inventar tiers de cliente além de Gold, Silver e Standard (SLA-2024 § 1). | **Híbrido** (prompt reforça; código valida enum no campo `customer_tier`) | Mesmo raciocínio do D-06: correção de tom é prompt, integridade do dado estruturado é código. | Risco correlato a INC-01 (informação incorreta aceita sem checagem) |
| N-04 | Nunca declarar "não encontrado" sem antes ter executado a busca no índice e obtido score de recuperação abaixo do threshold para os documentos relevantes ao tema da pergunta. | **Código** | Mesma lógica do D-04 — a ausência de resposta é uma alegação factual sobre o estado do índice, não uma opinião do modelo. | INC-03 |
| N-05 | Nunca misturar, na mesma resposta, valores da PROC-042 v1 e da PROC-042 v2 sem indicar claramente qual versão está sendo usada em cada trecho. | **Prompt** (instrução de composição da resposta) — com apoio do metadado de vigência do D-02 | É uma regra de redação da resposta; o dado correto já chega separado por versão graças ao metadado (código), mas a forma de apresentar é responsabilidade do prompt. | INC-02 |

---

## QUANDO EM DÚVIDA

| ID | Guardrail | Enforcement | Justificativa | Incidente(s) |
|----|-----------|-------------|----------------|--------------|
| Q-01 | Se a pergunta envolver carga perigosa combinada com devolução, frete expresso ou qualquer solicitação de exceção, nunca dar uma resposta binária simples ("sim"/"não") — indicar que o caso requer tratamento especial e orientar o encaminhamento ao setor de Gestão de Riscos (ramal 4500), conforme FAQ item 3 e item 32. | **Prompt** | É uma instrução de comportamento de fallback para um caso "cinzento" real do domínio (a FAQ mostra que atendentes já autorizaram exceções) — não há uma resposta binária correta a codificar. | INC-01 |
| Q-02 | Se dois documentos com informação conflitante forem recuperados (ex: duas versões de uma PROC), priorizar a versão mais recente por metadado de vigência/data de emissão **e** informar explicitamente ao usuário que existe uma versão anterior com valores diferentes. | **Híbrido** (ordenação por metadado é código; a redação da dupla-informação é prompt) | Mesma lógica de D-02/N-05 — decisão objetiva sobre qual usar (código), mas a transparência sobre a existência de duas versões é uma escolha de redação (prompt). | INC-02 |
| Q-03 | Se a confiança de recuperação for baixa (poucos chunks relevantes, score abaixo do threshold), prefixar a resposta com aviso explícito de baixa confiança e sugerir escalonamento ao supervisor (ou ao setor pertinente: Comercial, Jurídico, Gestão de Riscos, conforme o tema). | **Híbrido** (o cálculo do score é código; a decisão de para qual setor escalar, dado o assunto, é uma inferência do prompt) | Uma condição mensurável (score) deve ser calculada de forma determinística; a orientação de encaminhamento por assunto ainda depende de compreensão de linguagem natural. | INC-03 |
| Q-04 | Se a pergunta cruzar duas categorias de domínio (ex: frete + devolução — ~15% dos casos, conforme discovery), decompor a resposta por categoria, citando a fonte de cada parte separadamente em vez de uma resposta única fundida. | **Prompt** | É uma regra de estruturação de resposta para lidar com ambiguidade de escopo — não há um valor numérico ou binário a validar em código. | Risco correlato a INC-02/INC-03 (perda de precisão ao fundir fontes diferentes) |

---

## Rastreabilidade — incidente → guardrails que o previnem

| Incidente | Guardrails relacionados |
|-----------|--------------------------|
| INC-01 — Prazo de devolução indevido para carga perigosa | D-03, N-02, Q-01 |
| INC-02 — Citação de versão desatualizada da PROC-042 | D-01, D-02, N-01, N-05, Q-02 |
| INC-03 — Falso "não encontrado" para SLA Gold | D-04, N-04, Q-03 |

## Próximos passos

- A lógica de `response-validator.ts` (código) deve implementar as regras marcadas como **Código** ou **Híbrido** antes que o endpoint de query vá para revisão de QA (ver `specs/query-endpoint/`).
- Este documento alimenta diretamente a seção **Product Rules & Guardrails** do `AGENTS.md` (Ex. 2.3).
