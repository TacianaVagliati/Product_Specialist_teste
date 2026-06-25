# Exercício 1.2 — Análise de Inconsistências — PROC-042 v1 × PROC-042 v2

**Documentos comparados:** PROC-042 v1.0 (mar/2023) × PROC-042 v2.0 (nov/2023)  
**Responsável:** Diretoria Comercial  
**Data de análise:** Junho 2025  
**Elaborado por:** QA — DB1 Group

---

## Resumo Executivo

| Categoria | Quantidade |
|-----------|------------|
| Conflitos numéricos (multiplicadores e fatores) | 7 |
| Conflitos de processo (regras de negócio) | 3 |
| Conflito estrutural crítico (hierarquia de versões) | 1 |
| Pontos consistentes entre as versões | 2 |
| **Total de pontos comparados** | **13** |

---

## 1. Conflitos Numéricos

### 1.1 Multiplicadores Regionais

Todos os multiplicadores aumentaram na v2. Nenhuma região foi mantida igual.

| Região | v1 (mar/2023) | v2 (nov/2023) | Δ absoluto | Δ % | Direção |
|--------|--------------|--------------|-----------|-----|---------|
| Sul | 1.2 | 1.3 | +0.1 | +8,3% | ↑ frete mais caro |
| Sudeste | 1.0 | 1.1 | +0.1 | +10,0% | ↑ frete mais caro |
| Centro-Oeste | 1.3 | 1.4 | +0.1 | +7,7% | ↑ frete mais caro |
| Nordeste | 1.4 | 1.5 | +0.1 | +7,1% | ↑ frete mais caro |
| Norte | 1.6 | 1.8 | +0.2 | +12,5% | ↑ frete mais caro |

### 1.2 Fatores de Peso

Os fatores de peso se moveram na direção oposta: a faixa base (500–1.000 kg) permaneceu igual, e as faixas superiores foram reduzidas na v2.

| Faixa de peso | v1 (mar/2023) | v2 (nov/2023) | Δ % | Direção |
|---------------|--------------|--------------|-----|---------|
| 500 – 1.000 kg | 1.0 | 1.0 | 0% | = sem alteração |
| 1.001 – 3.000 kg | 1.2 | 1.15 | −4,2% | ↓ fator reduzido |
| > 3.000 kg | 1.5 | 1.4 | −6,7% | ↓ fator reduzido |

> **Atenção:** Multiplicadores regionais sobem e fatores de peso descem — efeitos em direções opostas. O impacto líquido no frete depende sempre da combinação região + faixa de peso. Não existe uma resposta única de "ficou mais caro ou mais barato".

### 1.3 Exemplos de Cálculo Concreto

Fórmula: `Valor base × Multiplicador regional × Fator de peso`

**Carga de 2.000 kg, destino Norte (tarifa base hipotética: R$ 1.000,00)**

```
v1: 1.000 × 1.6 (Norte) × 1.2 (1–3t) = R$ 1.920,00
v2: 1.000 × 1.8 (Norte) × 1.15 (1–3t) = R$ 2.070,00
Diferença: +R$ 150,00 (+7,8%) — mesma carga, mesma rota
```

**Carga de 4.000 kg, destino Norte (tarifa base hipotética: R$ 1.000,00)**

```
v1: 1.000 × 1.6 (Norte) × 1.5 (>3t) = R$ 2.400,00
v2: 1.000 × 1.8 (Norte) × 1.4 (>3t) = R$ 2.520,00
Diferença: +R$ 120,00 (+5,0%) — mesma carga, mesma rota
```

**Carga de 2.000 kg, destino Sudeste (tarifa base hipotética: R$ 1.000,00)**

```
v1: 1.000 × 1.0 (Sudeste) × 1.2 (1–3t) = R$ 1.200,00
v2: 1.000 × 1.1 (Sudeste) × 1.15 (1–3t) = R$ 1.265,00
Diferença: +R$ 65,00 (+5,4%) — os efeitos se somam, não se cancelam
```

---

## 2. Conflitos de Processo

### 2.1 Prazo de Entrega Adicional

| Aspecto | v1 (mar/2023) | v2 (nov/2023) |
|---------|--------------|--------------|
| Dias adicionais ao prazo padrão da rota | +2 dias úteis | +3 dias úteis |
| Justificativa declarada | Manuseio de carga pesada | Manuseio e roteirização de carga pesada |

**Impacto:** Um atendente usando v1 informa prazo X. Outro usando v2 informa prazo X+1. Além do conflito de informação ao cliente, o prazo prometido afeta diretamente a medição de SLA (SLA-2024) — a versão usada determina se o chamado foi cumprido dentro do prazo contratual ou não.

---

### 2.2 Política de Desconto de Volume

Esta não é apenas uma diferença de número — é uma mudança de mecânica de negócio.

| Aspecto | v1 (mar/2023) | v2 (nov/2023) |
|---------|--------------|--------------|
| Threshold nível 1 | > 10 fretes/mês | ≥ 8 fretes/mês |
| Desconto nível 1 | Negociar com Comercial (sem % definido) | 5% automático sobre o multiplicador regional |
| Threshold nível 2 | Não existe | > 15 fretes/mês |
| Desconto nível 2 | Não existe | 10% automático sobre o multiplicador regional |
| Descontos maiores | Aditivo contratual | Aprovação da Diretoria Comercial |

**Exemplo concreto — cliente com 9 fretes especiais no mês:**
- Na v1: não tem desconto, precisa negociar com o Comercial.
- Na v2: tem 5% de desconto garantido automaticamente.

**Impacto:** A resposta que o atendente dá ao cliente é diametralmente oposta dependendo da versão usada. O FAQ (item 45) ainda agrava: usa o threshold da v1 (>10) mas descreve o mecanismo da v2 (desconto automático) — criando uma terceira variante informal que não corresponde a nenhum dos dois documentos.

---

### 2.3 Status da PROC-043 (Frete de Cargas Perigosas)

| Aspecto | v1 (mar/2023) | v2 (nov/2023) |
|---------|--------------|--------------|
| Referência à PROC-043 | "Cargas perigosas seguem tabela específica (PROC-043)" | Idem + nota: "PROC-043 está em processo de revisão pelo Compliance e pode sofrer alterações" |

**Impacto:** A v2 introduz incerteza explícita sobre um documento de referência obrigatório para cargas perigosas. A PROC-043 não está disponível na base atual — e a v2 indica que, mesmo se estivesse, seu conteúdo pode estar em mudança.

---

## 3. Conflito Estrutural Crítico — Hierarquia entre Versões

| Aspecto | v1 (mar/2023) | v2 (nov/2023) |
|---------|--------------|--------------|
| Declara substituição da versão anterior? | Não | Não |
| Indica obsolescência da versão concorrente? | Não | Não |
| Disposição transitória | Não existe | Chamados abertos antes de 01/12/2023 → usar v1; chamados novos → usar v2 |
| Status no SharePoint | Coexiste com v2 sem hierarquia clara | Coexiste com v1 sem hierarquia clara |

**Por que isso é crítico:** A seção "Disposições Transitórias" da v2 ainda valida a v1 para um subconjunto de chamados, sem definir um critério de encerramento. A partir de quando nenhum chamado pré-01/12/2023 estaria mais em aberto? Isso não está documentado — e, na prática, ambas as versões têm vigência simultânea sem que qualquer sistema ou pessoa saiba qual aplicar em cada caso.

---

## 4. Pontos Consistentes entre as Versões

| Aspecto | v1 | v2 | Status |
|---------|----|----|--------|
| Fator de peso para 500–1.000 kg | 1.0 | 1.0 | ✅ Igual |
| Aprovação prévia para cargas acima de 5.000 kg | Gerente de operações regional | Gerente de operações regional | ✅ Igual |

---

## 5. Impacto no Assistente RAG

### Cenário 1 — Pergunta sobre multiplicador regional

> "Qual o multiplicador para frete especial com destino ao Norte?"

Se ambas as versões estiverem indexadas, o RAG pode recuperar chunks das duas e gerar resposta contraditória (1.6 vs. 1.8), escolher arbitrariamente, ou mesclar os valores sem clareza. Sem metadado de versão nos chunks, o modelo não tem como identificar qual é mais recente.

### Cenário 2 — Pergunta sobre prazo de entrega

> "Qual o prazo adicional para frete especial acima de 500 kg?"

v1 → "2 dias úteis". v2 → "3 dias úteis". O atendente promete prazo incorreto ao cliente, com impacto direto no cumprimento do SLA contratual.

### Cenário 3 — Pergunta sobre desconto de volume

> "Meu cliente tem 9 fretes especiais este mês. Tem direito a desconto?"

- v1: Não — precisa negociar (threshold é >10).
- v2: Sim — 5% automático (threshold é ≥8).
- FAQ: Não ("mais de 10 fretes") + desconto automático (contradição interna).

O RAG vai responder "sim" ou "não" dependendo de qual versão dominar o retrieval — com impacto financeiro direto e potencial para contestação contratual pelo cliente.

---

## 6. Recomendações para o PS — Antes da Indexação

| Prioridade | Ação | Responsável sugerido |
|------------|------|----------------------|
| 🔴 Alta | Emitir nota técnica ou aditivo declarando que a PROC-042 v2 substitui integralmente a v1 a partir de 01/12/2023. | Diretoria Comercial |
| 🔴 Alta | Retirar a PROC-042 v1 do SharePoint ou marcá-la como "obsoleta" para que o pipeline de ingestão não a indexe. | TI / Operações |
| 🟡 Média | Garantir que os chunks da v2 carreguem metadados de versão ("versão: 2.0", "vigência: 01/12/2023") para permitir filtragem e citação correta. | Engenharia RAG |
| 🟡 Média | Solicitar ao Compliance o status atual da PROC-043 — se está em vigor ou foi substituída. | PS / Compliance |
| 🟢 Baixa | Incluir guardrail no system prompt para sinalizar ao atendente quando a resposta envolve PROC-042 e recomendar confirmação com o Comercial até que a hierarquia seja formalizada. | PS (Prompt Engineering) |

---

> **Observação para o QA:** Todos os casos de teste relacionados a frete especial (cálculo de valor, prazo de entrega, desconto de volume) devem ser bloqueados ou marcados como "não testável com confiabilidade" até que a decisão de vigência entre v1 e v2 seja formalizada. Testar com a base atual produziria resultados não determinísticos.
