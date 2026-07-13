# Exercício 2.3 — Seção "Product Rules & Guardrails" do AGENTS.md — NovaTech Assistant

> Constitution do projeto. Todo agente de IA (Copilot, Claude Code) lê este arquivo antes de gerar qualquer artefato.
> As seções abaixo são preenchidas por papéis diferentes nos exercícios do Cenário 2.

## Project Overview
<!-- TODO (Tech Lead — Ex. 2.1) -->

## Tech Stack & Architecture
<!-- TODO (Tech Lead — Ex. 2.1): inclui regras de gerenciamento de contexto da ADR-0002 -->

## Coding Standards (Tech Lead)
<!-- TODO (Tech Lead — Ex. 2.1) -->

## Product Rules & Guardrails (Product Specialist)

> Fonte completa e rastreabilidade aos incidentes: `docs/guardrails.md` (Ex. 2.2). Esta seção é a versão consumível por agentes — regras prescritivas, glossário e restrições de schema. Qualquer agente que gerar prompts, respostas simuladas, testes de conteúdo ou código do `response-validator`/`response-builder` deve seguir o que está aqui.

### 1. Regras de comportamento do assistente

Cada regra tem um ID (rastreável a `docs/guardrails.md`) e uma tag de enforcement: `[PROMPT]` (instrução ao modelo, probabilística), `[CÓDIGO]` (validação determinística obrigatória em `src/services/response-validator.ts` ou `src/functions/query/response-builder.ts`), `[HÍBRIDO]` (as duas).

**MUST (DEVE):**
- `D-01` `[HÍBRIDO]` Toda resposta com informação normativa DEVE citar documento + seção (ex: `POL-001 § 3.2`) e preencher o campo `source_document` no JSON de retorno, mesmo com confiança baixa.
- `D-02` `[HÍBRIDO]` Quando existirem múltiplas versões de um documento, a resposta DEVE usar os valores da versão vigente (por metadado de vigência/data) e DEVE informar que existe versão anterior.
- `D-03` `[CÓDIGO]` Toda pergunta que combine "carga perigosa" (classes 1-6 ANTT) com devolução/frete expresso/benefício padrão DEVE receber resposta que reflita a exceção de `POL-001 § 3.2`. Bloquear/reescrever se a resposta gerada contradisser essa regra.
- `D-04` `[CÓDIGO]` A resposta "não encontrado" só pode ser emitida quando a busca no índice retornar score abaixo do threshold configurado para os documentos relevantes ao tema.
- `D-05` `[PROMPT]` Responder sempre em português formal, tom institucional de atendimento ao cliente.
- `D-06` `[HÍBRIDO]` Usar somente os tiers Gold, Silver, Standard; corrigir o usuário se citar um tier inexistente (ex: "Platinum"). Campo `customer_tier`, se presente na resposta, DEVE ser um dos 3 valores do enum.

**MUST NOT (NÃO DEVE):**
- `N-01` `[CÓDIGO]` NUNCA gerar valor numérico (prazo, multiplicador, percentual, SLA) que não esteja literalmente presente nos chunks recuperados na chamada atual.
- `N-02` `[CÓDIGO]` NUNCA afirmar que carga perigosa pode ser devolvida pelo processo padrão. Regra de maior severidade do domínio — travada no código, não apenas no prompt.
- `N-03` `[HÍBRIDO]` NUNCA inventar tier de cliente além de Gold/Silver/Standard.
- `N-04` `[CÓDIGO]` NUNCA declarar "não encontrado" sem antes ter consultado o índice e confirmado score de recuperação baixo.
- `N-05` `[PROMPT]` NUNCA misturar valores da PROC-042 v1 e v2 na mesma resposta sem indicar qual versão está sendo citada em cada trecho.

**WHEN IN DOUBT (QUANDO EM DÚVIDA):**
- `Q-01` `[PROMPT]` Pergunta sobre carga perigosa + devolução/frete expresso/exceção → nunca responder binário; indicar tratamento especial e orientar contato com Gestão de Riscos (ramal 4500).
- `Q-02` `[HÍBRIDO]` Documentos conflitantes recuperados → priorizar o mais recente por metadado de vigência e informar explicitamente a existência da versão anterior.
- `Q-03` `[HÍBRIDO]` Confiança de recuperação baixa → prefixar resposta com aviso de baixa confiança e sugerir escalonamento (supervisor, Comercial, Jurídico ou Gestão de Riscos, conforme o tema).
- `Q-04` `[PROMPT]` Pergunta cruza duas categorias de domínio (~15% dos casos) → decompor a resposta por categoria, citando fonte de cada parte separadamente.

### 2. Glossário de linguagem ubíqua

Termos que um agente de IA (ou um Dev usando Copilot) pode usar incorretamente sem esta definição. Extraídos de `docs/novatech/`.

| Termo | Definição vinculante | Fonte |
|-------|----------------------|-------|
| **Cliente Gold / Silver / Standard** | Únicos 3 tiers existentes. Gold = contrato anual > R$ 500.000 OU > 200 operações/mês. Silver = entre R$ 100.000–500.000 OU 50–200 operações/mês. Standard = demais. Não existe tier "Platinum". | `SLA-2024-tabela-sla-clientes.md § 1` |
| **Carga perigosa** | Classes 1 a 6 da ANTT (Resolução 5.947/2021): explosivos, gases, líquidos inflamáveis, sólidos inflamáveis, oxidantes/peróxidos, tóxicos/infectantes. NÃO elegível para devolução pelo processo padrão. | `POL-001-politica-devolucao.md § 3.2` |
| **Frete especial** | Frete de carga acima de 500kg. Calculado como `Valor base × Multiplicador regional × Fator de peso`. | `PROC-042-v2-frete-especial-revisado.md § 2` |
| **Multiplicador regional** | Fator por região de destino aplicado ao frete especial. Valores da v2 (vigente): Sul 1.3, Sudeste 1.1, Centro-Oeste 1.4, Nordeste 1.5, Norte 1.8. A v1 (desatualizada) tem valores diferentes — nunca usar sem checar vigência. | `PROC-042-v2-frete-especial-revisado.md § 2.1` |
| **SLA de resposta** | Tempo até o primeiro retorno ao cliente (mesmo que seja "estamos verificando"). Distinto de SLA de resolução. | `SLA-2024-tabela-sla-clientes.md § 2` |
| **SLA de resolução** | Tempo até o problema ser efetivamente resolvido. Gold: 24h úteis (geral) / 4h (incidente crítico). Silver: 48h / 8h. Standard: 72h / 24h. | `SLA-2024-tabela-sla-clientes.md § 2` |
| **Incidente crítico** | Carga com valor declarado > R$ 100.000 com status desconhecido há > 6h; carga perigosa com irregularidade documental/rastreamento; > 5 chamados do mesmo cliente em 24h sobre o mesmo problema; risco à segurança de pessoas. | `SLA-2024-tabela-sla-clientes.md § 3` |
| **Vigência de documento** | Metadado que indica se um documento é a versão normativa atual. Documentos obsoletos são marcados, não excluídos do índice — o assistente prioriza a versão vigente mas pode citar a anterior quando relevante. | ADR-0003 (Cenário 1) |
| **CT-e** | Conhecimento de Transporte Eletrônico — identificador obrigatório em qualquer chamado de devolução. | `POL-001-politica-devolucao.md § 3.3` |

### 3. Restrições que impactam geração de código

- Todo objeto de resposta retornado pelo `response-builder.ts` DEVE incluir o campo `source_document` (string não vazia), mesmo em respostas de baixa confiança — ver D-01.
- O tipo/schema de resposta (`src/shared/types.ts`) DEVE restringir `customer_tier` a um enum de 3 valores (`"Gold" | "Silver" | "Standard"`) — ver D-06/N-03.
- `response-validator.ts` DEVE implementar como regra determinística (não apenas texto de prompt): bloqueio de "carga perigosa + devolução padrão" (N-02) e checagem de que todo número na resposta existe literalmente nos chunks recuperados (N-01).
- A emissão da resposta padrão de "não encontrado" DEVE ser condicionada a um score de recuperação abaixo de threshold configurável — não pode ser decisão livre do modelo (D-04/N-04).
- O `prompt-builder.ts` DEVE ordenar/priorizar chunks pelo metadado de vigência (ADR-0003) antes de montar o prompt (D-02/Q-02).

### 4. Referências no repositório

- Guardrails completos + rastreabilidade aos incidentes: `docs/guardrails.md`
- Requirements do query endpoint (bounded contexts, scope boundaries): `specs/query-endpoint/requirements.md`
- System prompt versionado: `prompts/system-prompt.md` (mudanças registradas em `prompts/prompt-changelog.md`)
- Documentação de negócio (fonte de verdade dos termos acima): `docs/novatech/`
- ADRs referenciadas (contexto/vigência): `docs/adr/`

## Testing Standards (QA)
<!-- TODO (QA — Ex. 2.1) -->

## Project Management Rules (Delivery Manager)
<!-- TODO (Delivery Manager — Ex. 2.3) -->

## Build & Deploy
<!-- TODO (Tech Lead — Ex. 2.1) -->
