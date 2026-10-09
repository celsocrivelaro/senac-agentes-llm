# Exercício 11 — A suíte do case

## Contexto

A Parte 3 do trabalho cobra o conjunto de avaliação e a pergunta que o
enunciado formula assim: *como vocês sabem que uma mudança melhorou em vez de
piorar?* Até esta aula, o grupo não tinha como responder.

O sistema do case já mudou muitas vezes. Cada uma dessas mudanças foi
defendida com um número pontual ou com argumento — e nenhuma foi verificada
contra as anteriores. Este exercício constrói o instrumento que faltava.

> **Questão a ser respondida ao final:** qual foi o ruído de base do seu
> sistema, e quantas das mudanças que vocês já promoveram teriam passado pela
> regra que vocês vão escrever?

## Objetivo

Construir, no repositório do trabalho:

1. o ***golden dataset*** do case, com origem declarada por caso;
2. a avaliação nos **níveis que o case exige**, escolhidos com justificativa;
3. um **juiz LLM validado** contra rótulos humanos;
4. a **suíte de regressão** rodando contra duas versões do sistema, com a
   regra de promoção declarada antes do resultado.

## Requisitos

### 1. O dataset, com origem declarada

No mínimo **vinte casos**. Cada um com, além da entrada e do esperado:

- `origem`: `producao`, `desenvolvimento` ou `sintetico`;
- `classe`: o agrupamento que permite ler por fatia;
- `por_que_existe`: por que este caso está no conjunto.

**Exigências de composição**, e valem mais que o tamanho:

- **casos de recusa.** Um sistema que deveria dizer "não sei" e diz outra
  coisa falha de um jeito que nenhum caso positivo detecta;
- **ao menos três classes distintas**, para que a leitura por fatia exista;
- **casos sintéticos revisados.** Se houver sintéticos, declare quem revisou.
  Um conjunto sintético não revisado mede a imaginação do gerador.

Se o seu case ainda não tem produção, use os erros observados durante o
desenvolvimento — e diga isso, em vez de rotular tudo como sintético.

### 2. Os níveis, escolhidos com justificativa

Monte a tabela das propriedades que importam no seu sistema:

| Propriedade | Nível escolhido | Por que não no nível mais barato |
|---|---|---|

A última coluna é a que vale nota. Toda propriedade colocada no juiz precisa
de justificativa de por que `==`, expressão regular ou validação de schema
não resolveriam.

Se o seu sistema **age sobre o mundo**, a avaliação de **resultado** é
obrigatória: comparar o estado persistido antes e depois. Ela captura duas
classes que nenhuma avaliação de texto vê — o agente que relata ter feito o
que não fez, e o que fez duas vezes.

Se o seu sistema tem **trace** (e ele tem, desde a Aula 05), a avaliação de
**trajetória** também é obrigatória, com as três medidas separadas:
cobertura, chamadas supérfluas e ordem.

### 3. O juiz, e a validação dele

Escreva um juiz para **uma** propriedade sem gabarito do seu case. Ele
precisa de rubrica explícita, saída estruturada, critério binário e exemplos
dos dois lados.

Depois, **valide-o**:

- rotule à mão **ao menos trinta casos**, com uma nota escrita por caso;
- meça a concordância e **exiba a matriz**, não só o agregado;
- declare o **limiar de adoção** antes de ver o resultado;
- **leia os desacordos um a um** e classifique cada um: defeito do juiz,
  defeito da rubrica, ou defeito do rótulo humano.

O relatório dessa leitura é item de avaliação. Um juiz adotado sem ela
produz números que sobem enquanto o sistema piora.

### 4. A suíte, com repetição

Duas versões do sistema — pode ser uma mudança real que o grupo pretende
fazer —, cada caso executado **ao menos duas vezes**, e o relatório
apresentado **por fatia** além da média.

### 5. O ruído de base

Antes de comparar as duas versões, rode a **versão A duas vezes** e reporte a
diferença entre elas. Esse é o seu ruído de base, e ele precisa aparecer no
relatório.

### 6. A regra de promoção, escrita antes

Declare, **antes de olhar o resultado**, a regra que decide se a mudança
sobe. Ela deve tratar de duas coisas:

- o delta médio em relação ao ruído de base;
- a regressão por classe, ainda que a média suba.

Escrever a regra depois é escolher o critério que confirma o que já se queria
fazer, e o exercício procura exatamente isso.

### 7. O carimbo

Esta aula acrescenta **a versão do dataset** e **a versão da rubrica do
juiz**. Exiba o carimbo completo do seu sistema no relatório da suíte —
todos os campos acumulados desde a Aula 03.

Se algum campo do seu carimbo não existir porque o seu case não usa aquela
peça, diga isso explicitamente em vez de omitir a linha.

## O que deve sair na tela

```
DATASET v<x.y.z>   <n> casos
  por origem:   producao <n>   desenvolvimento <n>   sintetico <n>
  por classe:   <classe> <n>  ...

VALIDAÇÃO DO JUIZ (rubrica v<x.y.z>)
                    humano: sim   humano: não
    juiz: sim            <n>          <n>
    juiz: não            <n>          <n>
  concordância: <n>%   limiar declarado: <n>%   VEREDITO: <adotar|não>
  desacordos: <n>  (juiz <n> · rubrica <n> · rótulo <n>)

RUÍDO DE BASE
  versão A, execuções: <n>%, <n>%   dispersão: <n> pontos

SUÍTE
  classe            A       B     delta
  <classe>        <n>%    <n>%    <+/-n>
  MÉDIA           <n>%    <n>%    <+/-n>

DECISÃO: <promover | não promover>
  regra aplicada: <a regra escrita no requisito 6>

CARIMBO
  <campo> ... <valor>     (13 linhas)
```

## Desafios opcionais

**A.** Meça a concordância de **dois juízes de modelos diferentes** entre si e
com o humano. Discuta qual dos três desacordos é mais informativo — e o que
significa quando os dois juízes concordam entre si e discordam do humano.

**B.** Pegue uma mudança que o grupo **já promoveu** no passado e rode a suíte
retroativamente contra ela. Se a mudança não teria passado pela regra,
descreva o que ela quebrou.

## Entrega

No repositório do trabalho:

- o dataset, os avaliadores, o juiz e a suíte, com **parâmetros e
  justificativas no código**;
- em `exercicios/aula-11-suite-do-case.md`: a tabela de níveis com a justificativa por propriedade, a
  matriz de concordância com a leitura dos desacordos, e a regra de promoção
  **datada** — para evidenciar que é anterior ao resultado;
- o carimbo completo, versionado junto com o dataset.

> **Onde isto reaparece:** a **Parte 3 do trabalho** pede `docs/evals.md`. O que você escreve aqui é o rascunho dele — na entrega, consolidado em `docs/`.

## Dicas

- Construa o dataset **antes** de escrever qualquer avaliador. A ordem
  inversa produz um conjunto que mede o que é fácil medir.
- Trinta rótulos à mão levam menos tempo do que parece, e são a parte do
  exercício com maior retorno.
- Se a sua suíte não tem casos de recusa, ela não está avaliando o
  comportamento mais perigoso do seu sistema.
- Se o delta entre as duas versões for menor que o ruído de base, **esse é um
  resultado**, e um bom. Reporte-o.
