# Exercício 2.2 — Guardrails Formalizados como Artefato de Produto
**Papel:** Product Specialist | **Cenário:** 2 | **Programa:** Trilha AI First — DGS/DB1

---

> Este documento é o artefato a ser salvo em `specs/query-endpoint/guardrails.md` no repositório.
> Ele complementa o `requirements.md` e é referenciado pelo `AGENTS.md`.

---

# Guardrails do Assistente NovaTech
**Versão:** 1.0
**Autor:** Product Specialist
**Última atualização:** Junho/2026
**Rastreabilidade:** Cada guardrail está conectado a ao menos um incidente real registrado durante testes internos.

---

## DEVE — Comportamentos Obrigatórios

### G-D01 — Citar fonte em toda resposta

**Descrição:** Toda resposta gerada pelo assistente DEVE incluir o identificador do documento fonte e a seção específica de onde a informação foi extraída.

**Formato esperado:** "Conforme [NOME-DOC], [seção X.X], atualizado em [data]."

**Classificação de enforcement:**
- **Prompt (probabilístico):** Instrução no system prompt para sempre citar fonte
- **Código (determinístico):** Validação no `response-builder.ts` — se `source_document` estiver ausente ou vazio, a resposta não é retornada ao cliente; retorna erro interno e aciona fallback

**Justificativa da dupla camada:** O prompt orienta o comportamento do LLM, mas um modelo pode ignorar a instrução em contextos longos (context rot). O código garante que a interface nunca exiba uma resposta sem fonte, independente do comportamento do LLM.

**Incidente que previne:** Incidente 2 — O assistente citou "PROC-042, seção 2" mas os valores informados eram da versão desatualizada. Este guardrail, combinado com G-D03, força a citação da versão específica do documento.

---

### G-D02 — Responder em português formal

**Descrição:** O assistente DEVE responder em português formal e acessível, independente do idioma em que a pergunta for feita.

**Classificação de enforcement:**
- **Prompt (probabilístico):** Instrução no system prompt com exemplo de tom esperado
- **Código:** Não implementado nesta versão — a validação de idioma por código (ex: biblioteca de detecção de língua) é considerada over-engineering para o risco atual. Monitorar métricas de qualidade para reavaliar.

**Incidente que previne:** Não deriva de um incidente específico, mas de guardrail original do cenário 1. Risco: atendentes confusos por respostas em inglês (idioma do modelo base).

---

### G-D03 — Incluir data/versão ao citar documento com múltiplas versões

**Descrição:** Quando a fonte for um documento que possui mais de uma versão no índice (ex: PROC-042 v1 e v2), a resposta DEVE indicar explicitamente qual versão está sendo citada e sua data de emissão.

**Classificação de enforcement:**
- **Prompt (probabilístico):** Instrução específica: "Quando identificar chunks de versões diferentes do mesmo documento, DEVE apresentar ambas as versões com data, nunca escolher uma arbitrariamente."
- **Código (determinístico):** O metadado `document_version` e `emissao_date` devem ser incluídos no chunk context enviado ao LLM. O `prompt-builder.ts` é responsável por formatar esse metadado de forma que o modelo o veja antes de gerar a resposta.

**Incidente que previne:** Incidente 2 — O assistente usou multiplicadores da PROC-042 v1 (desatualizada) citando apenas "PROC-042, seção 2", sem diferenciar a versão. O atendente não tinha como saber que a informação era antiga.

---

### G-D04 — Declarar ausência de resposta com orientação de próximo passo

**Descrição:** Quando o assistente não encontrar informação com confiança suficiente (score < threshold), DEVE declarar explicitamente que não encontrou e DEVE sugerir um próximo passo concreto (escalar para supervisor, consultar documento específico, ou ligar para ramal).

**Formato esperado:** "Não encontrei informação sobre [tema] na documentação disponível. Recomendo [ação específica]."

**O que não é aceitável:** Silêncio, "não sei", ou resposta vaga sem orientação.

**Classificação de enforcement:**
- **Prompt (probabilístico):** Instrução com exemplos de mensagens de fallback aceitáveis
- **Código (determinístico):** Threshold de confiança no `search.ts` — quando todos os chunks retornados têm score < 0.75, o sistema injeta template de mensagem de fallback no contexto do LLM, forçando o formato esperado

**Incidente que previne:** Incidente 3 — O assistente disse "Não encontrei informação sobre isso" para uma pergunta sobre SLA Gold, quando a informação estava indexada. Este guardrail trata o lado oposto: quando de fato não há informação, garante que a mensagem seja útil e não apenas uma recusa.

---

## NÃO DEVE — Comportamentos Proibidos

### G-N01 — Nunca gerar valores numéricos não documentados

**Descrição:** O assistente NÃO DEVE gerar prazos (em dias ou horas), multiplicadores de frete, percentuais, valores monetários ou qualquer dado numérico que não esteja literalmente presente nos chunks recuperados do índice.

**Exemplos de violação:**
- Inventar "prazo de devolução de 5 dias" quando o chunk diz 7 dias
- Calcular uma média entre multiplicadores da v1 e v2 (ex: "o multiplicador para o Norte é aproximadamente 1.7")
- Citar SLA de 12h para cliente Platinum (tier inexistente)

**Classificação de enforcement:**
- **Prompt (probabilístico):** Instrução explícita com exemplos de violação
- **Código (determinístico):** O harness de validação no `response-validator.ts` extrai valores numéricos da resposta gerada e verifica se cada um aparece em ao menos um chunk do contexto. Respostas com valores numéricos sem correspondência nos chunks são rejeitadas e substituídas por fallback.

**Incidente que previne:** Incidente 2 (multiplicadores errados) e Incidente 1 (prazo de 7 dias aplicado a cargas perigosas que não têm prazo de devolução padrão).

---

### G-N02 — Nunca afirmar que carga perigosa pode ser devolvida pelo processo padrão

**Descrição:** O assistente NÃO DEVE informar que cargas classificadas como perigosas (classes 1 a 6 da ANTT, conforme POL-001 seção 3.2) podem ser devolvidas pelo processo padrão (Portal do Cliente, prazo de 7 dias).

**A resposta correta para este caso:** Redirecionar ao setor de Gestão de Riscos (ramal 4500) para tratamento individual.

**Classificação de enforcement:**
- **Prompt (probabilístico):** Instrução específica com a regra e o redirecionamento exato
- **Código (determinístico):** Detecção de keywords de carga perigosa na query de entrada (classes ANTT, "explosivo", "inflamável", "tóxico", etc.) combinada com detecção de keywords de devolução. Quando ambas estiverem presentes, injeta template de redirecionamento para o ramal 4500 **antes** de chamar o LLM — não depende do modelo seguir a instrução.

**Classificação como determinístico:** Este é o guardrail de maior risco do sistema. Uma resposta incorreta pode gerar consequências legais (regulação ANTT) e de segurança. Por isso, o enforcement é **majoritariamente no código**, não no prompt.

**Incidente que previne:** Incidente 1 — O assistente respondeu que o prazo de devolução para carga perigosa é 7 dias, quando na verdade cargas perigosas NÃO podem ser devolvidas pelo processo padrão.

---

### G-N03 — Nunca inventar tiers de cliente

**Descrição:** O assistente NÃO DEVE citar SLAs, benefícios ou características para tiers de cliente que não existem. Os únicos tiers válidos são: Gold, Silver e Standard (SLA-2024, seção 1).

**Exemplos de violação:**
- "Clientes Platinum têm resposta em 1h" (tier inexistente)
- "Clientes Diamond têm gerente dedicado" (tier inexistente)

**A resposta correta para este caso:** "O tier [X] não existe na NovaTech. Os tiers disponíveis são Gold, Silver e Standard. Por favor, confirme o tier do cliente pelo número do contrato."

**Classificação de enforcement:**
- **Prompt (probabilístico):** Instrução com lista explícita dos tiers válidos
- **Código (determinístico):** Detecção de tier inválido na query de entrada → injeção de template de resposta antes de chamar o LLM

**Incidente que previne:** Cenário identificado no Chunk FAQ-15 e no Cenário 1: clientes confundem "Platinum" com tier NovaTech. Se o assistente alucinasse SLAs para um tier inexistente, criaria expectativa incorreta no cliente.

---

### G-N04 — Nunca usar FAQ informal como fonte primária para informações críticas

**Descrição:** O assistente NÃO DEVE apresentar informações extraídas exclusivamente do FAQ-Atendimento (documento informal, não validado) com o mesmo nível de confiança de documentos normativos (POL, PROC, SLA).

**Classificação de enforcement:**
- **Código (determinístico):** Metadado `source_type: faq_informal` nos chunks do FAQ. O `response-builder.ts` verifica se todos os chunks são de fontes informais — se sim, inclui aviso automático: "Esta informação provém de documentação informal e não foi validada por Compliance."
- **Prompt (probabilístico):** Instrução para tratar chunks com `source_type: faq_informal` com cautela

**Incidente que previne:** Risco identificado no Cenário 1 (exercício 1.1): itens do FAQ como procedimento de carga danificada e seguro de carga não têm respaldo em documentos formais, mas são apresentados com confiança pelo FAQ.

---

## QUANDO EM DÚVIDA — Comportamentos de Fallback

### G-Q01 — Preferir a versão mais recente em caso de conflito entre versões

**Descrição:** Quando o contexto contiver chunks de versões diferentes do mesmo documento com valores conflitantes, o assistente DEVE apresentar ambas as versões, indicar que a mais recente deve prevalecer para novos chamados, e recomendar confirmação com supervisor para chamados em andamento.

**Classificação de enforcement:**
- **Prompt (probabilístico):** Instrução com exemplo de formato esperado
- **Código (complementar):** O `prompt-builder.ts` ordena os chunks com a versão mais recente em primeiro lugar no contexto (mitigando o efeito *lost in the middle*)

**Conectado a:** ADR-0003 (tratamento de documentos contraditórios) e G-D03.

---

### G-Q02 — Prefixar com aviso quando confiança é baixa

**Descrição:** Quando a resposta for gerada com score de confiança abaixo do threshold (0.75), DEVE iniciar com aviso padronizado antes do conteúdo da resposta.

**Formato do aviso:** "[⚠️ Confiança baixa — confirme com supervisor] " seguido da resposta.

**Classificação de enforcement:**
- **Código (determinístico):** O `response-builder.ts` verifica o campo `confidence_level` e injeta o prefixo automaticamente quando abaixo do threshold. O LLM não controla se o aviso aparece ou não.

---

### G-Q03 — Sugerir escalação quando a pergunta cruza múltiplos domínios críticos

**Descrição:** Quando a pergunta envolver simultaneamente carga perigosa + frete especial + devolução (pergunta multi-domínio com alto risco de erro), o assistente DEVE sugerir confirmação com supervisor mesmo que tenha encontrado informação com confiança alta.

**Classificação de enforcement:**
- **Prompt (probabilístico):** Instrução com exemplos de combinações que ativam a sugestão de escalação

---

## Sumário de Enforcement

| Guardrail | Enforcement via Prompt | Enforcement via Código | Nível de risco |
|-----------|----------------------|----------------------|----------------|
| G-D01 Citar fonte | ✓ | ✓ (validação `source_document`) | Alto |
| G-D02 Português formal | ✓ | ✗ | Baixo |
| G-D03 Versão ao citar doc | ✓ | ✓ (metadado no chunk) | Alto |
| G-D04 Fallback com orientação | ✓ | ✓ (template quando score baixo) | Médio |
| G-N01 Sem valores inventados | ✓ | ✓ (harness numérico) | Alto |
| G-N02 Sem devolução de carga perigosa | ✓ | ✓ (detecção de keywords, pré-LLM) | **Crítico** |
| G-N03 Sem tiers inexistentes | ✓ | ✓ (detecção na query) | Alto |
| G-N04 FAQ como fonte secundária | ✓ | ✓ (metadado `source_type`) | Médio |
| G-Q01 Preferir versão mais recente | ✓ | ✓ (ordenação de chunks) | Alto |
| G-Q02 Prefixo de baixa confiança | ✓ | ✓ (injetado pelo response-builder) | Médio |
| G-Q03 Escalação multi-domínio | ✓ | ✗ | Médio |

**Princípio:** Guardrails de risco **Crítico** e **Alto** têm enforcement obrigatório no código. Prompts são probabilísticos — um modelo pode ignorar uma instrução, especialmente em contextos longos. Código é determinístico — o comportamento é garantido independente do modelo.
