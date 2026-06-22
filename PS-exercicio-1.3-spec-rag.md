# Exercício 1.3 — Especificação de Requisitos de RAG (Ponto de Vista do Produto)
**Papel:** Product Specialist | **Cenário:** 1 | **Programa:** Trilha AI First — DGS/DB1

---

## Histórico de Iteração

Este documento apresenta a versão final (V2) da especificação. O histórico de iteração com o Claude está documentado na seção final.

---

# ESPECIFICAÇÃO DE REQUISITOS — PIPELINE RAG
## Assistente de Atendimento NovaTech
**Versão:** 2.0 (refinada com feedback do Claude)
**Autor:** Product Specialist
**Data:** Junho/2026

---

## 1. Fontes de Dados — O que indexar (e o que não indexar)

### 1.1 Fontes incluídas na base

| Fonte | Critério de inclusão | Tratamento especial |
|-------|---------------------|---------------------|
| SharePoint — documentos POL, PROC, SLA vigentes | Todo documento com status "ativo" e data de validade futura ou indefinida | Metadado `vigencia_a_partir` obrigatório |
| Confluence — páginas de procedimento com revisão nos últimos 12 meses | Página deve ter ao menos 1 revisão registrada no histórico | Extrair data da última revisão como metadado |
| Planilhas de referência (pasta de rede) | Apenas planilhas com nome no padrão `tabela-[tipo]-AAAAMM.xlsx` e data ≤ mês atual | Re-indexar automaticamente quando nova versão for publicada |

### 1.2 Fontes EXCLUÍDAS da indexação principal

| Fonte | Motivo da exclusão | Tratamento alternativo |
|-------|-------------------|----------------------|
| PROC-042 v1 após validação da v2 | Substituído pela v2 — manter ambos na base gera respostas contraditórias | Arquivar com metadado `obsoleto: true` — não indexar, mas manter rastreável |
| FAQ-Atendimento (documento completo) | Não validado por Compliance — confiabilidade incerta | Indexar apenas os itens validados formalmente (ver requisito 1.3) |
| Documentos escaneados sem OCR validado | Extração de texto imprecisa contamina embeddings | Processar OCR manualmente antes de indexar; prazo: definir com TI |
| Rascunhos e versões "draft" no SharePoint | Conteúdo não aprovado pode gerar respostas incorretas | Nunca indexar arquivos com sufixo `_draft`, `_v0`, `_wip` ou status "Em revisão" |

**Requisito REQ-F01 (testável):** Dado um documento com metadado `obsoleto: true`, o sistema NÃO deve retornar chunks desse documento em nenhuma query. Teste: buscar por multiplicador regional após arquivar PROC-042 v1 — nenhum chunk da v1 pode aparecer nos resultados.

### 1.3 Tratamento do FAQ informal

O FAQ contém conhecimento operacional crítico não coberto por documentos formais (carga danificada, seguro de carga, frete expresso com carga perigosa). Excluí-lo integralmente gera lacunas relevantes.

**Requisito REQ-F02 (testável):** Itens do FAQ só são indexados após validação formal por Compliance ou Operações. Cada item validado recebe metadado `fonte_tipo: faq_validado` e data de validação. Itens não validados recebem `fonte_tipo: faq_informal` e, se indexados, geram aviso automático na resposta.

**Requisito REQ-F03 (testável):** Toda resposta baseada em fonte do tipo `faq_informal` deve incluir o aviso: "Esta informação provém de documentação informal e não foi validada por Compliance. Confirme com seu supervisor antes de passar ao cliente." Teste: query sobre carga danificada deve incluir o aviso se o único chunk recuperado for do FAQ.

---

## 2. Documentos Contraditórios — Como o assistente deve se comportar

**Contexto:** O caso PROC-042 v1 vs. v2 evidencia que documentos contraditórios coexistirão enquanto o processo de governança documental não estiver maduro.

**Requisito REQ-C01 (testável):** O pipeline de ingestão deve identificar documentos com mesmo código base (ex: PROC-042) e registrar explicitamente a relação entre eles (predecessor/successor) com base na data de emissão. Teste: ingerir PROC-042 v1 e v2 — ambos devem ter campo `relacionado_com` apontando um para o outro.

**Requisito REQ-C02 (testável):** Quando o retriever recuperar chunks de duas versões do mesmo documento na mesma query, o assistente deve:
1. Apresentar os valores de AMBAS as versões (com data de cada uma)
2. Indicar qual é a mais recente
3. Incluir aviso: "Existe mais de uma versão deste procedimento. Confirme com seu supervisor qual versão se aplica ao seu chamado."

Teste: query "qual o multiplicador para o Norte?" deve retornar "v1: 1,6 (mar/2023) | v2: 1,8 (nov/2023) — use a versão aplicável ao seu chamado" — nunca apenas um valor sem contexto.

**Requisito REQ-C03 (testável):** O system prompt do assistente deve conter instrução explícita: "Quando dois chunks do mesmo documento em versões diferentes aparecerem no contexto, NUNCA calcule uma média nem escolha um valor arbitrariamente. Sempre apresente os dois e indique a data de cada versão." Teste: avaliar manualmente 10 respostas sobre frete especial — 100% deve apresentar os dois valores quando chunks de ambas as versões forem recuperados.

---

## 3. Ausência de Resposta — Comportamento quando a pergunta não tem cobertura

**Contexto:** Há perguntas frequentes sem cobertura documental (frete padrão <500kg, processo detalhado de escalação para Gestão de Riscos).

**Requisito REQ-A01 (testável):** O assistente deve declarar ausência de resposta quando o score de similaridade dos chunks recuperados estiver abaixo do threshold definido (valor inicial sugerido: 0,75 — a calibrar em testes). Declaração exata: "Não encontrei informação sobre [tema] na documentação disponível. Consulte [fonte sugerida] ou escale para o supervisor." Teste: query sobre "frete para 300kg" deve retornar ausência de resposta, não um valor inventado.

**Requisito REQ-A02 (testável):** O assistente NUNCA deve usar conhecimento geral do LLM para complementar respostas sobre procedimentos específicos da NovaTech (prazos, valores, multiplicadores, tiers). Se o chunk não tiver a informação, a resposta é "não encontrei". Teste: desabilitar chunks e verificar que o assistente não responde com valores genéricos de mercado.

**Requisito REQ-A03 (testável):** Perguntas sem cobertura devem ser automaticamente registradas em uma fila de "gaps de documentação" para revisão semanal pelo Product Specialist. Teste: após 1 semana de operação, o relatório de gaps deve ser gerado automaticamente com as queries sem resposta.

---

## 4. Atualização — Velocidade de propagação de mudanças

**Requisito REQ-U01 (testável):** Documentos publicados ou atualizados no SharePoint devem estar disponíveis no assistente em até 24 horas úteis após a publicação. Teste: publicar documento de teste no SharePoint às 09h de uma segunda-feira — o documento deve estar disponível no assistente até 09h de terça-feira.

**Requisito REQ-U02 (testável):** Quando um documento for atualizado, o pipeline deve re-indexar APENAS os chunks desse documento (não re-indexar toda a base). Teste: medir tempo de re-indexação de um único documento vs. base completa — re-indexação parcial deve ser concluída em menos de 30 minutos.

**Requisito REQ-U03 (testável):** O pipeline deve monitorar a pasta de rede de planilhas mensalmente e indexar automaticamente novas versões quando a data do arquivo for mais recente que a versão indexada. Teste: publicar nova tabela de fretes base no dia 1 do mês — deve ser indexada automaticamente até o dia 2.

**Requisito REQ-U04 (testável):** Quando um documento for atualizado e substituir uma versão anterior, o pipeline deve marcar os chunks da versão anterior com `obsoleto: true` antes de indexar os novos chunks. Teste: após atualizar POL-001, nenhum chunk da versão anterior deve aparecer em queries.

---

## 5. Rastreabilidade — Citação de fonte e transparência

**Requisito REQ-R01 (testável):** Toda resposta do assistente deve incluir: nome do documento, seção (quando disponível), data da última atualização do documento, e link direto para o documento no SharePoint/Confluence. Teste: 100% das respostas avaliadas em auditoria de qualidade devem conter todos os quatro campos.

**Requisito REQ-R02 (testável):** O trecho exato do documento usado para gerar a resposta deve estar disponível ao atendente sob demanda (botão "Ver fonte completa"). Teste: acessar "Ver fonte completa" em 10 respostas — em 100% dos casos deve exibir o chunk original sem paráfrase.

**Requisito REQ-R03 (testável):** Respostas que combinam informação de múltiplas fontes devem listar todas as fontes individualmente. Teste: query sobre "prazo de devolução para cliente Gold com carga especial" deve citar tanto POL-001 quanto SLA-2024 (e PROC-042 se relevante).

**Requisito REQ-R04 (testável):** Toda interação (query + resposta + fonte citada + ação do atendente) deve ser logada com timestamp, ID do atendente e ID do chamado. Logs devem ser acessíveis para auditoria por 90 dias. Teste: verificar log de auditoria para um chamado específico — deve reconstituir a interação completa.

---

## Histórico de Iteração com o Claude

### V1 — Primeira versão (antes do feedback)

A primeira versão da especificação foi gerada a partir do enunciado do exercício e incluía os 5 temas (fontes, contradições, ausência, atualização, rastreabilidade). Porém, após revisão, identifiquei limitações:

- Requisitos genéricos: "o sistema deve lidar com documentos contraditórios" sem definir o comportamento exato
- Ausência de critérios de testabilidade explícitos
- Sem distinção entre FAQ validado e não validado
- Requisito de atualização sem SLA específico

### Feedback solicitado ao Claude

Prompt de revisão:

```
Revisei a primeira versão da minha especificação de RAG para o projeto NovaTech.
Por favor, identifique:
1. Requisitos que não são testáveis (um QA não conseguiria escrever um teste para eles)
2. Ambiguidades que um desenvolvedor poderia interpretar de formas diferentes
3. Gaps: casos reais do projeto NovaTech que a especificação não cobre

[V1 da especificação colada aqui]
```

### Gaps e ambiguidades identificados pelo Claude

1. **"Lidar com contradições" não é requisito** — o que exatamente o assistente exibe? A v1 não especificava
2. **Threshold de confiança não definido** — "baixa confiança" é subjetivo sem um valor numérico
3. **FAQ não endereçado explicitamente** — a v1 dizia "indexar documentos validados" mas não resolvia o FAQ que tem conteúdo crítico mas é informal
4. **Re-indexação parcial não mencionada** — re-indexar a base toda a cada atualização seria inviável operacionalmente
5. **Logs sem prazo de retenção** — "logs devem ser mantidos" sem prazo é inútil para compliance

### Melhorias incorporadas na V2

- REQ-C02 e REQ-C03 especificam o comportamento exato para contradições (apresentar ambas as versões com data)
- REQ-A01 define threshold inicial de 0,75 (a calibrar)
- REQ-F02 e REQ-F03 endereçam o FAQ com distinção entre validado e informal
- REQ-U02 especifica re-indexação parcial
- REQ-R04 define retenção de 90 dias para logs

**Delta verificável entre V1 e V2:** V1 tinha 12 requisitos sem código e sem critério de teste. V2 tem 15 requisitos codificados (REQ-F01 a REQ-R04) com teste explícito para cada um.
