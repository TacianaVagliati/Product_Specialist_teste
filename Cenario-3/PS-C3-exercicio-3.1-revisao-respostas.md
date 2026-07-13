# Exercício 3.1 — Revisão Crítica das Respostas do Assistente
**Papel:** Product Specialist  
**Cenário:** 3  
**Programa:** Trilha AI First — DGS/DB1

> O guardrail `G-N04` (nunca usar FAQ informal como fonte isolada para tema crítico), citado nas justificativas abaixo, foi formalizado em `docs/guardrails.md` e propagado ao `AGENTS.md` como resultado direto desta revisão (INC-04). Ver também `specs/query-endpoint/harness-produto.md` (Ex. 3.2) para como esse guardrail entra no regression test e no HITL de runtime.

---

## Parte 1 — Avaliação Própria (antes do Claude)

Avaliação realizada com base no Anexo A e nos guardrails formalizados no Cenário 2.

---

### Resposta #1 — "Qual o prazo de devolução para produtos standard?"
**Resposta do assistente:** "O prazo é de 7 dias úteis após o recebimento. O cliente deve abrir chamado no portal e anexar fotos."
**Fonte citada:** POL-001, seção 3.2 | **Confiança:** Alta

**Minha avaliação: ✅ Correta**

Justificativa: O prazo de 7 dias úteis está correto (POL-001, seção 3.1). O procedimento de abertura de chamado com fotos está correto (POL-001, seção 3.3). A fonte citada é levemente imprecisa — a seção 3.2 é sobre exceções, não o prazo geral (que está na 3.1) — mas o conteúdo é factualmente correto. Avalio como adequada para go-live, com observação de que a citação de seção poderia ser mais precisa.

**Classificação:** Correta (citação de seção levemente imprecisa — não é erro factual)

---

### Resposta #2 — "Meu cliente é Silver. Qual o prazo de resolução?"
**Resposta do assistente:** "O cliente Silver tem resolução em até 48h."
**Fonte citada:** SLA-2024 | **Confiança:** Alta

**Minha avaliação: ✅ Correta**

Justificativa: O SLA de resolução para chamados gerais do cliente Silver é de 48h úteis (SLA-2024, seção 2). A informação está correta e a fonte é adequada. Observação: a resposta não distingue entre chamados gerais (48h) e incidentes críticos (8h) — poderia ser mais completa, mas é factualmente correta para a pergunta feita.

**Classificação:** Correta (poderia mencionar SLA diferenciado para incidentes críticos, mas não é erro)

---

### Resposta #3 — "Posso devolver carga perigosa classe 3?"
**Resposta do assistente:** "Não. Cargas perigosas (classes 1 a 6 da ANTT) não podem ser devolvidas pelo processo padrão. Recomendo escalar para o supervisor."
**Fonte citada:** POL-001, seção 3.2 | **Confiança:** Alta

**Minha avaliação: ✅ Correta**

Justificativa: A resposta aplica corretamente o guardrail G-N02. POL-001 seção 3.2 é exatamente a seção correta. A recomendação de escalar ao supervisor está alinhada com o comportamento esperado — embora o guardrail defina "ramal 4500 (Gestão de Riscos)" como destino específico, e a resposta usou "supervisor". Na interface real, o texto poderia ser mais preciso, mas a orientação está correta.

**Classificação:** Correta (recomendação de escalação poderia ser mais específica — ramal 4500)

---

### Resposta #4 — "Qual a política para carga danificada durante transporte?"
**Resposta do assistente:** "A política de danos prevê reembolso integral quando comprovada negligência da transportadora, mediante laudo técnico e fotos."
**Fonte citada:** Nenhuma | **Confiança:** Alta

**Minha avaliação: ❌ INCORRETA — Alucinação com confiança alta**

Justificativa: Esta é a resposta mais crítica do conjunto. Não existe documento formal sobre política de carga danificada em trânsito na base indexada. O Anexo A, na seção "Gaps identificados", confirma explicitamente: *"O FAQ Item 38 descreve um processo, mas não existe documento formal (POL ou PROC) sobre tratamento de carga danificada em trânsito."*

O assistente não apenas inventou informação — ele fez isso com **confiança Alta** e sem citar fonte. Este é o padrão mais perigoso de falha: o atendente não tem sinal de alerta e pode repassar informação incorreta ao cliente, criando expectativa de reembolso que pode não se confirmar.

Adicionalmente: mesmo o FAQ-Atendimento Item 38 (única fonte existente) menciona que o processo "passa pelo Jurídico" e tem e-mail específico (sinistros@novatech.com.br) — nada disso aparece na resposta gerada.

**Tipo de erro:** Alucinação — o assistente gerou informação plausível mas sem respaldo documental, com confiança incorretamente alta.

**Proposta de ajuste:**
- **Pipeline:** O campo `source_document` vazio deveria bloquear a resposta (guardrail G-D01 + VC-02 do requirements.md). O harness de código (response-validator) deve rejeitar respostas sem fonte.
- **Prompt:** Reforçar instrução: "Se não houver chunk com informação diretamente relevante, declare ausência — nunca gere informação plausível não documentada."
- **Interface:** O aviso de confiança alta com fonte ausente é contraditório — o sistema deve calcular confiança com base nos chunks, não deixar o modelo auto-avaliar.

---

### Resposta #5 — "Qual o SLA do cliente Enterprise?"
**Resposta do assistente:** "O tier Enterprise não foi encontrado na documentação. Os tiers documentados são Gold, Silver e Standard. Sugiro confirmar a classificação ou escalar ao supervisor."
**Fonte citada:** — | **Confiança:** Baixa

**Minha avaliação: ✅ Correta**

Justificativa: O guardrail G-N03 ("nunca inventar tiers") foi respeitado. A resposta declara ausência, lista os tiers válidos e orienta o atendente — exatamente o comportamento definido em QUANDO EM DÚVIDA. A confiança Baixa é apropriada. Único detalhe: o assistente não citou fonte para a lista de tiers (SLA-2024, seção 1) — poderia ser mais preciso.

**Classificação:** Correta (funcionamento do guardrail G-N03 confirmado)

---

### Resposta #6 — "Posso enviar carga perigosa com frete expresso?"
**Resposta do assistente:** "Sim, cargas perigosas podem ser enviadas via frete expresso mediante autorização prévia do compliance e documentação ANTT atualizada."
**Fonte citada:** FAQ-Atendimento, item 32 | **Confiança:** Alta

**Minha avaliação: ⚠️ PROBLEMÁTICA — Fonte não confiável para tema de risco**

Justificativa: A informação vem exclusivamente do FAQ-Atendimento (item 32), que o Anexo A classifica explicitamente como "documento informal — NÃO validado por Compliance ou Operações". Para um tema de segurança regulatória (carga perigosa + ANTT), usar uma fonte informal como base de resposta com confiança Alta é um risco inaceitável.

Problemas específicos:
1. Não existe PROC ou POL formal que defina o processo de "autorização do Compliance para frete expresso com carga perigosa" — o FAQ menciona isso como prática informal
2. O guardrail G-N04 ("nunca usar FAQ informal como fonte primária para informações críticas") foi violado
3. A confiança Alta é incorreta — deveria ser Baixa ou com aviso de fonte informal

**Tipo de erro:** Fonte não confiável — a informação pode ser verdadeira na prática, mas não tem respaldo documental formal. Para temas de carga perigosa (risco regulatório e de segurança), isso é inaceitável.

**Proposta de ajuste:**
- **Pipeline:** Metadado `source_type: faq_informal` nos chunks do FAQ deve acionar aviso automático na resposta (guardrail G-N04 já prevê isso)
- **Interface:** Badge de "Fonte informal — confirme com Compliance" quando `source_type: faq_informal`
- **Prompt:** Para perguntas que cruzam carga perigosa + qualquer outra dimensão (frete expresso, prazo, autorização), forçar consulta a documentos formais (POL, PROC) antes do FAQ

---

## Parte 2 — Avaliação com o Claude (segundo avaliador)

### Prompt enviado ao Claude

```
Você é avaliador de produto de um assistente de IA para atendimento de logística.
Avalie as 6 respostas abaixo usando o Anexo A como fonte de verdade.
Para cada resposta, diga: correta, parcialmente correta, ou incorreta — com justificativa.
Identifique especialmente alucinações, fontes não confiáveis, e violações de guardrails.

[Tabela das 6 respostas + conteúdo do Anexo A colados]
```

### Output do Claude (pontos principais)

O Claude concordou com minha avaliação nas respostas #1, #2, #3, #5.

**Divergências relevantes:**

**Resposta #4:** O Claude identificou corretamente a alucinação, mas foi além — apontou que o FAQ Item 38 existe e descreve um processo similar, o que poderia ter "contaminado" o modelo se o chunk foi recuperado. Insight adicional: sugere verificar se o chunk do FAQ foi ou não incluído no contexto da query de staging.

**Resposta #6:** O Claude classificou como "parcialmente correta" — argumentou que a informação do FAQ pode estar factualmente correta na prática operacional. Discordo desta classificação. Para informação sobre carga perigosa (tema de segurança e regulatório), "pode estar correta" não é critério aceitável. A ausência de documento formal é disqualificante independente da acurácia histórica da informação.

**Resposta #1:** O Claude não apontou a imprecisão da seção citada (3.2 vs 3.1). Eu identifiquei isso na minha análise — demonstra que revisão humana complementa a IA mesmo em respostas corretas.

### Comparação consolidada

| Resposta | Minha avaliação | Claude | Concordância |
|----------|----------------|--------|-------------|
| #1 | ✅ Correta (seção imprecisa) | ✅ Correta | Parcial — só eu identifiquei imprecisão da seção |
| #2 | ✅ Correta | ✅ Correta | Total |
| #3 | ✅ Correta | ✅ Correta | Total |
| #4 | ❌ Alucinação | ❌ Alucinação | Total — Claude adicionou detalhe sobre chunk FAQ |
| #5 | ✅ Correta | ✅ Correta | Total |
| #6 | ⚠️ Fonte não confiável — reprovada | ⚠️ Parcialmente correta | **Divergência** — mantenho reprovação |

**Minha posição sobre a divergência na #6:** Mantenho minha avaliação. Para qualquer informação relacionada a carga perigosa, o padrão de aceitação deve ser mais alto, não mais tolerante. O argumento "pode estar correto na prática" não é critério de produto — é exatamente o tipo de raciocínio que os guardrails foram criados para prevenir.

---

## Parte 3 — Propostas de Ajuste Consolidadas

| Resposta | Tipo de erro | Ajuste de produto |
|----------|-------------|-------------------|
| #4 | Alucinação + confiança alta sem fonte | (1) `response-validator.ts` deve bloquear respostas sem `source_document`. (2) Confiança calculada por score de retrieval, não auto-avaliada pelo modelo. |
| #6 | Fonte informal para tema crítico | (1) Metadado `source_type: faq_informal` ativa aviso automático. (2) Badge visual na interface. (3) Perguntas sobre carga perigosa devem priorizar fontes formais no retrieval. |

**Parecer de go-live:** As respostas #4 e #6 indicam dois riscos ativos:
- A ausência de harness de código que bloqueie resposta sem fonte (gap identificado no cenário 2, ainda não implementado)
- O FAQ informal sendo indexado sem controle de confiabilidade

Ambos têm mitigação definida nos guardrails do Cenário 2. O go-live **não deve ocorrer** até que G-D01 (citação de fonte obrigatória com bloqueio no código) e G-N04 (metadado de fonte informal) estejam implementados e validados.
