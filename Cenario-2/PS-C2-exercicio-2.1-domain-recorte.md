# Exercício 2.1 — Recorte de Domínio e Spec SDD do Query Endpoint

> **Projeto:** NovaTech — Assistente de IA para o time de Atendimento  
> **Módulo:** Query Endpoint (consulta do atendente à base de conhecimento)  
> **Cenário:** 2 — Definição e Especificação · **Papel:** Product Specialist  
> **Método:** SDD (Spec-Driven Development) · **Versão:** v2 (após iteração com Tech Lead)  
> **Fonte de verdade do domínio:** Anexo A (POL-001, PROC-042 v1/v2, SLA-2024, FAQ-Atendimento)  

---

## 0. Prior Decisions — herança do Cenário 1

Este módulo **não parte do zero**. Ele implementa as decisões tomadas na fase de Entendimento e Contexto (Cenário-Âncora 1). As ADRs abaixo são pré-requisitos, não escopo deste spec.

| # | Decisão anterior (Cenário 1) | Impacto neste módulo |
|---|------------------------------|----------------------|
| **ADR-01** | Arquitetura **RAG** sobre os 5 documentos do Anexo A, ingeridos como arquivos individuais (chunking + embedding). | O endpoint recupera trechos e **sempre** devolve a fonte; não é um LLM "de cabeça". |
| **ADR-02** | **Context budget / progressive disclosure** — não injetar os 5 documentos inteiros por consulta. Recuperação seletiva por relevância (aprendizado do Ex. 1.1: contexto em excesso degrada qualidade). | Top-k limitado; cada resposta cita apenas os chunks efetivamente usados. |
| **ADR-03** | **Contradições não são resolvidas pelo LLM.** PROC-042 v1 vs v2 coexistem sem hierarquia formal → a máquina expõe ambas, nunca escolhe silenciosamente. | Requisito funcional central deste módulo (ver §4 RF-05). |
| **ADR-04** | **Autoridade documental é um dado de primeira classe.** Documento normativo (POL) > contratual (SLA) > procedimento (PROC) > informal não-validado (FAQ). | Alimenta o cálculo de confiança e o aviso obrigatório para conteúdo de FAQ. |
| **ADR-05** | Infra já existente: chamados e medição de SLA rodam em **Azure DevOps** (timestamp de abertura). | O endpoint reusa a identidade do chamado/atendente; não cria novo sistema de tickets. |

> **Escala de referência (Cenário 1):** ~320 chamados/dia. O endpoint é caminho quente do atendimento — latência e confiabilidade são requisitos, não desejáveis.

---

## 1. Recorte de Domínio — Bounded Contexts

A NovaTech não é "um sistema". O conhecimento operacional está particionado por **domínio de negócio**. O recorte abaixo é por capacidade de negócio, **não** por camada técnica (não existe contexto "frontend" ou "backend").

```
                 ┌─────────────────────────────────────────┐
                 │   ATENDIMENTO  (contexto CORE — este módulo)│
                 │   O atendente pergunta; a IA responde       │
                 │   com fonte + confiança + escalação.        │
                 └───────────────┬─────────────────────────────┘
                                 │ consulta (read-only)
        ┌────────────┬───────────┼───────────────┬──────────────────┐
        ▼            ▼           ▼               ▼                  ▼
 ┌───────────┐┌────────────┐┌──────────┐┌───────────────┐┌──────────────────┐
 │ DEVOLUÇÕES││FRETE ESPECIAL││ SLAs &   ││ GESTÃO DE     ││ GESTÃO DOCUMENTAL│
 │ (POL-001) ││ (PROC-042)  ││ TIERS    ││ RISCOS        ││ (fora de escopo) │
 │           ││ v1 ⚠ v2     ││(SLA-2024)││(carga perigosa)││ dono das fontes  │
 └───────────┘└────────────┘└──────────┘└───────────────┘└──────────────────┘
```

### 1.1 Contexto CORE — **Atendimento**
O único contexto que este módulo **implementa**. Traduz a pergunta do atendente em uma resposta rastreável. Não detém regras de negócio próprias: ele **lê** os demais contextos e orquestra fonte, confiança e escalação.

### 1.2 Contextos consultados (read-only)

| Bounded context | Fonte de verdade | Papel na consulta |
|-----------------|------------------|-------------------|
| **Devoluções** | POL-001 (normativo) | Elegibilidade, prazos, exceções, custos, coleta reversa. |
| **Frete Especial** | PROC-042 v1 **e** v2 (procedimento, sem vigência formal) | Fórmula, multiplicadores regionais, fator de peso, prazo adicional. **Contexto onde vive a contradição.** |
| **SLAs & Tiers** | SLA-2024 (contratual) | Tiers Gold/Silver/Standard, tempos de resposta/resolução, definição de incidente crítico. |
| **Gestão de Riscos** | POL-001 §3.2 + FAQ | O módulo sabe **quando** escalar (carga perigosa ANTT 1-6, cadeia de frio, lacre violado) — não executa a tratativa. |

### 1.3 Fronteiras — o que este módulo NÃO cobre (scope boundaries derivadas dos contextos)

| Fora de escopo | Bounded context responsável | Por quê |
|----------------|-----------------------------|---------|
| Ciclo de vida e versionamento das fontes | **Gestão Documental** | O módulo consome documentos; não decide qual PROC-042 é vigente (isso é gap organizacional — ver §6). |
| **Cálculo real** do valor de frete | Frete Especial (execução) | A tabela mensal `frete-base-AAAAMM.xlsx` não é fonte ingerida (ADR-02). O módulo explica a regra, não computa o preço. |
| Execução de coleta reversa e reembolso | Operações / Devoluções (execução) | O módulo informa prazos; não agenda coleta nem processa crédito. |
| Tratativa de carga perigosa devolvida | **Gestão de Riscos** | Termina no encaminhamento ao ramal 4500. |
| Carga danificada em trânsito / sinistros | **Jurídico** (sinistros@novatech.com.br) | Existe só no FAQ informal (gap §6); o módulo encaminha, não decide. |
| Descontos e negociação comercial | **Comercial** | Atendente não tem autonomia (FAQ item 45). |
| Frete padrão (< 500kg) | — (gap §6) | Nenhum documento na base cobre; o módulo declara ausência de fonte (RF-04). |

---

## 2. Linguagem Ubíqua

Termos que um LLM **confundiria ou inventaria** sem esta definição. Todos extraídos do Anexo A — não há termos genéricos ("cliente = quem contrata") nesta lista.

| Termo | Definição (fonte) | Por que é armadilha para o LLM |
|-------|-------------------|-------------------------------|
| **Tier Gold / Silver / Standard** | Os **três** — e únicos — níveis de cliente (SLA-2024 §1). Gold: contrato > R$ 500k **ou** > 200 op./mês. | **Não existe "Platinum".** Cliente pode alegar (FAQ item 15); LLM tende a validar tier inexistente. |
| **Carga perigosa** | Carga das **classes 1 a 6 da ANTT** (Res. 5.947/2021): explosivos, gases, inflamáveis, oxidantes, tóxicos/infectantes (POL-001 §3.2). | LLM pode tratar como "carga frágil/valiosa". Aqui é classificação regulatória com regra de devolução própria. |
| **Frete especial** | Frete de carga com **peso acima de 500kg** (PROC-042 §1). | LLM associa "especial" a urgência/VIP. É estritamente uma faixa de peso. |
| **Incidente crítico** | Classificação com gatilhos objetivos (SLA-2024 §3): valor > R$ 100k com status desconhecido > 6h; carga perigosa com irregularidade; > 5 chamados do mesmo cliente em 24h; risco a pessoas. | LLM usa "crítico" de forma subjetiva. Aqui é binário e dispara SLA reduzido. |
| **SLA de resposta ≠ SLA de resolução** | Resposta = 1º retorno ao cliente (mesmo "estamos verificando"). Resolução = problema efetivamente resolvido (FAQ item 41). | LLM funde os dois. São dois relógios distintos por tier. |
| **Dias úteis** | Exclui sábados, domingos e feriados nacionais (POL-001 §3.1). Relógio de SLA pausa fora de 08h-18h — **exceto** incidente crítico Gold (SLA-2024 §5). | LLM calcula em dias corridos e ignora a pausa/exceção. |
| **Cadeia de frio rompida** | Temperatura fora da faixa da NF por **> 30 min contínuos** (sensor IoT) — torna carga refrigerada inelegível para devolução padrão (POL-001 §3.2). | Critério quantitativo específico; LLM inventaria limites. |
| **Multiplicador regional / Fator de peso** | Componentes da fórmula de frete especial, **com valores divergentes entre PROC-042 v1 e v2** (§5 deste doc). | LLM escolhe um valor sem avisar da divergência. |
| **CT-e** | Conhecimento de Transporte Eletrônico — nº obrigatório para abrir devolução (POL-001 §3.3). | Sigla de domínio; LLM pode confundir com NF-e. |
| **Coleta reversa** | Logística de retirada da mercadoria devolvida, agendada em até 2 dias úteis após aprovação (POL-001 §3.3). | Termo específico do fluxo de devolução. |
| **Documento não-validado** | FAQ-Atendimento: informal, **sem responsável formal, não validado por Compliance** (FAQ, cabeçalho). | LLM trata FAQ com a mesma autoridade de POL/SLA. Exige aviso obrigatório (RF-06). |

---

## 3. Outcomes (orientados a resultado)

O que muda para o negócio quando o módulo existe — **não** o que a feature faz.

| # | Outcome | Métrica de resultado |
|---|---------|---------------------|
| **O-1** | O atendente obtém uma resposta **fundamentada** sem sair do chamado nem procurar o documento manualmente. | Tempo até primeira resposta cai; ≥ 90% das consultas respondidas em **< 30s** (dentro do SLA de resposta de todos os tiers). |
| **O-2** | O atendente **cita a fonte** ao cliente com segurança. | 100% das respostas exibem o documento e o trecho de origem. |
| **O-3** | O atendente **nunca repassa informação de fonte não-validada** como se fosse oficial. | 0 respostas derivadas do FAQ sem o aviso "não validado por Compliance". |
| **O-4** | Contradições da base não viram erro operacional (ex.: cobrar multiplicador errado). | 0 casos de multiplicador aplicado sem que a divergência v1/v2 tenha sido exibida. |
| **O-5** | Escalações para Gestão de Riscos acontecem **quando devem** — nem a mais, nem a menos. | Toda consulta sobre carga perigosa/cadeia de frio/lacre retorna a instrução de escalar (ramal 4500). |
| **O-6** | A base **melhora com o uso**: erros reportados pelo atendente chegam ao dono do documento. | Todo feedback negativo gera item na fila de curadoria (§4 RF-07). |

---

## 4. Requisitos Funcionais (verificáveis / binários)

Cada RF é redigido para que o **QA escreva um teste passa/não-passa**. Formato Given/When/Then.

| RF | Requisito (Given / When / Then) | Teste do QA (pass = ✅) |
|----|--------------------------------|------------------------|
| **RF-01 · Fonte obrigatória** | **Given** qualquer consulta respondida, **When** o endpoint retorna, **Then** a resposta contém ≥ 1 documento de origem (ID + trecho citado). | Resposta sem `sources[]` não-vazio → ❌ |
| **RF-02 · Confiança exibida** | **Given** uma resposta, **Then** ela inclui um nível de confiança ∈ {Alta, Média, Baixa} derivado de (score de recuperação × autoridade da fonte × presença de contradição) — ADR-04. | Ausência do campo `confidence` → ❌ |
| **RF-03 · Escalação de carga perigosa** | **Given** consulta sobre devolução de carga perigosa (ANTT 1-6), cadeia de frio rompida ou lacre violado, **Then** a resposta afirma "não elegível pelo processo padrão" **e** instrui contato com Gestão de Riscos (ramal 4500). POL-001 §3.2. | Falta de qualquer das duas afirmações → ❌ |
| **RF-04 · Ausência de fonte** | **Given** consulta cuja resposta não existe na base (ex.: frete padrão < 500kg, seguro de carga, carga danificada), **Then** o endpoint declara "sem fonte formal na base" e **não fabrica** número/prazo; oferece encaminhamento. | Resposta inventa valor sem fonte → ❌ |
| **RF-05 · Contradição não resolvida pela máquina** | **Given** consulta de frete especial onde PROC-042 v1 **e** v2 são relevantes, **Then** a resposta exibe **ambas** as versões com data de emissão e sinaliza a divergência (multiplicadores, fator de peso 1.2/1.5 vs 1.15/1.4, prazo +2 vs +3 dias) e a regra de transição (chamados < 01/12/2023 → v1). Nunca escolhe silenciosamente. ADR-03. | Resposta cita só uma versão sem aviso → ❌ |
| **RF-06 · Aviso de fonte não-validada** | **Given** que a resposta deriva (parcial ou totalmente) do FAQ-Atendimento, **Then** inclui aviso explícito "informação não validada por Compliance — confirmar na normativa" e rebaixa a confiança para no máximo **Média**. | FAQ citado sem aviso, ou confiança Alta → ❌ |
| **RF-07 · Tier inexistente** | **Given** consulta que menciona tier fora de {Gold, Silver, Standard} (ex.: "Platinum"), **Then** a resposta afirma que o tier não existe, lista os três válidos e sugere verificar o nº do contrato. FAQ item 15 / SLA-2024 §1. | Resposta valida ou "assume" Platinum → ❌ |
| **RF-08 · Loop de feedback** | **Given** que o atendente marca a resposta como incorreta/incompleta, **When** envia o feedback, **Then** é criado um item na **fila de curadoria** (Gestão Documental) com: pergunta, resposta, fontes citadas e comentário; o item é roteado ao responsável do documento. | Feedback que não gera item rastreável → ❌ |
| **RF-09 · Latência** | **Given** carga de produção (~320 chamados/dia), **Then** p95 do tempo de resposta < 30s. | p95 ≥ 30s → ❌ |

---

## 5. Verification Criteria — casos de teste-âncora

Casos concretos (com dados do Anexo A) para o QA validar de ponta a ponta. Todos binários.

| # | Pergunta do atendente | Resultado esperado (pass) | RF coberto |
|---|-----------------------|---------------------------|-----------|
| VC-1 | "Cliente quer devolver uma carga de líquido inflamável entregue ontem." | "Não elegível pelo processo padrão (classe ANTT 3). Encaminhar à Gestão de Riscos — ramal 4500." Fonte: POL-001 §3.2. | RF-03, RF-01 |
| VC-2 | "Quanto é o multiplicador de frete para o Nordeste?" | Exibe **ambas**: PROC-042 v1 = **1.4** / v2 = **1.5**, com datas, aviso de coexistência e regra de transição. Confiança rebaixada por contradição. | RF-05, RF-02 |
| VC-3 | "O cliente disse que é Platinum, qual o SLA dele?" | "Não existe tier Platinum. Tiers válidos: Gold, Silver, Standard. Peça o nº do contrato." | RF-07 |
| VC-4 | "Posso mandar carga perigosa com frete expresso?" | Responde a partir do FAQ item 32 **com** aviso "não validado por Compliance"; confiança ≤ Média; sugere confirmar. | RF-06 |
| VC-5 | "Qual a tabela de frete para uma carga de 300kg?" | "Sem fonte formal na base para frete < 500kg. Encaminhar ao Comercial." Não inventa valor. | RF-04 |
| VC-6 | "Qual o prazo de resposta para incidente crítico de cliente Gold?" | "Até 30 min; o relógio de SLA **não pausa** fora do horário para incidente crítico Gold." Fonte: SLA-2024 §2/§5. | RF-01, RF-02 |

---

## 6. Gaps conhecidos da base (limitam o módulo — não são bugs)

Herdados do mapeamento do Cenário 1. O módulo os **expõe** (via RF-04), não os resolve — resolução pertence à Gestão Documental.

1. **PROC-043 (frete de carga perigosa)** citada mas não ingerida / em revisão pelo Compliance.
2. **Frete padrão < 500kg** — sem documento na base.
3. **Seguro de carga** — só no FAQ (item 22), sem norma formal.
4. **Carga danificada em trânsito** — só no FAQ (item 38); processo real é do Jurídico.
5. **Processo interno da Gestão de Riscos** para carga perigosa devolvida — não documentado (POL-001 cita só o ramal).

---

## 7. Iteração com o Tech Lead (v1 → v2)

Registro das ambiguidades levantadas pelo Tech Lead (Claude) na revisão da v1 e como foram resolvidas na v2. Mudanças **substantivas**, não cosméticas.

| # | Ambiguidade apontada na v1 | Resolução incorporada na v2 |
|---|----------------------------|-----------------------------|
| **A-1 · "Confiança" era um número mágico** | v1 dizia "a resposta mostra a confiança" sem definir como se calcula. Tech Lead: "confiança de quê? O QA não consegue testar." | RF-02 agora define a fórmula (score de recuperação × **autoridade da fonte** — ADR-04 × presença de contradição) e a escala {Alta, Média, Baixa}. FAQ nunca gera Alta (RF-06). |
| **A-2 · Contradição "usar a mais recente"** | v1 mandava adotar PROC-042 v2 por ser mais nova. Tech Lead: "e a regra de transição? Chamado aberto antes de 01/12/2023 usa v1 (PROC-042-v2 §5)." | RF-05 reescrito: a máquina **nunca** escolhe; exibe ambas + datas + regra de transição e pede o dado que desempata (data do chamado). Alinha com ADR-03. |
| **A-3 · Escopo vazando para cálculo de preço** | v1 previa "o assistente calcula o valor do frete". Tech Lead: "a tabela mensal `.xlsx` não está na base (ADR-02) — vai alucinar preço." | Movido para §1.3 (fora de escopo). O módulo **explica a regra e cita a fonte**; não computa valor. RF-04 cobre a ausência. |
| **A-4 · Feedback sem destino** | v1 tinha "botão de feedback" sem processo atrás. Tech Lead: "para onde vai? Quem corrige? Como volta ao assistente?" | RF-08 fecha o loop: feedback → **fila de curadoria (Gestão Documental)** → responsável do documento revisa → reindexação. Sustenta o outcome O-6. |
| **A-5 · Aviso de FAQ opcional** | v1 sugeria "idealmente avisar quando a fonte é o FAQ". Tech Lead: "‘idealmente’ não é testável e é justamente o maior risco de repassar informação errada ao cliente." | Virou requisito duro (RF-06) + rebaixamento automático de confiança; caso-âncora VC-4. |

---

## 8. Definition of Done

- [ ] Todos os RF-01…RF-09 com teste automatizado passando.
- [ ] Casos-âncora VC-1…VC-6 verdes.
- [ ] Nenhuma resposta em produção deriva do FAQ sem aviso (auditoria RF-06).
- [ ] Fila de curadoria recebendo e roteando feedback (RF-08).
- [ ] p95 < 30s sob carga de referência (RF-09).
