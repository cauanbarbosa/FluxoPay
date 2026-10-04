# FluxoPay
Criação de um Dashboard Financeiro Comercial da FluxoPay 

```markdown
# 🚀 FluxoPay — Business Intelligence & Financial Analytics

[![SQL](https://img.shields.io/badge/Language-SQL-blue.svg)](#)
[![PowerBI](https://img.shields.io/badge/Tool-Power%20BI-yellow.svg)](#)
[![Status](https://img.shields.io/badge/Status-Completed-success.svg)](#)

## 📌 Sobre o Projeto

A **FluxoPay** é uma solução de Business Intelligence desenvolvida para fornecer visibilidade estratégica sobre a saúde financeira e operacional da instituição. O projeto abrange desde o monitoramento da **inadimplência e recuperação de crédito** até a **análise de perfil de risco dos clientes** e o **diagnóstico de cancelamentos de contratos (churn)**.

A arquitetura de dados foi estruturada utilizando consultas SQL avançadas (CTEs, condicionais e agregação de dados) para tratar dados relacionais operacionais e alimentar **3 Dashboards Interativos** no Power BI.

---

## 💻 Arquitetura de Dados & Consultas SQL

A camada analítica foi construída em SQL para consolidação de tabelas, engenharia de recursos (*feature engineering*) e preparação dos modelos de dados.

### 🛠️ 1. Consolidação e Agregação do Histórico de Inadimplência
Esta consulta unifica as tabelas operacionais (`inadimplencia`, `contratos` e `clientes`), agrupando o histórico por contrato e calculando os totais recuperados, em aberto e a severidade da faixa de atraso.

```sql
create table inadimplencia_dash AS
with base as (
    select
        t2.id_contrato,
        t3.id_cliente,
        t3.nome,
        t3.cidade,
        t3.estado,
        t3.renda_mensal,
        t3.score_credito,
        t3.segmento_cliente,
        t2.produto_financeiro,
        t2.valor_contratado,
        t2.prazo_meses,
        t2.status_contrato,
        count(t1.id_ocorrencia) as qtd_ocorrencias,
        sum(case when t1.recuperado = 'Sim' then 1 else 0 end) as qtd_recuperadas,
        min(t1.data_ocorrencia) as primeira_ocorrencia,
        max(t1.data_ocorrencia) as ultima_ocorrencia,
        max(case t1.faixa_atraso
            when '1-30 dias'  then 1
            when '31-60 dias' then 2
            when '61-90 dias' then 3
            when '90+ dias'   then 4
        end) as ordem_pior_atraso,
        round(sum(t1.valor_em_aberto), 2) as valor_em_aberto,
        round(sum(t1.valor_recuperado), 2) as valor_recuperado
    from inadimplencia as t1
    inner join contratos as t2 on t1.id_contrato = t2.id_contrato
    inner join clientes  as t3 on t2.id_cliente  = t3.id_cliente
    group by t2.id_contrato
)
```
Engenharia de Recursos (*Feature Engineering*) & Tabela Analítica
Calcula métricas derivadas diretamente no banco de dados, incluindo o saldo líquido pendente, o status da recuperação e a classificação do risco por faixa de score.

```sql
select
    *,
    round(valor_em_aberto - valor_recuperado, 2) as saldo_pendente,
    case ordem_pior_atraso
        when 1 then '1-30 dias'
        when 2 then '31-60 dias'
        when 3 then '61-90 dias'
        when 4 then '90+ dias'
    end as pior_faixa_atraso,
    case
        when qtd_recuperadas = 0               then 'Não recuperado'
        when qtd_recuperadas = qtd_ocorrencias then 'Recuperado'
        else 'Parcialmente recuperado'
    end as situacao_recuperacao,
    case
        when score_credito <= 500 then 'Baixo'
        when score_credito <= 799 then 'Médio'
        else 'Alto'
    end as faixa_score
from base;
```

Extração e Filtro de Contratos Cancelados (Churn)
Consulta dedicada à extração e ordenação cronológica da massa de contratos cancelados para a análise de motivos de cancelamento e evasão de receita.

```sql
select 
    t1.id_contrato,
    t1.id_cliente,
    t2.nome,
    t2.renda_mensal,
    t2.cidade,
    t2.estado,
    t2.score_credito,
    t2.segmento_cliente,
    t1.data_contratacao,
    t1.produto_financeiro,
    t1.valor_contratado
from contratos as t1
left join clientes as t2
on t1.id_cliente = t2.id_cliente
where t1.status_contrato = 'Cancelado'
order by t1.data_contratacao;
```

**Estrutura dos Dashboards & Principais Insights**

### **Dashboard de Inadimplência & Recuperação de Crédito**

<img width="1337" height="750" alt="image" src="https://github.com/user-attachments/assets/3ca86e16-dc99-42a3-bfd6-e96c7474a23a" />

* Focado no acompanhamento do saldo em aberto, evolução temporal da inadimplência e taxa de sucesso na cobrança.
* **Desempenho por Produto:** O *Empréstimo Pessoal* registra o maior volume em aberto (US\$ 3,2M) e recuperado (US\$ 1,1M), seguido pelo *Crédito Consignado* (US\$ 1,6M em aberto / US\$ 0,6M recuperado) e *Cartão de Crédito Parcelado* (US\$ 1,5M em aberto / US\$ 0,5M recuperado).
* **Evolução Temporal:** Acompanhamento contínuo da curva histórica (2022–2026) da relação entre saldo pendente e valor recuperado.

### **Dashboard de Perfil da Carteira & Risco de Crédito**

<img width="1330" height="749" alt="image" src="https://github.com/user-attachments/assets/061891eb-1280-4467-a85a-30a8b89785bd" />

* Mapeia a distribuição da carteira ativa, relacionando renda mensal, capacidade de tomador de crédito e score.
* **Matriz de Renda vs. Contrato:** Análise de dispersão que correlaciona o valor liberado por contrato com a renda declarada do cliente.
* **Participação por Produto:** O *Empréstimo Pessoal* representa US\$ 7,09M (40,98%) do volume contratado, seguido por *Crédito Consignado* com US\$ 5,02M (28,99%), *Cartão de Crédito Parcelado* com US\$ 2,68M (15,46%) e Antecipação de Recebíveis* com US\$ 2,52M (14,57%).
* **Principais Praças:** Destaque para Brasília (US\$ 1,1M), Ribeirão Preto (US\$ 0,8M) e Santos (US\$ 0,8M).

### **Dashboard de Contratos Cancelados (Churn)**

<img width="1330" height="744" alt="image" src="https://github.com/user-attachments/assets/c965099c-2a24-44f2-945b-c5c0c156048b" />

* Investiga a taxa de evasão de contratos e identifica os segmentos e regiões com maior impacto financeiro.
* **Distribuição por Produto:** O maior volume cancelado está concentrado em *Empréstimo Pessoal* (US\$ 3,15M / 38,24%) e *Crédito Consignado* (US\$ 2,64M / 32,11%).
* **Análise por Segmento:** O segmento de *Varejo* representa a maior fatia cancelada (US\$ 3,7M / 529 contratos), seguido pelo *Consignado Público* (US\$ 1,7M / 241 contratos) e *Consignado Privado* (US\$ 1,4M / 187 contratos).
* **Principais Cidades:** Brasília lidera o valor de cancelamento (US\$ 388,02K), seguida por Sorocaba (US\$ 353,70K), Campinas (US\$ 273,45K) e Santos (US\$ 266,05K).

### **Conclusões & Ações Recomendadas**
**Reforço na Régua de Cobrança:** Priorizar o produto *Empréstimo Pessoal* nas réguas de acionamento inicial para reduzir o saldo pendente.
**Ajuste na Política de Crédito:** Restringir alçadas automáticas no segmento *Varejo* para clientes com score de crédito abaixo de 500.
**Plano de Retenção Regional:** Focar estratégias antichurn nas praças de Brasília e Sorocaba, que somam os maiores volumes de cancelamento.
