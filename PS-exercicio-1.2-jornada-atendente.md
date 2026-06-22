# Exercício 1.2 — Design de Jornada com Componente de IA
**Papel:** Product Specialist | **Cenário:** 1 | **Programa:** Trilha AI First — DGS/DB1

---

## Parte 1 — Jornada Textual (elaborada com o Claude)

### Contexto de produto

Com base no discovery:
- 45 atendentes, 320 chamados/dia (~192 com consulta a documentação)
- Dúvidas mais comuns: prazos de entrega (35%), frete (25%), devolução (20%), outros (20%)
- 15% dos chamados escalam para supervisor por falta de resposta
- Meta: reduzir de 12 para <2 minutos de busca por chamado

---

### Fluxo Principal — Caminho Feliz

**Trigger:** Atendente recebe dúvida do cliente via telefone, chat ou e-mail.

1. **Atendente formula a pergunta** no campo de busca do assistente (integrado ao Teams).
   - Exemplo: "Qual o prazo de devolução para cliente Gold que recebeu carga errada?"

2. **Assistente processa a query** (RAG em background):
   - Converte pergunta em embedding
   - Recupera chunks mais relevantes do Azure AI Search
   - Gera resposta usando os chunks como contexto

3. **Assistente exibe resposta** com:
   - Resposta em linguagem natural, português formal
   - Citação da fonte (ex: "POL-001, seção 3.5 — última atualização: 15/01/2024")
   - Trecho relevante do documento original (expandível)
   - Nível de confiança (alto / médio — baseado em score de similaridade dos chunks)

4. **Atendente avalia a resposta** (2-5 segundos):
   - Se coerente: usa no atendimento e encerra o fluxo
   - Se dúvida: vai para o Fluxo de Fallback
   - Se incorreta/incompleta: vai para o Fluxo de Feedback

5. **Registro automático:** a interação é logada com ID do atendente, ID do chamado, query, resposta e ação tomada.

---

### Fluxo de Fallback — Quando o Assistente Não Tem Confiança

**Trigger:** Assistente não encontrou chunks suficientemente relevantes (score abaixo do threshold) ou a pergunta envolve múltiplos domínios com informações contraditórias detectadas.

1. **Assistente exibe mensagem de baixa confiança:**
   > "Não encontrei uma resposta com confiança suficiente nesta documentação. Possíveis fontes relacionadas: [lista de 2-3 documentos parcialmente relevantes]. Recomendo verificar manualmente ou escalar para o supervisor."

2. **Assistente sugere a ação seguinte** (uma das três):
   - "Reformule a pergunta com mais detalhes (ex: tipo de carga, região, tier do cliente)"
   - "Consulte o documento [X] diretamente — link para o SharePoint"
   - "Escale para o supervisor — este chamado pode exigir julgamento humano"

3. **Atendente decide:**
   - Reformula a pergunta → volta ao Fluxo Principal com nova query
   - Consulta o documento diretamente → sai do assistente
   - Escala para supervisor → registra no sistema de chamados como "escalado por ausência de resposta do assistente"

**Guardrail aplicado neste fluxo:**
> **G1 — Guardrail de ausência de resposta:** O assistente NUNCA deve inventar um prazo, valor ou procedimento que não esteja nos chunks recuperados. Se não encontrar, declara explicitamente a ausência e sugere a fonte humana. Nunca responde com "provavelmente" ou "geralmente" para informações que dependem de documentação específica da NovaTech.

---

### Fluxo de Feedback — Quando a Resposta Está Errada

**Trigger:** Atendente identifica que a resposta do assistente está incorreta, desatualizada, incompleta ou contraditória com o que ele sabe.

1. **Atendente aciona o botão "Resposta incorreta"** (sempre visível abaixo de cada resposta).

2. **Mini-formulário de feedback** (máx. 30 segundos de preenchimento):
   - Motivo: [ ] Informação errada [ ] Informação desatualizada [ ] Incompleta [ ] Contraditória
   - Campo livre (opcional): "O correto é..."
   - Indicação da fonte correta (opcional): campo para colar link do SharePoint

3. **Registro automático** no sistema de qualidade:
   - Query original + resposta exibida + feedback do atendente + fonte indicada
   - Tag automática: `feedback_tipo: erro_factual` / `erro_versao` / `gap_documentacao`

4. **Roteamento do feedback:**
   - Feedbacks do tipo `erro_versao` → alerta automático para o Product Specialist (dashboard de qualidade)
   - Feedbacks do tipo `gap_documentacao` → fila de revisão para Compliance/Operações
   - Feedbacks recorrentes (>3 no mesmo tema em 48h) → notificação imediata para Product Owner

5. **Ciclo de correção:**
   - Compliance/Operações atualiza o documento no SharePoint
   - Pipeline de ingestão re-indexa o documento (SLA: máximo 24h após publicação)
   - Assistente passa a usar os chunks atualizados
   - O atendente que reportou recebe notificação: "A documentação sobre [tema] foi atualizada."

**Guardrail aplicado neste fluxo:**
> **G2 — Guardrail de carga perigosa:** O assistente NUNCA deve informar prazo, custo ou procedimento de devolução para cargas classificadas como perigosas (classes 1-6 ANTT) sem redirecionar explicitamente para a Gestão de Riscos (ramal 4500). Mesmo que o atendente reformule a pergunta, o guardrail permanece ativo para qualquer query que mencione "carga perigosa", "ANTT", "classe [1-6]" ou substâncias das classes listadas.

---

## Guardrails de Comportamento do Assistente

| # | Guardrail | Gatilho | Comportamento esperado |
|---|-----------|---------|----------------------|
| G1 | Ausência de resposta | Score de similaridade < threshold ou ausência de chunks relevantes | Declara ausência, sugere fonte, não inventa |
| G2 | Carga perigosa | Menção a ANTT, classes 1-6, explosivos, inflamáveis, tóxicos | Redireciona para Gestão de Riscos (ramal 4500), não informa prazo de devolução |
| G3 | Tier inexistente | Menção a "Platinum", "Diamond" ou qualquer tier não reconhecido | Informa que o tier não existe e lista os tiers válidos (Gold, Silver, Standard) |
| G4 | Documentos contraditórios detectados | Chunks de versões diferentes do mesmo documento com valores conflitantes | Apresenta ambas as versões com data, avisa sobre contradição, orienta a confirmar com supervisor |

---

## Parte 2 — Diagrama Visual (Claude Design)

O diagrama abaixo representa os 3 fluxos da jornada. Foi gerado usando Claude Design a partir da jornada textual acima.

*(Nota: o arquivo SVG/PNG do diagrama gerado pelo Claude Design está disponível como anexo: `PS-ex12-diagrama-jornada.svg`)*

### Descrição estrutural do diagrama para referência:

```
[INÍCIO]
    │
    ▼
[Atendente recebe dúvida do cliente]
    │
    ▼
[Atendente digita pergunta no assistente (Teams)]
    │
    ▼
[RAG: embedding → busca → chunks → LLM]
    │
    ├──── Score ALTO ────────────────────────────┐
    │                                            ▼
    │                               [Resposta com fonte e confiança ALTA]
    │                                            │
    │                               ┌────────────┴────────────────┐
    │                               │                             │
    │                        [Atendente usa]             [Atendente duvida]
    │                               │                             │
    │                        [Encerra chamado]         [Aciona "Incorreta"]
    │                                                             │
    ├──── Score BAIXO ───────────────────────────┐               │
    │                                            ▼               ▼
    │                               [FALLBACK: "Não encontrei"]  │
    │                               [Sugestão: reformule /       │
    │                                consulte doc / escale]      │
    │                                            │               │
    │                               ┌────────────┤               │
    │                               │            │               │
    │                        [Reformula]  [Escala supervisor]    │
    │                               │                            │
    │                        [Volta ao início]                   │
    │                                                            │
    └───────────────────────────────────────────────────────────►│
                                                                 ▼
                                             [FEEDBACK: mini-formulário]
                                             [Motivo + fonte correta (opcional)]
                                                                 │
                                                                 ▼
                                             [Roteamento automático]
                                             ┌──────────────────┤
                                             │                  │
                                    [Erro versão:        [Gap documentação:
                                     alerta PS]           fila Compliance]
                                             │                  │
                                             └──────────┬───────┘
                                                        ▼
                                             [Documento atualizado no SharePoint]
                                                        │
                                                        ▼
                                             [Pipeline re-indexa (SLA: 24h)]
                                                        │
                                                        ▼
                                             [Atendente notificado: "Atualizado"]
                                                        │
                                                        ▼
                                                    [FIM]
```

---

## Notas sobre o uso do Claude no exercício

**Iteração 1:** Pedi ao Claude para elaborar a jornada em texto a partir dos dados do discovery. Output inicial tinha apenas caminho feliz e um fallback genérico.

**Refinamento:** Pedi especificamente para incluir:
- O fluxo de feedback com roteamento explícito (não só "botão de feedback")
- Guardrails nomeados e específicos ao domínio de logística
- Indicação de como cada feedback fecha o loop no pipeline de RAG

**Iteração 2:** Claude expandiu o fluxo de feedback com ciclo de correção documentado. Adicionei manualmente os guardrails G3 e G4 (tier inexistente e documentos contraditórios) após a análise do exercício 1.1 — o modelo não os havia incluído espontaneamente.

**Uso do Claude Design:** A jornada textual foi fornecida ao Claude Design para geração do diagrama de fluxo visual. O diagrama foi ajustado para mostrar os 3 caminhos com cores distintas (azul = principal, amarelo = fallback, vermelho = feedback) e eliminar jargão técnico (ex: "RAG" substituído por "busca na documentação" no diagrama para uso com o cliente).
