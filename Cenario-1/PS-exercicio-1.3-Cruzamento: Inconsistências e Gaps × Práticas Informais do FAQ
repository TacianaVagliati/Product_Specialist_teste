# Exercício 1.3 — Cruzamento: Inconsistências e Gaps × Práticas Informais do FAQ

**Documentos de entrada:**
- `1-1-mapa-cobertura-tematica-novatech.md` — gaps da base documental
- `1-2-analise-inconsistencias-proc042-v1-vs-v2.md` — conflitos entre versões da PROC-042
- `FAQ-Atendimento` (não controlado) — conhecimento prático do time de atendimento

**Data de análise:** Junho 2025  
**Elaborado por:** QA — DB1 Group

---

## Como ler este documento

Cada item do FAQ foi cruzado com os documentos oficiais e classificado em uma das quatro situações:

| Classificação | Significado |
|---------------|-------------|
| ✅ Alinhado | FAQ está consistente com o documento oficial. |
| ⚠️ Parcialmente alinhado | FAQ está correto em parte, mas incompleto ou impreciso em algum ponto. |
| ❌ Conflito direto | FAQ contradiz um documento oficial em pelo menos um ponto mensurável. |
| 🔴 Risco RAG | A divergência tem potencial direto de gerar resposta errada no assistente. |

---

## Cruzamento por item do FAQ

---

### FAQ Item 3 — "Cliente perguntou se pode devolver carga perigosa. O que respondo?"

**O que o FAQ diz:**
> "Na prática, a gente orienta o cliente a ligar no ramal 4500 (Gestão de Riscos). Oficialmente não pode pelo processo padrão, mas já tiveram casos em que o pessoal de Riscos autorizou exceção. Então não diga que é impossível — diga que precisa de tratamento especial."

**O que os documentos oficiais dizem:**
- POL-001 §3.2: cargas perigosas classes 1 a 6 da ANTT **não são elegíveis para devolução pelo processo padrão**. O cliente deve contatar o setor de Gestão de Riscos (ramal 4500) para tratamento individual.

**Classificação:** ⚠️ Parcialmente alinhado

**Análise:**
O FAQ alinha com a POL-001 no ponto central (ramal 4500, tratamento especial). O problema está na frase "já tiveram casos em que o pessoal de Riscos autorizou exceção" — esse comportamento de exceção não está documentado em nenhum PROC e não deve ser apresentado como expectativa ao cliente. Se o RAG recuperar o FAQ junto com a POL-001, pode gerar resposta ambígua: "não pode pelo processo padrão, mas pode ter exceção" — o que é diferente do que a política oficial permite afirmar.

**Risco para o RAG:** O assistente não tem base para confirmar ou negar exceções — e não deve. O guardrail correto é escalar ao ramal 4500, sem criar expectativa de resultado.

---

### FAQ Item 8 — "Como funciona o frete especial?"

**O que o FAQ diz:**
> "Acima de 500kg, aplica a tabela de multiplicadores por região. Cuidado: existem duas versões da PROC-042. A mais recente tem multiplicadores mais altos. Na dúvida, use a mais recente (v2), mas se o cliente reclamar do valor, pode ser que o contrato dele ainda esteja na tabela antiga."

**O que os documentos oficiais dizem:**
- PROC-042 v1 (mar/2023): multiplicadores menores em todas as regiões.
- PROC-042 v2 (nov/2023): multiplicadores maiores; seção de disposições transitórias mantém v1 válida para chamados abertos antes de 01/12/2023.
- Nenhuma versão declara formalmente que substitui a outra.

**Classificação:** ⚠️ Parcialmente alinhado + 🔴 Risco RAG

**Análise:**
O FAQ reconhece corretamente a coexistência das duas versões e recomenda usar a v2 por padrão — o que é um bom julgamento prático. Porém:

1. A justificativa "se o cliente reclamar, pode ser que o contrato dele ainda esteja na tabela antiga" é uma heurística informal sem critério formal. O documento correto para definir isso seria a seção de disposições transitórias da v2, que usa a data de abertura do chamado (01/12/2023) — não o contrato do cliente.
2. O FAQ não informa os valores corretos de nenhuma das versões — apenas diz que a v2 "tem multiplicadores mais altos", sem dar os números.

| Ponto | FAQ | Oficial (v2) |
|-------|-----|--------------|
| Critério para usar qual versão | "Se o cliente reclamar" | Data de abertura do chamado (antes/depois de 01/12/2023) |
| Valores dos multiplicadores | Não informa | Sul 1.3, Sudeste 1.1, CO 1.4, NE 1.5, Norte 1.8 |

**Risco para o RAG:** Se o RAG indexar o FAQ junto com as duas versões da PROC-042, pode reproduzir o critério informal ("depende do contrato do cliente") em vez do critério documental correto (data do chamado). Isso é um critério de negócio — e o errado pode gerar cobrança incorreta.

---

### FAQ Item 15 — "Cliente diz que é Platinum. Existe esse tier?"

**O que o FAQ diz:**
> "Não existe tier Platinum na NovaTech. Às vezes o cliente confunde com outra transportadora ou com o programa de fidelidade antigo que foi descontinuado em 2022. Oriente que nossos tiers são Gold, Silver e Standard e peça o número do contrato para verificar."

**O que os documentos oficiais dizem:**
- SLA-2024 §1: classifica clientes em Gold, Silver e Standard. Nota explícita: "Não existem outros tiers além dos três listados acima."
- Não existe nenhum documento oficial sobre o encerramento do programa de fidelidade/tier Platinum.

**Classificação:** ✅ Alinhado (na orientação ao atendente) + gap documental

**Análise:**
O FAQ está correto: Platinum não existe, e os tiers são Gold, Silver e Standard — alinhado com SLA-2024 §1. O gap está na ausência de um comunicado formal sobre a descontinuação do programa em 2022. Se o assistente RAG for perguntado diretamente "o que é o tier Platinum?", vai encontrar apenas a menção informal do FAQ, sem documento de referência para citar. Isso não gera risco de resposta errada, mas gera resposta sem fonte — o que viola o requisito de citação de origem.

---

### FAQ Item 22 — "Cliente quer saber sobre seguro de carga. O que falar?"

**O que o FAQ diz:**
> "A NovaTech oferece seguro de carga como adicional. O valor é 0,3% do valor declarado da mercadoria para cargas padrão e 0,8% para cargas perigosas. Detalhe: isso vale para contratos a partir de 2023. Contratos mais antigos podem ter percentuais diferentes — confirme com o Comercial."

**O que os documentos oficiais dizem:**
- POL-001, PROC-042 v1, PROC-042 v2, SLA-2024: **nenhum documento menciona seguro de carga**.

**Classificação:** 🔴 Risco RAG — ausência total de cobertura oficial

**Análise:**
Este é o gap mais crítico do FAQ. Os percentuais de seguro (0,3% e 0,8%) existem apenas no FAQ não controlado, sem nenhuma base documental oficial. Os riscos são:

1. **Risco de alucinação:** o LLM pode combinar os percentuais do FAQ com informações de outros documentos e gerar uma resposta inventada com aparência de certeza.
2. **Risco de informação desatualizada:** o FAQ não tem data de atualização — esses percentuais podem ter mudado.
3. **Risco contratual:** seguro de carga tem impacto financeiro direto. Uma resposta incorreta ao cliente pode gerar contestação.
4. **Contratos pré-2023:** o FAQ menciona percentuais diferentes para contratos antigos sem dar os valores e sem indicar onde consultá-los.

**Ação necessária antes da indexação:** Localizar e incluir na base o documento oficial de seguro de carga (apólice, tabela comercial ou PROC específico). Enquanto não existir, o guardrail deve instruir o assistente a encaminhar ao Comercial — sem citar os percentuais do FAQ.

---

### FAQ Item 27 — "O tracking mostra 'em trânsito' há 5 dias. O que faço?"

**O que o FAQ diz:**
> "Depende da rota. Rotas para o Norte podem levar até 10 dias úteis. Para Sul/Sudeste, mais de 3 dias parado é estranho. Abra um chamado de rastreamento e classifique como prioridade alta se for Gold ou se o valor da carga for acima de R$ 50.000."

**O que os documentos oficiais dizem:**
- SLA-2024 §3: incidente crítico se carga com **valor declarado acima de R$ 100.000** estiver com status desconhecido há mais de 6 horas.
- Não existe tabela oficial de prazos de entrega por rota.

**Classificação:** ❌ Conflito direto + 🔴 Risco RAG

**Análise:**
Dois conflitos mensuráveis:

| Ponto | FAQ (informal) | SLA-2024 (oficial) | Diferença |
|-------|---------------|-------------------|-----------|
| Threshold para prioridade alta/crítico | > R$ 50.000 | > R$ 100.000 | 2× maior no oficial |
| Tempo sem status para acionar | Não especificado (implícito: 5 dias) | > 6 horas (para incidente crítico) | Muito diferente |

O FAQ usa R$ 50.000 como threshold; o SLA-2024 usa R$ 100.000. Um atendente seguindo o FAQ vai classificar como crítico algo que o documento oficial não classifica — gerando dados de SLA distorcidos e possivelmente consumindo o SLA de incidentes críticos desnecessariamente.

O segundo conflito é ainda mais grave: o FAQ sugere "5 dias parado é estranho" como gatilho; o SLA-2024 diz que incidentes críticos Gold devem ser respondidos em 30 minutos — o que implica acionar muito antes de 5 dias.

Sobre os prazos de entrega por rota (Norte = 10 dias úteis): essa é a única menção em toda a base. Não há tabela oficial. O RAG não pode citar isso com fonte.

---

### FAQ Item 32 — "Pode enviar carga perigosa com frete expresso?"

**O que o FAQ diz:**
> "Sim, mas precisa de autorização do Compliance e a documentação ANTT tem que estar atualizada. Na prática, demora uns 2 dias para conseguir a autorização, então o 'expresso' acaba não sendo tão expresso. Avise o cliente sobre isso."

**O que os documentos oficiais dizem:**
- POL-001: trata cargas perigosas na devolução (§3.2), mas não cobre frete expresso.
- PROC-042 v1 e v2: cobrem frete especial acima de 500kg, mas não mencionam modalidade "expresso" para cargas perigosas.
- PROC-043 (citada como referência para frete de cargas perigosas): **não disponível na base**.

**Classificação:** 🔴 Risco RAG — ausência total de cobertura oficial

**Análise:**
O FAQ descreve um processo real (autorização do Compliance + ANTT atualizada + 2 dias de prazo) que não tem nenhum respaldo documental na base atual. O PROC-043 seria o documento correto para isso — e está ausente.

Agravante: a v2 da PROC-042 menciona que a PROC-043 "está em processo de revisão pelo Compliance e pode sofrer alterações". Isso significa que mesmo quando o PROC-043 for localizado, seu conteúdo pode estar desatualizado.

**Ação necessária:** Solicitar ao Compliance o status atual da PROC-043 e o procedimento oficial para frete expresso de cargas perigosas. Até lá, o assistente deve encaminhar ao Compliance sem descrever o processo.

---

### FAQ Item 38 — "Cliente quer saber a política para carga que chegou danificada."

**O que o FAQ diz:**
> "Carga danificada em trânsito tem processo diferente de devolução. O cliente precisa registrar a ocorrência em até 48h após o recebimento, com fotos e laudo se possível. A NovaTech investiga e, se comprovada responsabilidade nossa, reembolsa integralmente. Mas isso passa pelo Jurídico, não pelo atendimento normal — encaminhe para o e-mail sinistros@novatech.com.br."

**O que os documentos oficiais dizem:**
- POL-001 §2: "não se aplica a mercadorias ainda em trânsito — consultar PROC-088."
- POL-001 §3.5: "defeito ou erro da NovaTech (avaria em trânsito): devolução sem custo para o cliente" — mas o procedimento descrito na §3.3 é o de devolução padrão, não de sinistro.
- PROC-088: citado mas não disponível na base.

**Classificação:** ⚠️ Parcialmente alinhado + 🔴 Risco RAG

**Análise:**
O FAQ descreve um processo de sinistro (48h, fotos, laudo, Jurídico, e-mail) que é distinto do processo de devolução da POL-001. O FAQ está provavelmente correto na prática — mas nenhum documento oficial normaliza esse fluxo.

O risco para o RAG é concreto: se o atendente perguntar "o que fazer quando a carga chegou danificada?", o RAG pode recuperar a POL-001 §3.5 (que menciona reembolso por avaria) e o procedimento de devolução padrão (§3.3), gerando uma resposta que confunde sinistro com devolução. O prazo de 48h do FAQ não existe em nenhum documento oficial — o RAG não tem base para citá-lo.

| Ponto | FAQ (informal) | POL-001 (oficial) |
|-------|---------------|------------------|
| Prazo para registro | 48h após recebimento | Não especificado para este cenário |
| Canal | sinistros@novatech.com.br | Não mencionado |
| Responsável | Jurídico | Não mencionado |
| Processo | Investigação + reembolso integral | §3.5 menciona reembolso mas via processo de devolução padrão |

---

### FAQ Item 41 — "Qual a diferença entre SLA de resposta e SLA de resolução?"

**O que o FAQ diz:**
> "Resposta é quando a gente dá o primeiro retorno ao cliente (mesmo que seja 'estamos verificando'). Resolução é quando o problema é efetivamente resolvido. O Gold tem 2h de resposta e 24h de resolução. Silver é 4h e 48h. Standard é 8h e 72h. Para incidentes críticos, os prazos são menores — veja a tabela SLA-2024."

**O que os documentos oficiais dizem:**
- SLA-2024 §2: Gold 2h/24h, Silver 4h/48h, Standard 8h/72h — exatamente os valores citados.

**Classificação:** ✅ Alinhado

**Análise:**
Um dos poucos itens do FAQ completamente alinhados com os documentos oficiais, inclusive com valores numéricos corretos. O FAQ inclusive instrui o atendente a consultar a tabela SLA-2024 para incidentes críticos — boa prática de referenciamento.

---

### FAQ Item 45 — "O cliente quer desconto no frete. Posso dar?"

**O que o FAQ diz:**
> "Atendente não tem autonomia para dar desconto. Para clientes com mais de 10 fretes especiais por mês, existe desconto automático na tabela (veja PROC-042). Para outros casos, encaminhe ao Comercial com justificativa."

**O que os documentos oficiais dizem:**
- PROC-042 v1: desconto para **mais de 10 fretes/mês**, mas via negociação com o Comercial — **não automático**.
- PROC-042 v2: desconto **automático** de 5% a partir de **8 fretes/mês**; 10% acima de 15 fretes/mês.

**Classificação:** ❌ Conflito direto + 🔴 Risco RAG

**Análise:**
O FAQ criou uma terceira versão informal que não corresponde a nenhum documento:

| Ponto | FAQ (informal) | PROC-042 v1 | PROC-042 v2 |
|-------|---------------|------------|------------|
| Threshold | > 10 fretes/mês | > 10 fretes/mês | ≥ 8 fretes/mês |
| Mecanismo | Automático | Negociar com Comercial | Automático (5% ou 10%) |

O FAQ pegou o threshold da v1 (>10) e o mecanismo da v2 (automático) — combinando duas regras que não são compatíveis entre si. O resultado é que:

- Um cliente com **9 fretes/mês** → FAQ diz: sem desconto. v2 diz: 5% automático.
- Um cliente com **11 fretes/mês** → FAQ diz: desconto automático. v1 diz: negociar com Comercial (sem % garantido). v2 diz: 5% automático.

O RAG vai herdar essa ambiguidade — e qualquer resposta sobre desconto de volume estará potencialmente errada até que a hierarquia entre as versões da PROC-042 seja formalizada.

---

## Consolidado: Classificação de todos os itens do FAQ

| Item FAQ | Tema | Classificação | Risco RAG |
|----------|------|---------------|-----------|
| Item 3 | Devolução de carga perigosa | ⚠️ Parcialmente alinhado | Baixo — desde que o guardrail de escalação ao ramal 4500 seja claro |
| Item 8 | Frete especial — duas versões | ⚠️ Parcialmente alinhado | 🔴 Alto — critério informal de "contrato do cliente" vs. data do chamado |
| Item 15 | Tier Platinum | ✅ Alinhado | Baixo — gap de citação de fonte, mas resposta correta |
| Item 22 | Seguro de carga | 🔴 Sem cobertura oficial | 🔴 Alto — valores sem nenhum respaldo documental |
| Item 27 | Tracking em trânsito | ❌ Conflito direto | 🔴 Alto — threshold R$50k vs. R$100k (SLA-2024) |
| Item 32 | Frete expresso + perigosa | 🔴 Sem cobertura oficial | 🔴 Alto — processo real sem PROC, PROC-043 ausente |
| Item 38 | Carga danificada | ⚠️ Parcialmente alinhado | 🔴 Alto — prazo 48h e canal Jurídico sem respaldo oficial |
| Item 41 | SLA resposta vs. resolução | ✅ Alinhado | Nenhum |
| Item 45 | Desconto no frete | ❌ Conflito direto | 🔴 Alto — threshold e mecanismo ambos errados |

---

## Impacto consolidado para o assistente RAG

### Itens que NÃO devem ser indexados sem normalização prévia

| Item FAQ | Motivo |
|----------|--------|
| Item 22 (seguro) | Valores sem documento oficial — alto risco de alucinação ou citação de fonte inválida. |
| Item 32 (frete expresso + perigosa) | Processo sem PROC — PROC-043 precisa ser localizado antes. |
| Item 45 (desconto) | Regra híbrida inválida — só pode ser usada após formalização da hierarquia PROC-042 v1/v2. |

### Itens que podem ser indexados com guardrail de escalação

| Item FAQ | Guardrail recomendado |
|----------|----------------------|
| Item 3 (carga perigosa) | "Encaminhe ao ramal 4500. Não afirme possibilidade de exceção." |
| Item 8 (frete especial) | "Cite a v2 como referência. Oriente confirmação com o Comercial para contratos anteriores a dez/2023." |
| Item 38 (carga danificada) | "Encaminhe para sinistros@novatech.com.br. Não informe prazo de 48h sem respaldo oficial." |
| Item 27 (tracking) | "Use o critério oficial de R$100.000 para incidente crítico. Ignorar o threshold de R$50.000 do FAQ." |

### Itens seguros para indexação

| Item FAQ | Motivo |
|----------|--------|
| Item 15 (Platinum) | Alinhado com SLA-2024. Guardrail de "sem fonte oficial para a descontinuação" é suficiente. |
| Item 41 (SLA) | Completamente alinhado com SLA-2024. Pode ser indexado normalmente. |

---

## Recomendações para o PS — Ações antes do go-live

| Prioridade | Ação | Impacto |
|------------|------|---------|
| 🔴 1 | Localizar ou criar documento oficial de seguro de carga (percentuais, vigência, contratos legados). | Impede indexação do FAQ item 22. |
| 🔴 2 | Formalizar hierarquia PROC-042 v1/v2 (nota técnica com data de vigência definitiva da v2). | Destrava os itens FAQ 8 e 45 e todos os casos de teste de frete especial. |
| 🔴 3 | Localizar PROC-043 e verificar status da revisão do Compliance. | Impede indexação do FAQ item 32. |
| 🟡 4 | Criar PROC de sinistros/carga danificada (formalizar: prazo, canal, responsável). | Dá respaldo oficial ao FAQ item 38, hoje o fluxo mais usado sem documentação. |
| 🟡 5 | Criar tabela oficial de prazos de entrega por rota. | Dá base para respostas sobre FAQ item 27 (Norte = 10 dias úteis). |
| 🟡 6 | Emitir comunicado formal sobre encerramento do tier Platinum. | Resolve o gap de citação de fonte no FAQ item 15. |
| 🟢 7 | Incluir os itens 15 e 41 do FAQ na base após validação de Compliance. | Enriquece o RAG com conhecimento prático sem risco. |

---

> **Observação para o QA:** Todos os casos de teste que envolvam temas dos itens 22, 32 e 45 do FAQ devem ser bloqueados ou marcados como `não testável com confiabilidade` até que as ações de prioridade 🔴 sejam concluídas. O FAQ não é fonte válida para geração de casos de teste com critério de aprovação definido — apenas os documentos oficiais o são.

