# card-application-flow-challenge

Desafio proposto para implementação de um workflow de contratação de cartão.

### Principais etapas da workflow

1. Entrada da Proposta:

- Após login na aplicação o cliente inicia o fluxo de solicitação de cartão e confirma os dados da proposta (renda, tempo de conta, investimentos).

- Escolhe as ofertas disponíveis validadas pela api **eligibility-service**
- Escolhe os benefícios desejados entro os disponíveis para a oferta (aplicando as regras de exclusão para o perfil do cliente).

### 2. Componentes e validações

### 2.1 Entrada de dados (requisição da proposta)

```
{
    "clienteId": "83a4a660-028c-4a81-9507-1ac4b749a123",
    "renda": 50000.00,
    "investimentos": 10000.00,
    "dataAberturaConta": "2020-01-01",
    "ofertaSelecionada": "C",
    "beneficiosSolicitados": ["cashback", "salaVip", "seguroViagem"]
}
```

### 2.2 Regras para eligibilidade da oferta

| Oferta | Condições|
|---|---|
| A | renda > 1000 |
| B | renda > 15000 e investimentos > 5000 |
| C | renda > 50000 e (hoje - dataAberturaConta) > 2 anos |

### 2.3 Regras de elegibilidade dos benefícios

| Benefício | Regra |
|---|---|
| Cashback | Não permitido se beneficio "pontos" for solicitado |
| Pontos | Não permitido se benefício "cashback" for solicitado |
| SeguroViagem | Apenas oferta C |
| SalaVip | Apenas ofertas B ou C |

### 2.4 Ativação

1. O cliente pode solicitar benefícios, mas o sistema só ativa os **belegíveis** pela oferta e que não conflitam.
Ex: Oferta C, solicitou cashback e pontos -> sistema aplica regra de negócio: **cashback e pontos são mutuamente exclusivos, o cliente deve escolher apenas um**.
2. Benefícios não elegíveis para a oferta são ignorados automáticamente, com notificação ao cliente.

### 2.5 Registro de eventos e histórico

| Campo | Descrição |
|--- | --- |
|clientId | UUID |
|ofertaEscolhida | A/B/C |
|statusProposta | elegivel/inelegivel/erro_validacao/criado |
|beneficiosAtivados | lista de beneficios |
|beneficiosNegados | lista de beneficios negados (ex: "cashback conflito ponto") |
|dataHora | timestamp |
|dadosSensiveisHash | hash dos dados de (renda, investimentos, dados pessoais) utilizando kms para criptografia e decriptografia |

### 3. Tratamento de dados sensiveis

- Não logar em texto puro.
- Persistir apenas hash para auditoria ou mudanças em ambiente criptografado (KMS) com acesso controlado.
- Dados pessoais, oferta, beneficios: armazenamento em bancos isolados com politica de retenção.

### 4. Response ao cliente (status da proposta)

- [201] - Sucesso.

```
{
    "propostaId": "83a4a660-028c-4a81-9507-1ac4b749a123",
    "status": "cartao_aprovado",
    "cartaoId": "card_789",
    "beneficiosAtivados": ["salaVip", "seguroViagem"],
    "mensagem": "Seu cartão com oferta C foi aprovado, com benefícios salaVip e seguroViagem ativados. cashback não disponível para esta oferta."
}
```

- [200] - Inelegivel.

```
{
    "status": "inelegivel",
    "motivo": "Renda insuficiente para oferta C (mínimo R$50.000,00)"
}
```

### 5. Workflow

![Workflow de contratação](/images/workflow.jpg)
