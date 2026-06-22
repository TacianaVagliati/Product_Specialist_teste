# Exercício 2.1 — Recorte de Domínio e Spec SDD do Query Endpoint
**Papel:** Product Specialist | **Cenário:** 2 | **Programa:** Trilha AI First — DGS/DB1

---

## Parte 1 — Recorte de Domínio

### 1.1 Bounded Contexts do Projeto NovaTech Assistant

Análise realizada com o Claude a partir do Anexo A e dos dados do discovery. O princípio usado: cada contexto agrupa conceitos que têm coesão de negócio forte entre si e fronteira clara com os demais.

---

#### Bounded Context 1 — Atendimento ao Cliente (core domain)

**Dentro deste contexto:**
- Perguntas dos atendentes sobre prazos, fretes, devoluções e SLAs
- Respostas do assistente com citação de fonte
- Fluxo de fallback quando a confiança é baixa
- Registro de feedback de respostas incorretas
- Histórico de consultas por atendente e por chamado

**Fora deste contexto:**
- Publicação ou edição de documentos (isso é Gestão Documental)
- Negociação de contratos ou SLAs (isso é Comercial)
- Aprovação de exceções para cargas perigosas (isso é Gestão de Riscos)

**Relacionamento com outros contextos:**
- Consome do contexto **Documentação e Conhecimento** (os chunks indexados)
- Produz feedback que alimenta o contexto **Qualidade e Governança**

---

#### Bounded Context 2 — Documentação e Conhecimento

**Dentro deste contexto:**
- Pipeline de ingestão de documentos (extração, chunking, embedding, indexação)
- Versionamento e controle de vigência dos documentos
- Tratamento de documentos contraditórios (ex: PROC-042 v1 vs v2)
- Detecção e marcação de documentos obsoletos

**Fora deste contexto:**
- Criação ou aprovação dos documentos originais (isso é responsabilidade das áreas da NovaTech — Operações, Compliance, Comercial)
- Interface com o atendente (isso é Atendimento ao Cliente)

**Relacionamento com outros contextos:**
- Alimenta o contexto **Atendimento ao Cliente** com chunks indexados e metadados de vigência
- Recebe sinalizações do contexto **Qualidade e Governança** sobre gaps e erros

---

#### Bounded Context 3 — Qualidade e Governança

**Dentro deste contexto:**
- Registro e triagem de feedbacks de respostas incorretas
- Dashboard de métricas de qualidade (taxa de confiança baixa, perguntas sem resposta, erros reportados)
- Alertas sobre padrões de falha recorrentes
- Rastreabilidade de resposta → documento fonte

**Fora deste contexto:**
- Geração das respostas (isso é Atendimento ao Cliente)
- Correção dos documentos (isso é Documentação e Conhecimento)

**Relacionamento com outros contextos:**
- Recebe dados de **Atendimento ao Cliente** (feedbacks, logs de consultas)
- Envia sinalizações para **Documentação e Conhecimento** (gaps, documentos desatualizados)

---

#### Contexto de Suporte — SLAs e Contratos (supporting domain)

Este contexto existe na NovaTech mas está **fora do escopo do assistente como emissor**. O assistente **consulta** as regras de SLA para responder perguntas dos atendentes, mas não gerencia contratos nem tiers. As regras de SLA são dados de entrada para o contexto de Atendimento ao Cliente.

---

### 1.2 Linguagem Ubíqua do Domínio

Termos extraídos do Anexo A que precisam ser usados de forma consistente por todos os membros do time e por todos os agentes de IA. Termos sem esta definição seriam interpretados de forma ambígua por um LLM.

| Termo | Definição canônica | Contexto de uso | Erro comum sem definição |
|-------|-------------------|-----------------|--------------------------|
| **carga perigosa** | Carga classificada nas classes 1 a 6 da ANTT, conforme Resolução nº 5.947/2021. Inclui: explosivos (1), gases (2), líquidos inflamáveis (3), sólidos inflamáveis (4), oxidantes/peróxidos (5), tóxicos/infectantes (6) | Atendimento, Documentação | LLM pode interpretar como "carga de alto valor" ou "carga frágil" |
| **frete especial** | Modalidade de frete aplicável a cargas com peso **acima de 500kg**. Calculado com fórmula: Valor base × Multiplicador regional × Fator de peso | Atendimento, Documentação | LLM pode confundir com "frete expresso" ou "frete premium" |
| **cliente Gold** | Tier de cliente com contrato anual acima de R$ 500.000 **ou** mais de 200 operações/mês. SLA: resposta em até 2h úteis, resolução em até 24h úteis | Atendimento | LLM pode interpretar "Gold" como o metal ou como sinônimo genérico de "premium" |
| **cliente Silver** | Tier de cliente com contrato anual entre R$ 100.000 e R$ 500.000 **ou** entre 50 e 200 operações/mês | Atendimento | Confusão com "cliente Platinum" (tier inexistente) |
| **cliente Standard** | Todos os demais clientes. SLA: resposta em até 8h úteis, resolução em até 72h úteis | Atendimento | Nenhum outro tier existe além de Gold, Silver e Standard |
| **multiplicador regional** | Fator numérico aplicado ao valor base do frete especial conforme região de destino. Versão vigente (PROC-042-v2): Sul 1.3, Sudeste 1.1, CO 1.4, NE 1.5, Norte 1.8 | Documentação, Atendimento | LLM pode usar valores da v1 (desatualizada) ou inventar multiplicadores |
| **fator de peso** | Multiplicador aplicado ao frete especial conforme faixa de peso. Vigente: 1.0 (500-1.000kg), 1.15 (1.001-3.000kg), 1.4 (>3.000kg) | Documentação, Atendimento | Confusão com fatores da v1: 1.0/1.2/1.5 |
| **SLA de resposta** | Tempo máximo para o primeiro retorno ao cliente após abertura do chamado (mesmo que seja "estamos verificando") | Atendimento | Confusão com SLA de resolução |
| **SLA de resolução** | Tempo máximo para o encerramento efetivo do chamado | Atendimento | Confusão com SLA de resposta |
| **incidente crítico** | Chamado que atende a ao menos um dos critérios: carga >R$100k com status desconhecido há >6h; carga perigosa com irregularidade; >5 chamados do mesmo cliente em 24h sobre o mesmo problema; risco à segurança de pessoas | Atendimento | LLM pode classificar qualquer chamado urgente como "crítico" |
| **chunk** | Trecho de documento indexado no Azure AI Search, usado como contexto para geração de resposta pelo LLM | Documentação, interno do time | Termo técnico sem ambiguidade para o time, mas deve constar no glossário |
| **source_document** | Campo obrigatório no JSON de retorno da API, contendo identificador do documento fonte e seção | Código, Atendimento | Campo pode ser omitido em respostas sem fonte clara |
| **confiança** | Score de similaridade do retrieval. Respostas com score abaixo do threshold devem incluir aviso explícito | Atendimento, Qualidade | LLM pode usar "confiança" como auto-avaliação subjetiva |
| **devolução padrão** | Processo de devolução via Portal do Cliente, elegível apenas para cargas não-perigosas dentro do prazo de 7 dias úteis | Atendimento | LLM pode aplicar o processo padrão a cargas perigosas (incidente real) |

---

## Parte 2 — Requirements.md do Query Endpoint

> Este documento é o artefato a ser salvo em `specs/query-endpoint/requirements.md` no repositório.

---

```markdown
# requirements.md — Query Endpoint
**Módulo:** query-endpoint
**Bounded Context:** Atendimento ao Cliente
**Autor:** Product Specialist
**Status:** Em Revisão (aguarda aprovação do Tech Lead — Gate 1)
**Versão:** 1.1 (refinada após feedback simulado do Tech Lead — ver seção Prior Decisions)

---

## Outcomes

Resultados mensuráveis que definem sucesso deste módulo para os atendentes:

- O1: Atendente recebe resposta fundamentada em documentação em menos de 30 segundos
  para 95% das queries em condições normais de carga.
- O2: Atendente sabe sempre qual documento embasou a resposta (sem "caixa preta").
- O3: Atendente nunca recebe informação incorreta sobre cargas perigosas ou tiers inexistentes
  sem aviso explícito de incerteza.
- O4: Quando o assistente não sabe, o atendente recebe orientação clara sobre o próximo passo
  (escalar, consultar documento diretamente, ligar para ramal específico).

---

## Scope Boundaries

Este módulo cobre o bounded context **Atendimento ao Cliente**:

**Dentro do escopo:**
- Receber pergunta do atendente em linguagem natural
- Recuperar chunks relevantes do índice (Azure AI Search)
- Gerar resposta em português formal com citação de fonte
- Retornar estrutura JSON com `answer`, `source_document`, `confidence_level`, `chunks_used`
- Aplicar guardrails de comportamento (ver seção Constraints)
- Registrar log de cada interação (query, chunks recuperados, resposta, ID atendente, ID chamado)

**Fora do escopo deste módulo:**
- Ingestão ou atualização de documentos (módulo: `pipeline-ingestao`)
- Persistência de histórico de conversas (fora do escopo desta fase)
- Interface visual no Teams (módulo: `teams-bot`)
- Dashboard de métricas (módulo: `painel-web`)
- Aprovação de exceções para cargas perigosas (responsabilidade da Gestão de Riscos NovaTech)

---

## Constraints

Restrições não-negociáveis derivadas dos guardrails definidos e dos incidentes registrados:

**C1 — Rastreabilidade obrigatória**
Toda resposta DEVE incluir o campo `source_document` com identificador do documento
e seção. Respostas sem fonte identificável não devem ser retornadas como confiáveis.

**C2 — Proibição de invenção numérica**
O assistente NÃO DEVE gerar valores numéricos (prazos em dias, multiplicadores de frete,
valores de SLA em horas) que não estejam literalmente presentes nos chunks recuperados.
Deriva do Incidente 2 (multiplicadores da versão desatualizada).

**C3 — Guardrail de carga perigosa**
Queries que mencionen carga perigosa (classes 1-6 ANTT, ou qualquer substância das classes
listadas em POL-001 seção 3.2) NÃO DEVEM retornar informação de devolução pelo processo
padrão. Devem redirecionar ao ramal 4500 (Gestão de Riscos). Deriva do Incidente 1.

**C4 — Guardrail de tiers**
O assistente NÃO DEVE inventar tiers de cliente além de Gold, Silver e Standard.
Queries sobre "Platinum" ou outros tiers devem retornar negativa com lista dos tiers válidos.

**C5 — Comportamento de baixa confiança**
Quando o score de similaridade estiver abaixo do threshold (a calibrar em testes — valor
inicial sugerido: 0.75), a resposta DEVE incluir aviso explícito e sugerir escalação.
Não retornar silêncio — retornar orientação de próximo passo.

**C6 — Context budget**
System prompt + chunks recuperados + pergunta + histórico limitado a 3 turnos DEVEM
caber no orçamento de ~12K tokens (ADR-0002). Máximo de 5 chunks de ~1.500 tokens cada.

**C7 — Documentos contraditórios**
Quando chunks de versões diferentes do mesmo documento forem recuperados, a resposta
DEVE apresentar ambas as versões com data de cada uma, sem calcular média ou escolher
arbitrariamente. (ADR-0003)

---

## Prior Decisions

Decisões arquiteturais do Cenário 1 que este módulo deve respeitar:

| ADR | Decisão | Impacto neste módulo |
|-----|---------|---------------------|
| ADR-0001 | Modelo: Azure OpenAI (GPT-4o), janela 128K | Limite prático de context budget (ADR-0002 define ~12K para uso real) |
| ADR-0002 | Context budget: ~4K system prompt + ~8K chunks (5 × ~1.500 tokens) + pergunta + 3 turnos | Constraint C6 — influencia chunking e retrieval |
| ADR-0003 | Documentos contraditórios: apresentar ambas as versões, priorizar mais recente, não excluir obsoletos | Constraint C7 — lógica de detecção de contradições no prompt-builder |
| ADR-0004 | Protótipo identificou problemas de chunking em tabelas (ChromaDB + sentence-transformers) | Pipeline de ingestão deve ter tratamento especial para tabelas (escopo do módulo pipeline-ingestao) |

---

## Verification Criteria

Critérios testáveis pelo QA — cada um deve ter teste automatizado correspondente:

| ID | Critério | Tipo de verificação |
|----|----------|-------------------|
| VC-01 | 95% das queries respondem em < 30 segundos sob carga normal | Teste de performance (p95 latência) |
| VC-02 | 100% das respostas incluem campo `source_document` no JSON de retorno | Teste unitário no response-builder |
| VC-03 | Queries sobre carga perigosa + devolução retornam negativa explícita e ramal 4500 | Teste com fixture de carga perigosa (Chunks POL-001-B) |
| VC-04 | Queries sem chunks com score ≥ threshold retornam mensagem padrão "não encontrei" com sugestão | Teste com query sobre tópico não indexado |
| VC-05 | Queries sobre "cliente Platinum" retornam negativa com lista dos tiers válidos | Teste com fixture de tier inexistente |
| VC-06 | Quando chunks de PROC-042 v1 e v2 são recuperados juntos, ambas as versões são apresentadas | Teste com fixture de chunks contraditórios |
| VC-07 | Campo `confidence_level` presente e ≥ 0.75 em respostas sem aviso de baixa confiança | Teste unitário no response-builder |
| VC-08 | Histórico de conversa truncado a 3 turnos — turnos mais antigos descartados | Teste unitário no prompt-builder |
```

---

## Parte 3 — Mockup da Interface de Resposta no Teams

O mockup abaixo representa a interface do Adaptive Card exibida no Teams para o atendente. Criado com Claude Design a partir dos requirements acima.

*(Arquivo SVG/PNG do mockup disponível como `PS-C2-ex21-mockup-teams.svg`)*

**Elementos obrigatórios derivados dos requirements:**
- Campo de pergunta (input)
- Área de resposta em texto natural
- Badge de confiança (Alta / Média / Baixa)
- Citação de fonte com nome do documento e seção
- Botão "Resposta incorreta" (feedback)
- Mensagem de fallback quando confiança é baixa

---

## Parte 4 — Histórico de Iteração com o Claude (como Tech Lead)

### Prompt de revisão enviado

```
Você é o Tech Lead deste projeto. Acabei de escrever o requirements.md do query endpoint
(colo abaixo). Identifique:
1. Ambiguidades que um desenvolvedor interpretaria de formas diferentes
2. Verification criteria que não são testáveis como estão escritos
3. Constraints que faltam dado o que sabemos sobre os incidentes registrados

[v1 do requirements.md colado aqui]
```

### Feedback do "Tech Lead" (Claude) — principais pontos

1. **C5 estava ambíguo:** "baixa confiança" sem valor numérico não é implementável. → Adicionado "valor inicial sugerido: 0.75" e instrução para calibrar em testes.

2. **VC-01 incompleto:** "em < 30 segundos" sem definir condições de carga e percentil. → Refinado para "95% das queries sob carga normal (p95 latência)".

3. **Falta constraint sobre context budget:** A v1 não mencionava o limit de 12K tokens da ADR-0002, o que deixaria o Dev sem orientação para o prompt-builder. → Adicionado C6.

4. **VC-06 ausente na v1:** O tratamento de documentos contraditórios (ADR-0003) não tinha critério de verificação correspondente. → Adicionado VC-06.

5. **`confidence_level` não mencionado no JSON de retorno:** A v1 listava `answer`, `source_document`, `chunks_used` mas esqueceu o campo de confiança, que é necessário para VC-05 e C5. → Adicionado à lista de campos obrigatórios no Scope.

### Delta V1 → V2 verificável

| Item | V1 | V2 |
|------|----|----|
| C5 | "score abaixo do threshold" | "score abaixo de 0.75 (a calibrar)" |
| VC-01 | "< 30 segundos" | "95% < 30s sob carga normal (p95)" |
| C6 | Ausente | Adicionado (context budget 12K tokens, ADR-0002) |
| VC-06 | Ausente | Adicionado (chunks contraditórios) |
| JSON de retorno | 3 campos | 4 campos (adicionado `confidence_level`) |

