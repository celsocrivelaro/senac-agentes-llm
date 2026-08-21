# Exercício 1 - Assistente de Entregas

## Contexto

Uma transportadora quer um assistente que converse com o atendente, entenda para onde e quando o pacote vai ser entregue, e devolva um resumo confiável da entrega — consultando dados reais, sem inventar endereço nem feriado.

Você vai construir esse assistente juntando as quatro ideias da aula 01: conversa com memória, saída estruturada, chamada de ferramentas e integração com API externa.

## Objetivo

Criar um programa `05-assistente-entregas.py` que:

1. Converse com o usuário até obter **CEP de destino** e **data pretendida de entrega**
2. Consulte o **endereço** real daquele CEP
3. Verifique se a data cai em **feriado**
4. Imprima um resumo final da entrega

## APIs que você vai usar

Ambas são públicas e não precisam de chave. Teste as duas no navegador antes de programar.

**Endereço por CEP**

```
https://brasilapi.com.br/api/cep/v2/01310100
```

```json
{
  "cep": "01310100",
  "state": "SP",
  "city": "São Paulo",
  "neighborhood": "Bela Vista",
  "street": "Avenida Paulista"
}
```

**Feriados nacionais do ano**

```
https://brasilapi.com.br/api/feriados/v1/2026
```

```json
[
  { "date": "2026-01-01", "name": "Confraternização mundial", "type": "national" },
  { "date": "2026-02-16", "name": "Carnaval", "type": "national" }
]
```

## Requisitos

### 1. Coleta dos dados (baseie-se no `01-chatbot.py` e no `03-integracao-software.py`)

- O assistente pergunta o CEP e a data de entrega, de forma amigável
- Mantém o histórico da conversa, para o usuário poder responder em partes
- Ao final da coleta, produz um JSON no formato:

```json
{
  "cep": "01310100",
  "data": "2026-02-16"
}
```

- O CEP deve sair **somente com números** e a data no formato **ISO 8601** (`yyyy-mm-dd`)
- O seu programa deve ler esse JSON com `json.loads` dentro de um `try/except` — se o modelo devolver algo inválido, o programa avisa e não quebra

### 2. Ferramentas (baseie-se no `04-integracao-api.py`)

Escreva duas funções Python e ofereça as duas ao modelo como ferramentas:

| Função | O que faz |
| --- | --- |
| `buscar_endereco(cep)` | Consulta a BrasilAPI e devolve rua, bairro, cidade e estado |
| `listar_feriados(ano)` | Devolve os feriados nacionais do ano informado |

Regras:

- Se o CEP não existir, a função devolve `{"erro": "..."}` em vez de estourar uma exceção — assim o modelo pode pedir o CEP de novo
- O agente deve ter um **limite de passos** (por exemplo, 8), para nunca entrar em laço infinito
- Imprima cada chamada de ferramenta na tela, como no exemplo da aula, para acompanhar o que o agente está fazendo

### 4. Resumo final

```
=== RESUMO DA ENTREGA ===
Endereço: Avenida Paulista, Bela Vista — São Paulo/SP
CEP: 01310-100
Data solicitada: 16/02/2026 (segunda-feira)
Situação: feriado nacional — Carnaval
Sugestão: entregar em 18/02/2026 (quarta-feira)
```

## Exemplo de interação

```
Bot: Olá! Para onde vamos enviar o pacote? Preciso do CEP de destino.
Usuário: é pra Avenida Paulista, CEP 01310-100
Bot: Anotado! E para qual data você quer agendar a entrega?
Usuário: dia 16 de fevereiro de 2026

[ferramenta] buscar_endereco({'cep': '01310100'})
[ferramenta] listar_feriados({'ano': 2026})

=== RESUMO DA ENTREGA ===
...
```

## Desafios opcionais

- Aceitar datas em linguagem natural (“sexta que vem”, “daqui a 10 dias”) — o modelo interpreta, o Python valida

## Entrega

- Arquivo `05-assistente-entregas.py` no Blackboard
- Um print (ou cópia do terminal) de uma execução completa, incluindo as linhas `[ferramenta]`
- Um caso de erro tratado: uma execução com CEP inválido em que o programa não quebra

## Dicas

- Comece pelas funções puras: faça `buscar_endereco` e `listar_feriados` funcionarem sozinhas, chamadas direto no Python, **antes** de conectar o modelo
- Descreva bem as ferramentas no campo `description` — é só isso que o modelo lê para decidir quando chamar cada uma
- Se o agente responder sem consultar nada, reforce no prompt de sistema: *“nunca invente dados; use sempre as ferramentas”*