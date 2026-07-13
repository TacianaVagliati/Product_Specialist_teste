# Exercício 3.2 — Harness de Produto para Melhoria Contínua
**Papel:** Product Specialist  
**Cenário:** 3  
**Programa:** Trilha AI First — DGS/DB1

---

> Este documento é o artefato a ser salvo em `specs/query-endpoint/harness-produto.md`.
> Define como o assistente evolui após o go-live sem degradar qualidade nem violar guardrails.

---

# Harness de Produto — NovaTech Assistant
**Versão:** 1.0  
**Autor:** Product Specialist 
**Referências:** `specs/query-endpoint/guardrails.md`, `specs/query-endpoint/requirements.md`, ADR-0002, ADR-0003

---

## 1. Processo de Feedback — Do Atendente à Melhoria

### 1.1 Captura do feedback

O atendente aciona o botão "Resposta incorreta" na interface do Teams (fluxo de feedback definido no Cenário 1, implementado no módulo `feedback-api`). O mini-formulário coleta:
- Motivo: `[ ] Informação errada` `[ ] Informação desatualizada` `[ ] Incompleta` `[ ] Contraditória`
- Fonte correta (opcional): campo livre
- ID do chamado (automático)

Cada feedback é registrado com: query original, resposta exibida, motivo, fonte sugerida, timestamp, ID do atendente.

### 1.2 Triagem e classificação

**Frequência:** Revisão semanal pelo Product Specialist, com suporte do sistema de alertas (ver seção 1.4).

**Classificação de cada feedback:**

| Tipo | Critério | Ação correspondente |
|------|----------|---------------------|
| **Gap de documentação** | Pergunta frequente sem cobertura documental | Fila para Compliance/Operações formalizarem o documento |
| **Documento desatualizado** | Informação correta foi substituída por versão mais recente | Pipeline re-indexa o documento atualizado (SLA: até 24h) |
| **Documento contraditório** | Duas versões coexistem sem hierarquia clara | Escalação para responsável do documento + ADR se necessário |
| **Falha de guardrail** | Guardrail foi violado (ex: carga perigosa com informação errada) | Correção urgente — revisão do prompt e/ou código antes de qualquer outra mudança |
| **Chunk incorreto recuperado** | Retrieval buscou documento não relevante | Ajuste de metadados, estratégia de chunking, ou query rewriting |
| **Falha de prompt** | Comportamento correto mas resposta mal formulada | Ajuste de prompt com regression testing obrigatório |

### 1.3 Ciclo de correção por tipo

**Para gap de documentação:**
1. PS documenta a pergunta e a resposta esperada como novo golden query em `/prompts/eval/golden-queries.json`
2. Compliance/Operações da NovaTech formalizam o documento
3. Pipeline ingere o novo documento (processo automatizado, SLA 24h)
4. PS valida que a golden query agora retorna resposta correta
5. Atendente que reportou recebe notificação

**Para documento desatualizado:**
1. Área responsável atualiza o documento no SharePoint
2. Pipeline re-indexa automaticamente
3. Versão antiga marcada como `obsoleto: true` (ADR-0003)
4. PS executa regression test na golden query correspondente

**Para falha de guardrail:**
1. Incidente registrado como urgente
2. PS e Tech Lead revisam o guardrail violado em `specs/query-endpoint/guardrails.md`
3. Se falha de prompt: ajuste de prompt → regression testing → aprovação HITL → deploy
4. Se falha de código: hotfix em `response-validator.ts` → code review → deploy emergencial
5. Incident report documentado em `/docs/runbooks/`

**Para ajuste de prompt:**
1. PS propõe nova versão do prompt em `/prompts/system-prompt.md`
2. Regression testing obrigatório (ver seção 2)
3. Se aprovado: aprovação HITL (ver seção 3)
4. Registro em `/prompts/prompt-changelog.md` com: data, autor, motivo, golden queries afetadas

### 1.4 Alertas automáticos

Feedbacks que ativam notificação imediata ao PS (não esperam revisão semanal):

| Trigger | Threshold | Notificação para |
|---------|-----------|-----------------|
| Feedbacks sobre carga perigosa | Qualquer feedback com palavra-chave ANTT/classe/perigosa | PS + Tech Lead (risco regulatório) |
| Feedbacks repetidos do mesmo tema | >3 feedbacks no mesmo tema em 48h | PS |
| Resposta sem fonte retornada ao atendente | Qualquer ocorrência | PS + Dev (indica falha de harness) |
| Taxa de feedback negativo | >15% das queries em 24h | PS + Delivery Manager |

---

## 2. Regression Testing de Produto

### 2.1 Por que regression testing em IA é diferente

Em sistemas determinísticos, uma mudança em A não afeta B se A e B são independentes. Em sistemas de IA, uma mudança no system prompt pode alterar o comportamento do modelo em dimensões não relacionadas. Uma nova regra adicionada pode fazer o modelo "esquecer" uma regra anterior (context rot). Um novo documento indexado pode criar ambiguidade que afeta queries antes corretas.

**Princípio:** Antes de qualquer mudança em produção (prompt, documentos, pipeline), rodar o conjunto de golden queries e verificar que as respostas não pioraram.

### 2.2 Golden Query Set

Mantido em `/prompts/eval/golden-queries.json`. Composto por:

**Queries de cobertura normal (caminho feliz):**
- Prazo de devolução padrão → resposta correta + fonte POL-001
- SLA Gold resolução → 24h + fonte SLA-2024
- SLA Silver resolução → 48h + fonte SLA-2024
- Multiplicador Norte frete especial 600kg → 1.8 + fonte PROC-042-v2

**Queries de guardrail (invariantes — NUNCA devem regredir):**
- Carga perigosa + devolução → negativa + ramal 4500 (G-N02)
- Tier Platinum/Enterprise → negativa + lista de tiers válidos (G-N03)
- Pergunta sem cobertura → declaração de ausência + sugestão (G-D04)
- Documentos contraditórios (PROC-042 v1 vs v2) → ambas versões com data (G-D03 + ADR-0003)

**Queries de edge case:**
- Pergunta multi-domínio (carga perigosa + frete especial + devolução)
- Pergunta com tier inválido embutido em contexto mais longo
- Pergunta em inglês → resposta em português (G-D02)

### 2.3 Critérios de aprovação do regression test

Para cada golden query, a resposta pós-mudança deve atender:

| Dimensão | Critério mínimo |
|----------|----------------|
| Guardrails de bloqueio | 100% — nenhum guardrail G-N01, G-N02, G-N03 pode regredir |
| Factual accuracy | ≥ 95% das queries de cobertura com resposta factualmente correta |
| `source_document` presente | 100% das respostas com fonte identificada (VC-02) |
| Confiança calibrada | Score ≥ 0.75 para queries com cobertura documental |
| Formato português formal | 100% das respostas |

**Se qualquer guardrail de bloqueio regredir → mudança bloqueada. Sem exceções.**

Se factual accuracy cair abaixo de 95% → mudança pausada para revisão.

### 2.4 Quando rodar regression testing

| Mudança | Regression test obrigatório |
|---------|---------------------------|
| Qualquer alteração no system prompt | Sim — conjunto completo |
| Adição de novo documento à base | Sim — queries relacionadas ao tema + guardrails |
| Mudança em parâmetros de retrieval (threshold, número de chunks) | Sim — conjunto completo |
| Atualização de documento existente | Sim — queries sobre aquele documento + guardrails |
| Mudança no `response-validator.ts` | Sim — guardrails de bloqueio |
| Mudança no `prompt-builder.ts` | Sim — queries de contexto longo e multi-turno |

---

## 3. Pontos de Human-in-the-Loop (HITL)

### Princípio

Nem toda mudança no assistente precisa de aprovação humana. HITL aumenta a segurança mas desacelera a evolução. O critério de HITL é o risco da mudança, não o tamanho.

### 3.1 HITL obrigatório — aprovação do Product Specialist

| Mudança | Por que HITL | Quem aprova |
|---------|-------------|-------------|
| Qualquer alteração no system prompt | Prompts afetam comportamento em dimensões não testadas; risco de regredir guardrails | Product Specialist |
| Adição de novo tipo de documento à base (ex: manual de um parceiro) | Fontes novas podem criar contradições com documentação existente | Product Specialist |
| Alteração de threshold de confiança | Afeta quando aviso de baixa confiança aparece — impacto direto na experiência do atendente | Product Specialist |
| Modificação de guardrail em `guardrails.md` | Guardrails são invariantes de produto — qualquer mudança é decisão de produto | Product Specialist + Tech Lead |
| Resposta padrão de fallback (mensagem que o atendente vê quando não há resposta) | Texto voltado ao usuário, impacta percepção do produto | Product Specialist |

### 3.2 HITL obrigatório — aprovação do Tech Lead

| Mudança | Por que HITL | Quem aprova |
|---------|-------------|-------------|
| Alteração em `response-validator.ts` (guardrails de código) | Mudança nos bloqueios determinísticos; pode criar buracos de segurança | Tech Lead |
| Mudança em parâmetros de retrieval no `search.ts` | Afeta quais chunks chegam ao modelo — impacto amplo e difícil de prever | Tech Lead |
| Alteração no structured output schema (Zod) | Mudança de contrato entre pipeline e consumidores | Tech Lead |

### 3.3 HITL na interface — respostas de baixa confiança sobre temas críticos

Além dos HITL de mudança, há um HITL de runtime:

**Regra:** Quando a query combinar carga perigosa + qualquer outra dimensão (devolução, frete, prazo) E o score de confiança for < 0.85, a resposta **não é enviada diretamente ao atendente**. Em vez disso, entra em uma fila de revisão humana (supervisor de atendimento) que pode:
- Aprovar e enviar a resposta ao atendente
- Editar e enviar
- Rejeitar e escalar ao setor de Gestão de Riscos

**Justificativa:** Carga perigosa é o domínio de maior risco regulatório e de segurança do projeto. A taxa de 12% de erros em staging, combinada com o incidente 1 (guardrail violado em staging), justifica um HITL de runtime até que o sistema demonstre >99% de precisão neste domínio especificamente.

**SLA da revisão humana:** Supervisor tem até 10 minutos para revisar. Se não houver resposta em 10 minutos, o atendente recebe mensagem padrão de escalação ao ramal 4500.

---

## 4. Preservação dos Guardrails do Cenário 2

Os guardrails formalizados em `specs/query-endpoint/guardrails.md` são **invariantes** — não podem ser removidos ou enfraquecidos sem uma decisão explícita de produto documentada como ADR.

**Mecanismo de preservação:**

1. Os guardrails de bloqueio (G-N01, G-N02, G-N03) estão implementados no `response-validator.ts` (determinístico) — não dependem de prompt
2. O regression testing inclui queries específicas para cada guardrail de bloqueio
3. Qualquer mudança que faça um guardrail de bloqueio falhar no regression test bloqueia o deploy automaticamente
4. O `AGENTS.md` (seção Product Rules & Guardrails) instrui o Copilot a nunca remover ou enfraquecer os guardrails ao gerar código

**O que nunca pode regredir independente de qualquer melhoria:**
- G-N02: Carga perigosa nunca recebe informação de devolução pelo processo padrão
- G-N03: Tiers inexistentes nunca recebem SLAs inventados
- G-D01: Toda resposta tem `source_document` — se campo ausente, resposta é bloqueada
