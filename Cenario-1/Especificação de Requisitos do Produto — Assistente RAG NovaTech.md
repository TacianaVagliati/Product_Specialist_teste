
# Especificação de Requisitos do Produto — Assistente RAG NovaTech
## Histórico de Iteração com Revisão Claude

**Projeto:** Assistente de IA para Atendimento ao Cliente — NovaTech  
**Autor:** QA — DB1 Group  
**Data:** Junho 2025  
**Base documental:** POL-001, PROC-042 v1 e v2, SLA-2024, FAQ-Atendimento (análise prévia)

---

## Como ler este documento

Este arquivo registra o processo completo de especificação iterativa:

1. **Versão 1 (V1):** especificação inicial produzida com base na leitura dos documentos.
2. **Revisão Claude:** feedback estruturado identificando gaps e ambiguidades na V1.
3. **Versão 2 (V2):** especificação refinada incorporando o feedback.

---

---

# VERSÃO 1 — Especificação Inicial

*Produzida pelo autor após análise dos documentos. Ainda não revisada.*

---

## V1.1 — Fontes de Dados: O que deve ser indexado

O assistente deve ser alimentado pelas seguintes fontes da NovaTech:

- **SharePoint corporativo:** todos os documentos PDF e Word classificados como ativos, excluindo versões anteriores de documentos que tenham sido formalmente substituídos.
- **Confluence (wiki interna):** todas as páginas ativas. Páginas arquivadas ou marcadas como obsoletas devem ser excluídas.
- **Pasta de rede:** planilhas de referência mensais (tabelas de SLA, tabelas de frete), sempre na versão mais recente disponível.

Documentos que **não devem ser indexados:**

- Versões antigas de procedimentos quando existir versão mais recente (ex: PROC-042 v1 deve ser excluída se a v2 for declarada vigente).
- Documentos sem identificação de responsável ou data de atualização.
- O FAQ-Atendimento em sua forma atual, por ser um documento não controlado e sem validação de Compliance.

## V1.2 — Documentos Contraditórios

Quando o assistente recuperar dois ou mais documentos com informações conflitantes sobre o mesmo tema, deve:

1. Apresentar as duas versões ao atendente, com indicação da data de cada uma.
2. Recomendar confirmação com a área responsável antes de repassar a informação ao cliente.
3. Não escolher uma versão por conta própria.

Quando houver uma versão mais recente identificável pela data, o assistente deve destacá-la como referência preferencial, mas sem suprimir a informação da versão anterior.

## V1.3 — Ausência de Resposta na Base

Quando a pergunta do atendente não tiver resposta na documentação indexada, o assistente deve:

1. Informar claramente que não encontrou a informação na base de documentos da NovaTech.
2. **Não** tentar responder com conhecimento geral do LLM — toda resposta deve ser baseada exclusivamente na documentação indexada.
3. Sugerir a área responsável pelo tema, quando possível.
4. Nunca dar uma resposta parcial como se fosse completa.

## V1.4 — Requisitos de Atualização

Quando novos documentos forem publicados nas fontes (SharePoint, Confluence, pasta de rede), devem estar disponíveis no assistente em até **24 horas** após a publicação.

Quando documentos forem removidos ou marcados como obsoletos, devem ser retirados do índice no próximo ciclo de atualização (máximo 24 horas).

## V1.5 — Requisitos de Rastreabilidade

Toda resposta do assistente deve obrigatoriamente incluir:

- Nome do documento de origem.
- Seção ou página relevante.
- Data da última atualização do documento.

Quando solicitado pelo atendente, o assistente deve exibir o trecho exato do documento que fundamentou a resposta.

---

---

# REVISÃO CLAUDE — Feedback sobre a V1

*O que segue é a análise crítica da V1, identificando gaps, ambiguidades e requisitos não testáveis.*

---

## Avaliação geral

A V1 estabelece uma boa estrutura e demonstra entendimento correto de que a qualidade do RAG depende da curadoria dos dados. Os cinco temas obrigatórios foram cobertos. No entanto, há **11 gaps e ambiguidades** que precisam ser resolvidos antes que os requisitos sejam testáveis.

---

## Gap 1 — "Documento ativo" não tem critério de definição

**Onde aparece:** V1.1 — "documentos classificados como ativos"  
**Problema:** O que define se um documento é "ativo"? Quem classifica? Onde essa classificação está registrada? Sem um critério objetivo, o pipeline de ingestão não saberá o que incluir ou excluir — e o QA não saberá o que testar.

**Pergunta não respondida:** Se um documento no SharePoint não tem nenhuma marcação de status (nem "ativo" nem "obsoleto"), o que acontece? Ele entra ou fica de fora?

**Exemplo real do cenário:** PROC-042 v1 não está marcada como obsoleta no SharePoint — ela simplesmente coexiste com a v2. A V1 da especificação diz que ela "deve ser excluída se a v2 for declarada vigente" — mas não define quem declara, como, onde e qual o prazo para que essa declaração seja reconhecida pelo pipeline.

---

## Gap 2 — Critério de "versão mais recente" não é inequívoco

**Onde aparece:** V1.1 e V1.2  
**Problema:** A especificação usa "versão mais recente" como critério de preferência, mas não define o que determina a recência: a data do documento? A data de upload no SharePoint? A data de última edição? No caso da PROC-042, a v2 tem data de emissão 10/11/2023, mas as disposições transitórias ainda mantêm a v1 válida para chamados anteriores a 01/12/2023. A "mais recente" não é simplesmente a "correta".

---

## Gap 3 — Contradição: critério de apresentação não é testável

**Onde aparece:** V1.2  
**Problema:** "Apresentar as duas versões ao atendente" é uma instrução comportamental sem critério de disparo preciso. Quando exatamente o assistente conclui que dois documentos são "contraditórios" sobre o mesmo tema? Se o score de similaridade de dois chunks for alto, mas o conteúdo não for contraditório, o assistente vai exibir as duas versões desnecessariamente? Se for baixo, pode perder contradições reais.

**Falta:** Um critério mensurável de detecção de contradição (ex.: mesmo metadado de tema/seção + valores numéricos diferentes) e um threshold mínimo para acionar o disclaimer.

---

## Gap 4 — FAQ não tem tratamento de transição

**Onde aparece:** V1.1 — "FAQ não deve ser indexado"  
**Problema:** A decisão de excluir o FAQ está correta, mas a V1 não endereça o que acontece com os **6 itens do FAQ que descrevem processos reais sem nenhum documento oficial** (seguro de carga, carga danificada, frete expresso + perigosa). Simplesmente não indexar o FAQ significa que esses processos ficam invisíveis para o assistente — e o atendente vai perguntar sobre eles. A especificação precisa definir o que o assistente faz nesses casos.

---

## Gap 5 — "Sugerir área responsável" requer dado que pode não existir

**Onde aparece:** V1.3  
**Problema:** Para o assistente sugerir a área responsável, essa informação precisa estar nos metadados de cada chunk (campo `area_responsavel`). A V1 assume que esse dado existe, mas não o torna um requisito explícito da indexação. Se os chunks não forem enriquecidos com esse metadado, a funcionalidade simplesmente não funciona — e não há como testar.

---

## Gap 6 — "24 horas" de atualização não define o relógio

**Onde aparece:** V1.4  
**Problema:** "Em até 24 horas após a publicação" é ambíguo em três dimensões:
1. **Horário:** se um documento for publicado às 23h de sexta-feira, as 24h incluem o fim de semana ou são 24h úteis?
2. **Gatilho:** o que conta como "publicado"? Upload no SharePoint? Aprovação por responsável? Notificação ao time de TI?
3. **Verificabilidade:** como o QA vai testar isso? Precisa de um log com timestamp de publicação vs. timestamp de disponibilidade no índice.

---

## Gap 7 — Documentos referenciados mas ausentes (PROC-043, PROC-088)

**Onde aparece:** Não aparece — **este gap está ausente da V1.**  
**Problema:** A análise documental identificou que PROC-043 e PROC-088 são citados em documentos oficiais mas não estão disponíveis na base. A especificação não define o que o assistente deve fazer quando perguntado sobre um tema cujo documento de referência existe (é citado) mas não foi indexado. Responder "não encontrei" pode ser enganoso — o documento existe, só não está disponível.

---

## Gap 8 — Rastreabilidade: "quando solicitado" é ambíguo

**Onde aparece:** V1.5 — "quando solicitado pelo atendente, o assistente deve exibir o trecho exato"  
**Problema:** Esta frase cria dois comportamentos diferentes — um padrão (só cita nome, seção e data) e um expandido (mostra o trecho). Porém não define como o atendente "solicita" o trecho, nem o que acontece quando o trecho é muito longo para exibição. Mais importante: para o RAG ser auditável, o trecho deveria estar sempre disponível — não só quando solicitado.

---

## Gap 9 — Ausência de requisito de qualidade mínima para indexação

**Onde aparece:** Não aparece — **este gap está ausente da V1.**  
**Problema:** A V1 não define o que acontece com documentos de baixa qualidade: PDFs escaneados sem OCR, planilhas com formatação quebrada, páginas Confluence com links mortos ou conteúdo incompleto. Esses documentos podem ser indexados e degradar a qualidade do retrieval sem que nenhum critério de rejeição os filtre.

---

## Gap 10 — Nenhum requisito de governança de curadoria

**Onde aparece:** Não aparece — **este gap está ausente da V1.**  
**Problema:** A especificação trata a curadoria da base como um evento (publicação → indexação), mas o cenário NovaTech tem três áreas diferentes atualizando documentos sem processo unificado. Não há requisito de quem aprova um documento antes de ele ser indexado, nem de quem pode remover um documento do índice. Sem isso, a base vai se deteriorar com documentos contraditórios ao longo do tempo.

---

## Gap 11 — Nenhum requisito de comportamento em falha técnica

**Onde aparece:** Não aparece — **este gap está ausente da V1.**  
**Problema:** O que o assistente exibe quando o serviço de LLM está indisponível? E quando o retrieval retorna zero resultados por erro técnico (não por ausência de informação)? A V1 cobre o cenário de "informação não encontrada na base" mas não distingue "não existe" de "erro técnico que impediu a busca".

---

## Resumo do feedback

| Gap | Categoria | Impacto no QA | Prioridade |
|-----|-----------|---------------|------------|
| 1 — Definição de "documento ativo" | Ambiguidade | Impede criação de critério de aceite para ingestão | Alta |
| 2 — Critério de "versão mais recente" | Ambiguidade | Impede teste de retrieval de versão correta | Alta |
| 3 — Detecção de contradição sem threshold | Não testável | Impossível verificar quando o disclaimer deve aparecer | Alta |
| 4 — FAQ: processos reais sem doc oficial | Gap de cobertura | Lacuna funcional — atendente vai perguntar | Alta |
| 5 — "Área responsável" como dado implícito | Gap de requisito | Funcionalidade quebrada sem metadado | Média |
| 6 — "24h" sem definição de relógio | Ambiguidade | Impede criação de SLA de atualização testável | Média |
| 7 — Docs citados mas ausentes (PROC-043, 088) | Gap de cobertura | Resposta "não encontrei" pode ser enganosa | Média |
| 8 — Rastreabilidade parcial | Ambiguidade | Auditabilidade incompleta | Média |
| 9 — Sem critério de qualidade mínima | Gap de requisito | Documentos ruins degradam o RAG silenciosamente | Média |
| 10 — Sem requisito de governança | Gap estrutural | Base se deteriora sem processo de curadoria contínua | Média |
| 11 — Sem requisito de falha técnica | Gap de cobertura | Comportamento em produção indefinido | Baixa |

---

---

# VERSÃO 2 — Especificação Refinada

*Incorpora todos os 11 gaps identificados na revisão. Cada requisito foi escrito para ser testável.*

---

## V2.1 — Fontes de Dados: O que deve (e não deve) ser indexado

### 2.1.1 Fontes incluídas

| Fonte | O que indexar | Critério de inclusão |
|-------|--------------|----------------------|
| SharePoint corporativo | PDFs e arquivos Word | Status = "Publicado" ou equivalente no campo de metadados do SharePoint. Documentos sem campo de status preenchido são tratados como **pendentes** e não indexados até que o responsável preencha o campo. |
| Confluence (wiki interna) | Páginas com status "Publicado" ou sem status explícito | Páginas com status "Arquivado", "Em revisão" ou "Obsoleto" são excluídas. |
| Pasta de rede | Planilhas de referência | Apenas o arquivo com data mais recente por categoria (ex.: tabela de SLA mais recente, tabela de frete mais recente). Versões anteriores não são indexadas. |

### 2.1.2 O que não deve ser indexado

- Documentos com mais de uma versão ativa no mesmo repositório **sem que haja uma nota formal de hierarquia entre elas** (ex.: PROC-042 v1 e v2 — ambas ficam fora do índice até que o responsável emita nota técnica declarando qual versão está em vigor e a partir de quando).
- Documentos sem responsável identificado e sem data de atualização preenchida.
- O FAQ-Atendimento em sua forma atual, por ser documento informal não validado por Compliance.
- Documentos que não passem nos critérios de qualidade mínima (ver 2.1.3).

### 2.1.3 Critério de qualidade mínima para indexação

Um documento só é elegível para indexação se atender a **todos** os seguintes critérios:

| Critério | Requisito mínimo |
|----------|-----------------|
| Legibilidade | Texto extraível eletronicamente (PDFs escaneados sem OCR aprovado são rejeitados). |
| Identificação | Contém título, responsável e data de atualização. |
| Integridade | Não contém seções marcadas como "a preencher", "TBD" ou equivalente em mais de 10% do conteúdo. |
| Formato | Arquivo não corrompido e abrível pelo parser correspondente ao tipo (PDF, DOCX, XLSX, HTML). |

Documentos rejeitados por qualidade mínima devem gerar **alerta automático** para o responsável da área de Operações de TI com identificação do arquivo e do critério não atendido.

### 2.1.4 Governança da base: quem pode incluir, alterar e remover

| Ação | Quem pode executar | Como é registrado |
|------|-------------------|-------------------|
| Incluir documento novo no índice | Publicação na fonte pelo responsável da área + pipeline automático | Log de ingestão com timestamp de publicação e timestamp de disponibilidade no índice |
| Marcar documento como obsoleto | Responsável da área, alterando o campo de status na fonte | Documento removido do índice no próximo ciclo. Log registra quem executou e quando. |
| Resolver conflito entre versões | Responsável da área + aprovação de ao menos uma das Diretorias (Comercial ou Operações) | Nota técnica assinada publicada no SharePoint como documento independente; pipeline reconhece automaticamente. |

### 2.1.5 Tratamento de documentos citados mas ausentes (PROC-043, PROC-088)

Quando um documento indexado citar outro documento que **não está disponível na base**, o assistente deve:

1. Responder com a informação disponível no documento indexado.
2. Acrescentar o aviso: *"Este tema faz referência ao documento [nome], que não está disponível na base atual. Para informações completas, consulte [área responsável]."*
3. **Não** indicar que "não há informação" — há informação parcial, e isso deve ser comunicado com precisão.

---

## V2.2 — Documentos Contraditórios

### 2.2.1 Definição operacional de contradição

Para fins deste sistema, dois trechos de documentos são considerados **contraditórios** quando:

- Referem-se ao mesmo tema (identificado pelo mesmo campo de metadado `tema` ou `tipo_documento`) **e**
- Apresentam pelo menos um valor numérico, prazo ou regra de negócio diferente (ex.: multiplicador regional diferente, prazo de entrega diferente, threshold diferente).

Diferenças de redação sem impacto em valores ou regras **não são consideradas contradição** para fins de disparo do disclaimer.

### 2.2.2 Comportamento esperado diante de contradição

Quando o assistente detectar contradição segundo o critério acima, deve:

1. Apresentar **ambas as versões**, identificando cada uma pelo nome do documento e data de atualização.
2. Indicar qual é a versão mais recente com base no campo `data_atualizacao` dos metadados.
3. Exibir o disclaimer: *"Encontrei versões divergentes sobre este tema. A versão mais recente é [documento, data]. Recomendo confirmar com [área responsável] antes de repassar ao cliente."*
4. **Não escolher uma versão como definitiva** — essa decisão cabe ao atendente ou ao especialista da área.

### 2.2.3 Critério de "versão mais recente"

A versão mais recente é determinada pelo campo `data_atualizacao` do metadado do chunk, **não** pela data de upload no repositório ou pela data de edição do arquivo. Esse campo deve ser preenchido obrigatoriamente no processo de ingestão. Em caso de empate de datas, o assistente apresenta ambas sem indicar preferência.

### 2.2.4 Contradição em temas de alto impacto

Para os seguintes temas, o disclaimer de contradição é **obrigatório** mesmo que a diferença seja pequena:

- Valores financeiros (multiplicadores de frete, descontos, penalidades, percentuais de seguro).
- Prazos com impacto em SLA contratual.
- Regras de elegibilidade que definem se o cliente tem ou não direito a um serviço.

---

## V2.3 — Ausência de Resposta na Base

### 2.3.1 Comportamento padrão: informação não encontrada

Quando o retrieval não retornar nenhum chunk com score de similaridade acima do threshold mínimo definido, o assistente deve:

1. Informar ao atendente: *"Não encontrei informação sobre este tema na documentação oficial da NovaTech."*
2. Indicar a área responsável pelo tema, quando o metadado `area_responsavel` estiver disponível: *"Para este tipo de questão, o contato indicado é [área/ramal]."*
3. **Nunca** usar conhecimento geral do LLM para preencher a lacuna.
4. **Nunca** apresentar uma resposta parcial como se fosse completa.

### 2.3.2 Distinção entre "não encontrado" e "erro técnico"

| Cenário | Mensagem ao atendente |
|---------|-----------------------|
| Informação não existe na base (resultado vazio ou abaixo do threshold) | *"Não encontrei informação sobre este tema na documentação oficial da NovaTech. Recomendo consultar [área responsável]."* |
| Erro técnico que impediu a busca (timeout, falha no serviço, LLM indisponível) | *"Não foi possível consultar a base de documentos no momento. Por favor, tente novamente em instantes ou consulte diretamente a documentação no SharePoint/Confluence."* |

### 2.3.3 Temas com FAQ mas sem documento oficial

Para temas que existem no FAQ mas não têm documento oficial indexado (ex.: seguro de carga, carga danificada em trânsito, frete expresso para cargas perigosas), o assistente deve aplicar o comportamento de "não encontrado" da seção 2.3.1 e encaminhar para a área responsável. O FAQ não é fonte válida de resposta.

---

## V2.4 — Requisitos de Atualização

### 2.4.1 SLA de disponibilidade de novos documentos

| Evento | SLA de disponibilidade no índice | Relógio |
|--------|----------------------------------|---------|
| Publicação de novo documento em dia útil (08h–18h) | Até 4 horas após publicação | Horas corridas a partir do timestamp de publicação na fonte |
| Publicação fora do horário comercial ou em fim de semana/feriado | Até 4 horas após o início do próximo dia útil (08h) | Relógio começa às 08h do dia útil seguinte |
| Remoção ou marcação como obsoleto | Até 24 horas corridas | Independente de dia ou horário |

### 2.4.2 Definição de "publicado"

Considera-se "publicado" o momento em que o documento recebe status "Publicado" na fonte original (SharePoint ou Confluence) ou é depositado na pasta de rede com nome de arquivo conforme a convenção definida pelo time de Operações. O pipeline de ingestão deve registrar esse timestamp em log auditável.

### 2.4.3 Verificabilidade do SLA de atualização

O cumprimento do SLA deve ser verificável por log contendo:

- Timestamp de publicação na fonte.
- Timestamp de ingestão no pipeline.
- Timestamp de disponibilidade no índice (primeiro momento em que o chunk aparece em resultados de busca).
- Status de resultado (sucesso, falha, rejeitado por qualidade mínima).

---

## V2.5 — Requisitos de Rastreabilidade

### 2.5.1 Citação obrigatória em toda resposta

Toda resposta do assistente, sem exceção, deve incluir ao final:

```
Fonte: [Nome do documento] — [Seção/Página] — Atualizado em [data_atualizacao]
```

Quando a resposta for baseada em mais de um chunk/documento, todas as fontes devem ser listadas.

### 2.5.2 Trecho de fundamentação

O trecho exato do documento que fundamentou cada afirmação deve estar sempre disponível no sistema de logs. Sua exibição ao atendente segue a seguinte regra:

- **Exibição automática:** quando a resposta envolver valores numéricos (prazos, multiplicadores, percentuais, thresholds de SLA).
- **Exibição sob demanda:** para respostas descritivas, o atendente pode solicitar o trecho com o comando "mostrar trecho" ou equivalente configurado na interface do Teams.
- **Tamanho máximo exibido:** 300 tokens. Trechos maiores são truncados com indicação de onde consultar o documento completo.

### 2.5.3 Auditabilidade

Cada interação deve ser registrada em log com os seguintes campos obrigatórios:

| Campo | Descrição |
|-------|-----------|
| `timestamp` | Data e hora da pergunta |
| `user_id` | Identificador do atendente (integrado ao Microsoft 365) |
| `pergunta` | Texto exato digitado pelo atendente |
| `chunks_recuperados` | IDs e scores de todos os chunks retornados pelo retrieval |
| `resposta` | Texto exato da resposta gerada |
| `fontes_citadas` | Lista de documentos citados na resposta |
| `tipo_resultado` | "resposta_fundamentada" / "nao_encontrado" / "contradicao_detectada" / "erro_tecnico" |

Logs devem ser retidos por no mínimo 12 meses e acessíveis para auditoria pelo time de Compliance.

---

## Tabela de Testabilidade — V2

| Requisito | Caso de teste | Critério de aprovação |
|-----------|---------------|----------------------|
| V2.1.1 — Inclusão por status | Publicar documento com status "Publicado" e verificar indexação em até 4h (horário comercial). | Documento aparece em resultados de busca dentro do SLA. |
| V2.1.2 — Exclusão de docs sem hierarquia | Com PROC-042 v1 e v2 ambas sem nota de hierarquia, perguntar sobre multiplicador regional. | Assistente não responde com valor de nenhuma versão; informa que o tema está pendente de formalização. |
| V2.1.3 — Rejeição por qualidade mínima | Submeter PDF escaneado sem OCR ao pipeline. | Pipeline rejeita o documento e gera alerta ao responsável. |
| V2.2.1/2.2.2 — Disclaimer de contradição | Indexar dois docs com multiplicador Norte diferente. Perguntar sobre frete para o Norte. | Assistente exibe ambos os valores, datas e disclaimer de confirmação. |
| V2.2.3 — Critério de versão mais recente | Dois docs com mesmo tema, datas diferentes nos metadados. | Assistente destaca o de data mais recente como referência preferencial. |
| V2.3.1 — Não encontrado | Perguntar sobre seguro de carga sem doc indexado. | Assistente responde "não encontrei" + área responsável. Não inventa valores. |
| V2.3.2 — Distinção erro técnico | Simular timeout do serviço de LLM. | Assistente exibe mensagem de erro técnico (não "não encontrei"). |
| V2.3.3 — FAQ não é fonte válida | Perguntar sobre seguro de carga (tem FAQ, não tem doc oficial). | Assistente responde "não encontrei" e não cita o FAQ. |
| V2.4.1 — SLA de atualização (horário comercial) | Publicar doc às 10h de dia útil. Verificar disponibilidade no índice. | Disponível até às 14h do mesmo dia. |
| V2.4.1 — SLA de atualização (fora do horário) | Publicar doc às 22h de sexta-feira. Verificar disponibilidade. | Disponível até às 12h da segunda-feira. |
| V2.4.3 — Log de atualização | Publicar novo documento e consultar log. | Log contém: timestamp de publicação, de ingestão, de disponibilidade e status. |
| V2.5.1 — Citação obrigatória | Fazer qualquer pergunta respondida com sucesso. | Resposta contém "Fonte: [nome] — [seção] — Atualizado em [data]". |
| V2.5.2 — Exibição automática de trecho | Perguntar sobre valor numérico (ex.: multiplicador de frete). | Trecho exato do documento é exibido sem solicitação adicional. |
| V2.5.3 — Campos de log | Executar 5 interações e consultar logs. | Todos os 7 campos obrigatórios presentes em cada registro. |

---

## Histórico de alterações V1 → V2

| Seção V2 | Origem da alteração | Gap resolvido |
|----------|---------------------|---------------|
| V2.1.1 | Novo critério de "status publicado" e tratamento de docs sem status | Gap 1 |
| V2.1.1 / V2.2.3 | Critério de "versão mais recente" por campo `data_atualizacao` | Gap 2 |
| V2.2.1 | Definição operacional de contradição com critério mensurável | Gap 3 |
| V2.3.3 | Temas de FAQ sem doc oficial → comportamento "não encontrado" | Gap 4 |
| V2.1.3 / V2.3.1 | Metadado `area_responsavel` como requisito explícito de ingestão | Gap 5 |
| V2.4.1 / V2.4.2 | SLA de 4h com definição de relógio e gatilho de "publicado" | Gap 6 |
| V2.1.5 | Comportamento para docs citados mas ausentes (PROC-043, PROC-088) | Gap 7 |
| V2.5.1 / V2.5.2 | Rastreabilidade: automática para valores numéricos, sob demanda para descritivos | Gap 8 |
| V2.1.3 | Critério de qualidade mínima para indexação | Gap 9 |
| V2.1.4 | Requisito de governança da base (quem inclui, altera, remove) | Gap 10 |
| V2.3.2 | Distinção entre "não encontrado" e "erro técnico" | Gap 11 |

---

*DB1 Group — Documento de uso interno · Junho 2025*
