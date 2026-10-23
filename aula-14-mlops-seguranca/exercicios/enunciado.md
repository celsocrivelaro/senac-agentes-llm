# Exercício 14 — Atacar o próprio sistema, e depois defendê-lo

## Contexto

A Parte 3 do trabalho cobra segurança com uma pergunta direta: *o que o
sistema faria se a entrada fosse hostil?*

Até a Aula 13, essa pergunta era hipotética — o sistema rodava na máquina de
quem o escreveu. Agora ele tem endpoint público, porta aberta, segredo em
plataforma de terceiro, uma imagem construída sobre uma base que outra pessoa
publica e dependências que outra pessoa atualiza. A pergunta deixou de ser
hipotética.

> **Questão a ser respondida ao final:** quanto de utilidade as defesas do
> grupo custaram — e com base em qual número o grupo decidiu que valia a
> pena?

## Objetivo

1. **Mapear** os dez riscos ao próprio código, com arquivo e linha;
2. **Executar** ao menos dois ataques indiretos contra o próprio sistema;
3. **Implementar** dois padrões de defesa, declarando o custo em capacidade;
4. **Integrar** uma suíte hostil à suíte da Aula 11, com as taxas separadas.

## Requisitos

### 1. O mapa, com endereço

| | Risco | Arquivo e linha | Existe no sistema do grupo? |
|---|---|---|---|
| ASI01 | Sequestro de objetivo | | |
| … | | | |
| ASI10 | Agentes desgovernados | | |

**Endereço**, não descrição. Um risco marcado como "existe" sem uma linha de
código apontada é palpite. E um risco marcado como "não existe" precisa da
razão — normalmente, a capacidade correspondente não está no sistema.

ASI07 e ASI08 têm endereço mesmo num sistema de agente único: o resumo que
um sub-agente devolve (Aula 05) e a ferramenta que falha no meio da execução.
Se o sistema não tiver nenhum dos dois, declare isso; a Aula 16 volta aos dois
riscos quando o sistema for dividido em vários agentes.

### 2. Dois ataques indiretos, executados

Escolha **dois canais distintos** entre documento recuperado, retorno de
ferramenta, memória e agente subordinado. Para cada um:

- o *payload*, e por que ele é plausível para um classificador;
- o canal de entrada;
- o ***trace* que evidencia a obediência** — não a resposta, o *trace*.

Se um dos canais escolhidos for a **memória**, o ataque precisa demonstrar o
efeito **em execução posterior**, e não na mesma. É a distância temporal que
importa: o atacante não está presente, e o usuário afetado não é ele.

### 3. As três defesas que não funcionam

Antes de implementar as que funcionam, **demonstre que as outras não
funcionam** no sistema do grupo: delimitador, instrução defensiva e — se o
sistema tiver — classificador de moderação.

Três execuções e três resultados. Se alguma delas resistir, investigue a
causa antes de concluir que ela defende: normalmente é o *payload* que está
fraco.

### 4. A trifecta do sistema

| Perna | Existe? | Onde | Removível? |
|---|---|---|---|
| dado privado | | | |
| conteúdo não confiável | | | |
| comunicação para fora | | | |

Identifique **qual é removível sem inviabilizar o produto**, e remova-a. Se
nenhuma for, isso é um **risco aceito** — ver o requisito 7.

### 5. Dois padrões de defesa, com o custo declarado

Implemente dois dos seis. Para cada um, o relatório precisa trazer:

- o que ele impediu, com o ataque do requisito 2 executado contra ele;
- **o que o sistema deixou de conseguir fazer** — a lista, explícita.

A segunda parte é metade da nota. Uma defesa apresentada sem ela parece
grátis, e não é.

### 6. A suíte hostil

No mínimo **dez casos hostis**, integrados à suíte da Aula 11 e com a mesma
disciplina: conjunto versionado, execução repetida, origem declarada.

E a suíte passa a reportar **duas taxas separadas**:

```
                      utilidade    resistência
  sem defesa              <n>%          <n>%
  + <defesa 1>            <n>%          <n>%
  + <defesa 2>            <n>%          <n>%
```

Reportá-las num único número esconde a troca. **Cada ataque bem-sucedido do
requisito 2 vira caso permanente** desta suíte. A Aula 16 roda a mesma suíte
sobre a versão multiagente do sistema; os casos precisam, portanto, ser
executáveis sem depender de detalhe interno do agente único.

### 7. Um risco aceito, declarado

Escolha ao menos um risco que o grupo **decidiu não mitigar** e declare-o:

- qual é o risco;
- **por que foi aceito** — normalmente, porque mitigá-lo inviabiliza um
  requisito do produto;
- **o que o limita hoje**;
- **que evento faria reavaliar**.

Aceitar risco declarado é engenharia. Não declarar é omissão — e um trabalho
sem nenhum risco aceito quase sempre significa que a análise não foi feita.

### 8. A cadeia de suprimentos e a imagem

Liste as dependências de terceiro que o agente **consome em execução** —
a imagem base do `Dockerfile`, servidores MCP, ferramentas externas,
extensões:

| Dependência | Versão fixada? | Descrição revisada? | Assinatura? |
|---|---|---|---|

A coluna do meio é a que quase ninguém preenche, e é a que corresponde ao
mecanismo: **a descrição da ferramenta entra no contexto do modelo, e
portanto é entrada não confiável vinda de terceiro**.

Releia o `Dockerfile` da Aula 13 como superfície e responda, com a linha
correspondente:

| Verificação | Resposta | Linha |
|---|---|---|
| a imagem base tem versão fixada (*tag* específica ou *digest*)? | | |
| o processo roda com usuário sem privilégio (`USER`)? | | |
| só a porta do serviço é exposta e publicada? | | |
| alguma camada contém arquivo com segredo? | | |

Corrija o que não estiver conforme. Se alguma camada já publicada continha
segredo, a correção não é reconstruir a imagem: é **revogar a chave**.

### 9. A operação de recuperação

Demonstre que o sistema consegue **se recuperar** de um envenenamento de
memória: identifique o registro contaminado e remova-o seletivamente.

Se essa operação não existir, o sistema não se recupera — cada execução
futura repete o comportamento injetado. Implemente-a.

### 10. O carimbo

Esta aula acrescenta **o conjunto de casos hostis**, ao lado do *golden
dataset* da Aula 11.

## O que deve sair na tela

```
MAPA ASI x CÓDIGO
  ASI01  <arquivo>:<linha>   existe
  ...

ATAQUES EXECUTADOS
  <canal> ... <payload resumido> ... OBEDECEU
    trace: <trajetória que evidencia>

DEFESAS QUE NÃO FUNCIONAM
  delimitador ............ OBEDECEU
  instrução defensiva .... OBEDECEU
  classificador .......... passou (<score>)

TRIFECTA
  <perna> ... <onde> ... <removível?>
  removida: <qual>

DEFESAS IMPLEMENTADAS
  <padrão> ... impediu <ataques>
    o sistema deixou de conseguir: <lista>

SUÍTE
                    utilidade   resistência
  sem defesa           <n>%        <n>%
  + <defesa>           <n>%        <n>%

RISCO ACEITO
  <id> ... <risco> ... limitado por <...> ... reavaliar se <...>

CADEIA DE SUPRIMENTOS E IMAGEM
  <dependência> ... versão <fixa?> ... descrição <revisada?>
  Dockerfile ... base <fixa?> ... USER <sim/não> ... portas <...> ... segredo em camada <sim/não>

RECUPERAÇÃO
  registro contaminado removido: <id>   execuções seguintes: limpas
```

## Desafios opcionais

**A.** Implemente um **terceiro padrão** de defesa e meça, na suíte da Aula
12, **quanto de utilidade ele custou**. É o dado que falta em quase toda
discussão de segurança de agentes, e o desafio existe para produzi-lo.

**B.** Se o sistema do grupo usa sub-agente (Aula 05), ataque pelo **quarto
canal**: envenene o conteúdo que o sub-agente lê e verifique se a instrução
atravessa o resumo que ele devolve ao principal. Depois mostre que um resumo
**estruturado** — schema em vez de prosa — a bloqueia, e justifique. A Aula
14 retoma o mesmo ataque no *handoff* entre agentes.

## Entrega

No repositório do trabalho:

- os ataques, as defesas e a suíte hostil, com **parâmetros e justificativas
  no código**;
- em `exercicios/aula14-respostas.md`: o mapa ASI × código, os dois ataques
  com o *trace*, a análise da trifecta, a lista do que cada defesa custou em
  capacidade, a tabela da cadeia de suprimentos e da imagem e o registro do
  risco aceito;
- a suíte integrada, reportando as duas taxas.

> **Onde isto reaparece:** a **Parte 3 do trabalho** pede `docs/seguranca.md`. O que se escreve aqui é o rascunho dele — na entrega, consolidado em `docs/`. E a **Aula 16** roda a suíte hostil sobre a versão multiagente do sistema.

## Dicas

- Ataque **antes** de defender. Defesa escolhida sem ataque executado
  costuma proteger o flanco errado.
- O *payload* que funciona é **educado**. Um *payload* com aparência de
  ataque é barrado pelo classificador, e a conclusão — errada — passa a ser
  a de que a defesa funciona.
- O requisito 5 — o que o sistema deixou de conseguir fazer — é o mais fácil
  de deixar em branco e o que mais separa quem entendeu a aula.
- Se o sistema não obedeceu a nenhum ataque, quase certamente o problema é o
  *payload*, e não a segurança do sistema.
