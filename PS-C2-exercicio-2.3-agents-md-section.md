# Exercício 2.3 — Seção "Product Rules & Guardrails" do AGENTS.md
**Papel:** Product Specialist | **Cenário:** 2 | **Programa:** Trilha AI First — DGS/DB1

---

> O bloco abaixo é o conteúdo a ser inserido em `AGENTS.md` na seção
> `## Product Rules & Guardrails (Product Specialist)`.
> Está escrito para ser lido por agentes de IA (Copilot, Claude Code) — prescritivo, não narrativo.

---

```markdown
## Product Rules & Guardrails (Product Specialist)

> Lido por: Copilot, Claude Code, qualquer agente gerando código ou artefatos deste projeto.
> Fonte de verdade: specs em `/specs/query-endpoint/`, guardrails em
> `/specs/query-endpoint/guardrails.md`, documentação de domínio em `/docs/novatech/`.

---

### Glossário de Domínio

Termos usados neste projeto com significado específico. NUNCA use estes termos com outro
significado. Se um termo não está aqui, pergunte antes de assumir.

| Termo | Definição canônica |
|-------|--------------------|
| `carga perigosa` | Carga classificada nas classes 1–6 da ANTT (Resolução nº 5.947/2021). Inclui: explosivos (1), gases (2), líquidos inflamáveis (3), sólidos inflamáveis (4), oxidantes/peróxidos (5), tóxicos/infectantes (6). NUNCA interpretar como "carga de alto valor" ou "carga frágil". |
| `frete especial` | Modalidade de frete para cargas ACIMA de 500kg. Fórmula: Valor base × Multiplicador regional × Fator de peso. NUNCA confundir com "frete expresso" ou "frete prioritário". |
| `cliente Gold` | Tier com contrato anual > R$ 500.000 OU > 200 operações/mês. SLA: resposta ≤ 2h úteis, resolução ≤ 24h úteis. |
| `cliente Silver` | Tier com contrato anual R$ 100k–500k OU 50–200 operações/mês. SLA: resposta ≤ 4h úteis, resolução ≤ 48h úteis. |
| `cliente Standard` | Todos os demais clientes. SLA: resposta ≤ 8h úteis, resolução ≤ 72h úteis. |
| `tiers válidos` | APENAS: Gold, Silver, Standard. Platinum, Diamond, Bronze ou qualquer outro NÃO EXISTE. |
| `multiplicador regional` | Fator numérico do frete especial por região. Versão vigente (PROC-042-v2, nov/2023): Sul=1.3, Sudeste=1.1, CO=1.4, NE=1.5, Norte=1.8. |
| `fator de peso` | Multiplicador por faixa de peso no frete especial. Vigente: 1.0 (500–1.000kg), 1.15 (1.001–3.000kg), 1.4 (>3.000kg). |
| `incidente crítico` | Chamado com ao menos 1 dos critérios: carga >R$100k status desconhecido >6h; carga perigosa com irregularidade; >5 chamados mesmo cliente/24h mesmo problema; risco à segurança de pessoas. |
| `source_document` | Campo obrigatório no JSON de retorno. Formato: `{ "id": "PROC-042-v2", "section": "2.1", "version": "2.0", "date": "2023-11-10" }`. |
| `confidence_level` | Score numérico de similaridade do retrieval (0.0–1.0). Threshold padrão: 0.75. Abaixo → aviso obrigatório. |
| `chunk` | Trecho de documento indexado no Azure AI Search. Máximo de 5 chunks por query (ADR-0002). |
| `devolução padrão` | Processo via Portal do Cliente, prazo 7 dias úteis. APENAS para cargas não-perigosas. |

---

### Regras de Comportamento do Assistente

Estas regras governam o comportamento do assistente em runtime. Ao gerar código nos módulos
`src/functions/query/`, `src/services/`, ou `src/bot/`, o agente DEVE garantir que estas
regras sejam implementadas.

#### DEVE

- **DEVE** incluir campo `source_document` em toda resposta da API. Se o LLM não retornar
  fonte, o `response-builder.ts` DEVE rejeitar a resposta e retornar fallback — nunca
  retornar resposta sem fonte ao cliente.

- **DEVE** incluir campo `confidence_level` (float, 0.0–1.0) em toda resposta da API.

- **DEVE** responder em português formal. O system prompt em `/prompts/system-prompt.md`
  contém a instrução de idioma — não remover nem sobrescrever.

- **DEVE** apresentar ambas as versões quando chunks de versões diferentes do mesmo
  documento forem recuperados. O `prompt-builder.ts` DEVE ordenar a versão mais recente
  primeiro no contexto.

- **DEVE** declarar ausência de informação explicitamente quando `confidence_level` < 0.75,
  com sugestão de próximo passo. Formato: "Não encontrei informação sobre [tema] na
  documentação. Recomendo [ação]."

#### NÃO DEVE

- **NÃO DEVE** retornar valor numérico (prazo, multiplicador, SLA em horas, percentual)
  que não esteja literalmente em ao menos um chunk do contexto. O `response-validator.ts`
  DEVE implementar verificação determinística para isto.

- **NÃO DEVE** afirmar que carga perigosa (qualquer das classes 1–6 ANTT) pode ser
  devolvida pelo processo padrão. O `query/handler.ts` DEVE detectar keywords de carga
  perigosa + devolução na query de entrada e injetar template de redirecionamento ao
  ramal 4500 ANTES de chamar o LLM.

- **NÃO DEVE** citar SLAs ou benefícios para tiers além de Gold, Silver e Standard.
  Queries com tier inválido DEVEM receber negativa + lista dos tiers válidos antes de
  chamar o LLM.

- **NÃO DEVE** apresentar chunks com `source_type: "faq_informal"` sem aviso explícito
  de que a fonte não foi validada por Compliance.

#### QUANDO EM DÚVIDA

- Se `confidence_level` < 0.75: prefixar resposta com `"[⚠️ Confiança baixa — confirme com supervisor] "`.
  Este prefixo DEVE ser injetado pelo `response-builder.ts`, não pelo LLM.

- Se chunks contraditórios detectados: apresentar ambos os valores com versão e data.
  Recomendar confirmação com supervisor para chamados em andamento.

- Se query cruza carga perigosa + frete especial + devolução simultaneamente: sugerir
  escalação ao supervisor mesmo com confiança alta.

---

### Restrições que Impactam Geração de Código

Ao gerar código neste projeto, o agente DEVE respeitar:

1. **Campo `source_document` é obrigatório no tipo `AssistantResponse`** (definido em
   `src/shared/types.ts`). Nunca remover este campo. Nunca torná-lo opcional.

2. **Campo `confidence_level` é obrigatório no tipo `AssistantResponse`**. Tipo: `number`
   entre 0 e 1.

3. **O `response-validator.ts` DEVE ser chamado antes de retornar qualquer resposta**
   ao cliente. Não criar shortcuts que bypassem a validação.

4. **Guardrail de carga perigosa é pré-LLM** — a detecção de keywords e injeção do
   template de redirecionamento DEVE acontecer no `query/handler.ts` antes da chamada
   ao `completion.ts`. Nunca delegar este comportamento exclusivamente ao prompt.

5. **Context budget: máximo 5 chunks por query** (ADR-0002). O `search.ts` DEVE limitar
   o número de chunks retornados. Nunca aumentar este limite sem ADR aprovado.

6. **Histórico de conversa: máximo 3 turnos** (ADR-0002). O `prompt-builder.ts` DEVE
   truncar turnos mais antigos. Documentar o truncamento no log.

7. **Chunks do FAQ** (`source_type: "faq_informal"`) DEVEM ter metadado preservado
   através de todo o pipeline. O `search.ts` DEVE retornar `source_type` como parte
   do resultado de cada chunk.

---

### Referências a Specs e Documentação

| Artefato | Caminho no repositório |
|----------|----------------------|
| Requirements do query endpoint | `/specs/query-endpoint/requirements.md` |
| Guardrails detalhados | `/specs/query-endpoint/guardrails.md` |
| Recorte de domínio e linguagem ubíqua | `/specs/query-endpoint/requirements.md` (seção Prior Decisions) |
| System prompt versionado | `/prompts/system-prompt.md` |
| Changelog de prompt | `/prompts/prompt-changelog.md` |
| Documentação NovaTech (fonte de verdade) | `/docs/novatech/` |
| Corpus de chunks para testes | `/data/retrieval-corpus/chunks-novatech.md` |
| ADR-0001 (modelo LLM) | `/docs/adr/0001-escolha-azure-openai.md` |
| ADR-0002 (context budget) | `/docs/adr/0002-estrategia-contexto.md` |
| ADR-0003 (documentos contraditórios) | `/docs/adr/0003-documentos-contraditorios.md` |

---

### Bounded Contexts — Fronteiras para Geração de Código

O assistente NovaTech está dividido em 3 bounded contexts. Ao gerar código, respeite
estas fronteiras:

| Contexto | Módulos | Fora do escopo deste contexto |
|----------|---------|-------------------------------|
| **Atendimento ao Cliente** | `src/functions/query/`, `src/functions/feedback/`, `src/services/`, `src/bot/` | Ingestão de documentos, edição de docs, aprovação de exceções |
| **Documentação e Conhecimento** | `src/pipeline/` | Interface com atendentes, geração de respostas |
| **Qualidade e Governança** | `src/web/` (painel), logs de `query/handler.ts` | Geração de respostas, edição de documentos |
```

---

## Nota sobre o processo de criação

**Iteração com o Claude:**

O rascunho inicial da seção AGENTS.md foi revisado com o seguinte prompt:

```
Você é o Tech Lead deste projeto e vai revisar a seção "Product Rules & Guardrails"
que o PS escreveu para o AGENTS.md. Identifique:
1. Regras narrativas (que descrevem o que é) em vez de prescritivas (que dizem o que fazer)
2. Referências a caminhos de arquivo incorretos ou inconsistentes com o Anexo C
3. Restrições de código que estão vagas demais para influenciar o Copilot

[seção v1 aqui]
```

**Principais ajustes após revisão:**

- Regras de comportamento convertidas de descrições ("o assistente cita fonte") para imperativos com sujeito explícito ("O `response-builder.ts` DEVE rejeitar...")
- Adicionada coluna "Fora do escopo" na tabela de bounded contexts — a v1 só tinha "Módulos"
- Caminhos de arquivo alinhados com estrutura real do Anexo C (ex: `src/functions/query/handler.ts`)
- Guardrail de carga perigosa explicitado como "pré-LLM" — na v1 estava descrito como instrução de prompt, o que é enforcement insuficiente para risco crítico
