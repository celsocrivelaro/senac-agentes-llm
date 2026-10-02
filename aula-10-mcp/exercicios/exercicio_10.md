# Exercício 10 — A integração do case vira servidor

## Contexto

A Parte 1 do trabalho pediu uma integração com software tradicional, admitindo
que fosse *mock*. A **Parte 3** exige que essa integração seja reescrita como
**servidor MCP**, com o ganho a demonstrar declarado no enunciado do trabalho: a
ferramenta deixa de ser código acoplado a um agente e passa a ser um serviço que
**os vários agentes do sistema** consomem — e que qualquer agente de fora
consumiria.

Ela chega lá, e não na Parte 2, porque é a arquitetura multiagente que cria o
critério da aula: MCP paga quando há **mais de um dono**.

Este exercício é o ensaio dessa entrega, e acrescenta a ela a parte que o
enunciado do trabalho não cobra: **a conta**. A Aula 10 estabeleceu que MCP tem
preço, que ele é pago na escolha de ferramenta, e que a decisão de adotá-lo se
justifica pelo número de consumidores — não pelo protocolo ser novo.

> **Questão a ser respondida ao final:** quantas ferramentas ficam declaradas
> depois de conectados os servidores do seu sistema, e quantas delas foram
> desativadas por causa desse número?

## Objetivo

Implementar, no repositório do trabalho:

1. um **servidor MCP** que exponha ao menos **uma ferramenta** e **um recurso**;
2. o **agente do case** consumindo esse servidor, sem alteração do laço;
3. um **script de contagem** das ferramentas declaradas, antes e depois;
4. a **decisão de adoção**, item a item, para as demais integrações do sistema.

```
   ANTES                              DEPOIS
   ┌──────────────┐                   ┌──────────────┐   ┌──────────────┐
   │ agente       │                   │ agente       │   │ servidor MCP │
   │  ├ laço      │                   │  ├ laço      │◀─▶│  ├ tool      │
   │  ├ estado    │                   │  ├ estado    │   │  └ resource  │
   │  └ ferramenta│  código acoplado  │  └ (cliente) │   └──────────────┘
   └──────────────┘                   └──────────────┘    consumível por
                                                          qualquer agente
```

## Requisitos

### 1. O servidor

Escolha **uma** das integrações do seu case — a que a Parte 1 implementou, ou
outra do mesmo sistema. Publique-a como servidor MCP, com:

- **ao menos uma ferramenta**, e ela deve ser a operação que o **modelo** decide
  invocar durante o laço;
- **ao menos um recurso**, e ele deve ser conteúdo que a **aplicação** decide
  incluir no contexto.

A separação entre os dois é o critério do §2 da nota 01, e a escolha precisa
estar justificada em comentário no código: *por que esta operação é ferramenta e
aquela é recurso?* Publicar como ferramenta algo que a aplicação sempre precisa
incluir é o defeito que o exercício procura.

### 2. A descrição, tratada como prompt público

Cada ferramenta precisa de uma descrição que responda três coisas:

- o que a ferramenta faz;
- o **formato exato** de cada argumento, com os valores válidos quando forem
  enumeráveis;
- **o que ela não faz** — a parte mais esquecida, e a que mais evita chamada
  indevida.

Registre, em `exercicios/aula-10-servidor-mcp.md`, **duas versões** da descrição de uma das ferramentas: a
primeira que você escreveu e a versão corrigida depois de observar o modelo
escolher errado. A diferença entre as duas, com o número de chamadas de cada
uma, é item de avaliação.

### 3. O erro, classificado

Toda ferramenta do servidor deve distinguir:

| Natureza | Como volta |
|---|---|
| erro de **domínio** (argumento inválido, registro inexistente) | como **conteúdo**, com o que errou, qual era o certo e o que fazer agora |
| erro de **protocolo ou infraestrutura** | como falha, tratada pelo código do cliente |

Um erro de domínio devolvido como falha de protocolo desaparece do contexto do
modelo, e ele repete o mesmo argumento até o orçamento acabar.

### 4. A escrita, idempotente

Se o servidor expõe alguma operação com efeito no mundo, ela precisa de **chave
de idempotência derivada do conteúdo** e de `ja_existia` no retorno. A razão
mudou em relação à Aula 05: aqui a proteção é contra clientes que **você não
escreveu e não controla**.

`uuid4()` a cada chamada não é chave de idempotência.

### 5. O agente, sem reescrita

Conecte o agente do case ao servidor. Demonstre, com `git diff` no relatório de
entrega, que **o laço, o objeto de estado, o orçamento, o motivo de término e o
detector de laço não mudaram**. Se algum deles mudou, explique por quê — pode ser
uma decisão legítima, mas precisa ser deliberada.

### 6. A medição

Escreva um script que imprima, para a configuração atual do seu sistema:

- número de ferramentas ativas;
- quantas delas o agente efetivamente chamou, no conjunto de tarefas do case;
- as famílias de nomes próximos entre servidores diferentes.

Rode-o em duas configurações: com **todos** os servidores que o seu sistema
poderia usar, e com o conjunto que você decidiu **manter ativo**. A diferença
entre os dois números é o resultado do exercício.

### 7. A decisão de adoção, para o resto do sistema

Liste todas as integrações do seu case em uma tabela:

| Integração | Consumidores | Sobrecarga / valor do trabalho | Vira MCP? | Por quê |
|---|---|---|---|---|

**Pelo menos uma delas deve ser recusada**, e a recusa precisa estar justificada
pelo critério da aula: MCP paga quando há mais de um dono, e não paga quando a
operação é de alta frequência e baixa complexidade. Um sistema em que tudo vira
MCP indica que o critério não foi aplicado.

### 8. O carimbo

O versionamento da Aula 03 continua valendo, e esta aula acrescenta dois campos:
**a revisão da especificação do protocolo** e **a versão do servidor consumido**.

A razão é diferente das anteriores. Nos casos da Aula 06 e da Aula 08, quem muda
o carimbo é quem roda o experimento. Num servidor de terceiro, **quem muda é
outra pessoa** — sem aviso, e sem que o resultado deixe de ser sintaticamente
válido.

Fixe a versão do servidor consumido, como se fixa qualquer dependência.

## O que deve sair na tela

```
SERVIDOR: <nome>
  ferramentas ..... <n>   (<lista>)
  recursos ........ <n>   (<lista de URIs>)
  revisão do protocolo: <AAAA-MM-DD>   versão do servidor: <x.y.z>

AGENTE
  linhas alteradas no laço ............ 0
  linhas alteradas no estado .......... 0
  linhas alteradas no orçamento ....... 0

FERRAMENTAS DECLARADAS
  configuração    ferram.   chamadas no conjunto de tarefas
  todos              <n>      <n>
  ativos             <n>      <n>
  desativadas        <n>

FAMÍLIAS DE NOME PRÓXIMO
  <prefixo>_*   <ferramentas que colidem>

DECISÃO DE ADOÇÃO
  <integração> ... MCP      (<n> consumidores)
  <integração> ... função   (1 consumidor, alta frequência)
```

## Desafios opcionais

**A.** Implemente a variante por **execução de código** sobre o mesmo servidor:
o agente descobre a função de que precisa e escreve o código que a chama, em vez
de receber todas as declarações. Reporte quantos registros deixaram de
atravessar o modelo no mesmo conjunto de tarefas, e descreva em uma frase o
risco que essa variante introduz.

**B.** Conecte o agente a um servidor MCP público de terceiro e conte quantas
ferramentas ele sozinho acrescenta. Descreva o que você precisaria verificar
antes de usá-lo num sistema real.

## Entrega

No repositório do trabalho:

- o servidor, o cliente e o script de medição, com **parâmetros e justificativas
  no código** — sem relatório à parte;
- em `exercicios/aula-10-servidor-mcp.md`: as duas versões da descrição, a tabela de decisão de adoção e o
  carimbo com os dois campos novos;
- o `git diff` que evidencia o requisito 5.

> **Onde isto reaparece:** a **Parte 3 do trabalho** pede `docs/mcp.md`. O que você escreve aqui é o rascunho dele — na entrega, consolidado em `docs/`.

## Dicas

- A ferramenta que você já tem provavelmente vira servidor com pouco mais que um
  decorador. O trabalho do exercício **não é** esse; é a descrição, o erro, a
  idempotência e a conta.
- Conte as declarações **antes** de decidir o que desativar. A ordem inversa
  produz uma decisão que não se consegue defender.
- Se todas as suas integrações passarem no critério de adoção, releia o
  critério.
