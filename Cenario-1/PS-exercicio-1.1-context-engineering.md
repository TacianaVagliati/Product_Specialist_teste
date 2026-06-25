# Exercício 1.1 — Mapa de Cobertura Temática e Hipóteses de Gaps — NovaTech RAG

**Projeto:** Assistente RAG — Atendimento ao Cliente NovaTech  
**Documentos analisados:** 5 (POL-001, PROC-042 v1, PROC-042 v2, SLA-2024, FAQ-Atendimento)  
**Data de análise:** Junho 2025  
**Elaborado por:** QA — DB1 Group

---

## Legenda

| Símbolo | Significado |
|---------|-------------|
| ✅ | Coberto — documento oficial disponível |
| ⚠️ | Parcialmente coberto / presente apenas no FAQ (não controlado) |
| ❌ | Gap crítico — ausente da base documental oficial |
| 🔗 | Documento citado em outro doc, mas não disponível na base |

---

## 1. POL-001 — Política de Devolução

| Tema | Status | Referência |
|------|--------|------------|
| Prazo geral de devolução (7 dias úteis) | ✅ | POL-001 §3.1 |
| Exceções ao prazo padrão (perigosas, refrigeradas, lacre violado) | ✅ | POL-001 §3.2 |
| Procedimento de devolução (5 etapas — Portal + CT-e) | ✅ | POL-001 §3.3 |
| Custos de devolução (cliente vs. NovaTech) | ✅ | POL-001 §3.5 |
| Devolução parcial por volume | ✅ | POL-001 §3.4 |
| Carga danificada em trânsito | ❌ | Gap — processo real operado via FAQ (item 38), sem PROC oficial |
| Fluxo de sinistros / envolvimento do Jurídico | ❌ | Gap — FAQ cita sinistros@novatech.com.br, sem normalização |
| Interceptação de carga em trânsito | 🔗 | PROC-088 citado em POL-001 §2, mas não disponível na base |

---

## 2. PROC-042 — Frete Especial

| Tema | Status | Referência |
|------|--------|------------|
| Fórmula base de cálculo (Valor base × Multiplicador × Fator peso) | ✅ | PROC-042 v1 e v2 |
| Multiplicadores regionais (5 regiões) | ✅ ⚠️ | Presente em ambas versões — valores divergem (ver hipóteses de gap) |
| Fatores de peso por faixa (500–1000 / 1001–3000 / >3000 kg) | ✅ ⚠️ | Presente em ambas versões — valores divergem |
| Aprovação prévia para cargas acima de 5.000 kg | ✅ | Consistente entre v1 e v2 |
| Desconto de volume | ⚠️ | Presente, mas regras divergem entre v1 e v2 |
| Prazo de entrega adicional para frete especial | ✅ ⚠️ | Presente em ambas versões — +2 dias (v1) vs. +3 dias (v2) |
| Hierarquia formal entre versões (v1 vs. v2) | ❌ | Gap estrutural — nenhuma versão declara formalmente que substitui a outra |
| Fretes para cargas abaixo de 500 kg | ❌ | Gap — PROC-042 só se aplica acima de 500 kg; faixa inferior não coberta |
| Frete de cargas perigosas pesadas | 🔗 | PROC-043 citado em v1 e v2, mas não disponível na base |

---

## 3. SLA-2024 — Atendimento por Tier

| Tema | Status | Referência |
|------|--------|------------|
| Classificação de clientes (Gold / Silver / Standard) | ✅ | SLA-2024 §1 |
| Tempos de resposta e resolução por tier | ✅ | SLA-2024 §2 |
| Definição de incidente crítico (4 critérios) | ✅ | SLA-2024 §3 |
| Penalidades por descumprimento (crédito 5% / 10%) | ✅ | SLA-2024 §4 |
| Medição de SLA e horário comercial | ✅ | SLA-2024 §5 — Azure DevOps |
| Processo operacional de concessão de crédito | ❌ | Gap — SLA define os percentuais, mas não o fluxo de execução |
| SLA específico para cargas acima de 5.000 kg | ❌ | Gap — não há cruzamento entre SLA-2024 e PROC-042 para este cenário |

---

## 4. Temas sem nenhum documento oficial

| Tema | Status | Fonte atual (não oficial) |
|------|--------|--------------------------|
| Seguro de carga (0,3% padrão / 0,8% perigosa) | ❌ | FAQ item 22 |
| Regras de seguro para contratos pré-2023 | ❌ | FAQ item 22 |
| Frete expresso para carga perigosa (autorização Compliance) | ❌ | FAQ item 32 |
| Prazos de entrega por rota (ex.: Norte até 10 dias úteis) | ❌ | FAQ item 27 |
| Tier Platinum descontinuado (2022) | ⚠️ | FAQ item 15 — sem comunicado formal de extinção |
| Ramal 4500 — Gestão de Riscos (fluxo de acionamento) | ⚠️ | POL-001 §3.2 + FAQ item 3 — sem PROC de acionamento |

---

## 5. Hipóteses de Gaps — Priorização para o PS

### Prioridade Alta — Risco de alucinação ou resposta errada no RAG

| # | Gap | Por que é crítico para o RAG |
|---|-----|------------------------------|
| 1 | **Hierarquia PROC-042 v1 / v2** | Sem decisão formal de qual versão vale, o RAG não tem como responder com segurança sobre multiplicadores ou prazos de frete especial. Qualquer resposta sobre cálculo de frete será potencialmente incorreta. |
| 2 | **Carga danificada em trânsito** | Fluxo real operado pelo time (48h, sinistros@, Jurídico) não tem PROC. O RAG vai ou omitir a resposta ou baseá-la no FAQ não validado — gerando risco operacional e legal. |
| 3 | **Seguro de carga** | Tema completamente ausente da documentação oficial. Alta probabilidade de alucinação se o LLM tentar responder usando apenas o FAQ como referência. |
| 4 | **PROC-043 ausente** | Citada como obrigatória para cargas perigosas pesadas em ambas as versões da PROC-042. O RAG vai mencionar a referência sem ter o conteúdo — resposta incompleta em tema de segurança e compliance. |

### Prioridade Média — Gap funcional com impacto no atendimento

| # | Gap | Recomendação |
|---|-----|--------------|
| 5 | **PROC-088 ausente** | Solicitar o documento ao time de Operações antes da indexação. |
| 6 | **Fretes abaixo de 500 kg** | Confirmar se existe algum documento não compartilhado ou se a regra está embutida em outro PROC. |
| 7 | **Prazos de entrega por rota** | Verificar se existe tabela de prazos padrão por rota no SharePoint — dado crítico para atendimento. |
| 8 | **Processo de concessão de crédito (violação de SLA)** | SLA-2024 define os percentuais; falta o PROC operacional de como o crédito é gerado no sistema. |

### Prioridade Baixa — Risco controlável com disclaimer

| # | Gap | Observação |
|---|-----|------------|
| 9 | **Tier Platinum descontinuado** | FAQ cobre bem este caso. Um guardrail simples no prompt resolve. |
| 10 | **Contratos legados pré-2023** | Encaminhamento ao Comercial é a resposta correta — pode ser tratado como guardrail de escalação. |

---

## 6. Resumo Quantitativo

| Status | Quantidade |
|--------|------------|
| ✅ Temas com cobertura oficial completa | 14 |
| ⚠️ Temas parciais ou apenas no FAQ | 6 |
| ❌ Gaps críticos sem cobertura oficial | 7 |
| 🔗 Documentos citados e ausentes | 2 (PROC-043, PROC-088) |
| **Total de temas mapeados** | **29** |

---

> **Próximo passo recomendado:** Levar este mapa para a sessão de discovery com os responsáveis de Operações, Compliance e Comercial da NovaTech para confirmar a existência de documentos não compartilhados e priorizar a normalização dos gaps antes da indexação.

