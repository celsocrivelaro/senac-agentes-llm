# Exercício 5 — A arquitetura do agente do seu trabalho

## Contexto

Esta aula abriu cinco padrões e um agente. O exercício é escolher **quais deles o seu case exige** — e isso se decide no papel, antes da primeira linha de código.

O agente aqui é **o do seu trabalho**. Não é um exemplo, não é um domínio inventado: é o sistema que você vai construir ao longo do semestre.

> **Escrever código é opcional.** Se ajudar a pensar, escreva. O que se avalia é a arquitetura.
>
> Se você quiser sentir no código o que cada padrão cobra antes de escolher, o [exercicio_05.md](exercicio_05.md) implementa uma arquitetura já desenhada, num domínio dado.

## O que fazer

Uma arquitetura inicial do seu agente, em **três camadas**, e o **esboço do fluxo** que as liga.

### 1. Entrada

O que chega ao sistema.

- **O quê**: texto livre, formulário, arquivo, evento de outro sistema?
- **De onde, e quem dispara**: o usuário procura o sistema, ou o sistema acorda sozinho?
- **Quão heterogêneo**: quantos tipos diferentes de entrada existem, e em que proporção?

A última pergunta é a que mais decide. Se tudo que chega é da mesma natureza, **não há o que rotear** — e um *Router* ali seria custo sem contrapartida. Estime a proporção: *"~70% são consulta simples, ~20% reclamação"* já basta.

### 2. System

O que existe em volta do modelo.

- **O *system prompt***, em uma frase: o que este agente é, e o que ele não é;
- **As ferramentas**: o que cada uma faz, se é leitura ou escrita, e se é reversível;
- **O estado**: o que precisa sobreviver de um passo ao seguinte — objetivo, trajetória, o que já foi escrito, o que já falhou;
- **O orçamento**: um número para cada teto — passos, tokens, tempo.

Números chutados são aceitáveis. **Teto ausente não é.**

### 3. Processamento

Como a entrada vira saída — e **qual padrão em cada etapa**.

Cinco a oito etapas, com o padrão marcado ao lado. Nem toda etapa tem padrão: entrada e retorno costumam não ter.

## O esboço do fluxo

O produto do exercício. Em diagrama ASCII ou em lista, no formato:

```
1. ENTRADA     mensagem de texto livre do solicitante          [—]

2. TRIAGEM     classifica em 4 rotas                     [ROUTER]
                 simples  -> regra em código, sem LLM
                 ambíguo  -> segue para 3
                 fora     -> recusa
                 nenhuma  -> fila humana

3. ANÁLISE     consulta o registro e compara      [AGENTE, 8 passos]

4. PARECER     redige e verifica contra critério  [AVALIADOR, 3 rodadas]

5. REGISTRO    grava o parecer                 [ESCRITA - idempotente]

6. RETORNO     devolve ao solicitante                          [—]
```

Duas regras:

**Todo padrão com autonomia vem com o teto ao lado.** Orquestrador sem `MAX_SUBTAREFAS`, avaliador sem `MAX_RODADAS` e agente sem orçamento estão incompletos — e o número que você escrever ali é o primeiro orçamento do seu sistema.

**Marque as escritas**, e se são reversíveis. Escrita irreversível sem confirmação humana é uma decisão que você está tomando agora, mesmo sem perceber.

## O que sai de cada etapa

O esboço diz **quem decide**. Falta a outra metade: **o que cada etapa produz**.

Para cada etapa do seu fluxo, uma linha com o que entra e o que sai:

```
2. TRIAGEM
   entra:  {"texto": "o pedido 48219 não chegou"}
   sai:    {"rota": "reclamacao", "justificativa": "..."}

3. ANÁLISE
   entra:  a rota + os dados do pedido vindos do sistema
   sai:    {"veredito": "atraso confirmado", "dias": 13, "artigo": "..."}

6. RETORNO
   sai:    o texto que o solicitante lê, em até 3 linhas
```

E, no fim, **o que o usuário efetivamente vê** — a saída final, escrita como ela vai aparecer. Uma frase de exemplo basta.

### Por que isto vale metade do exercício

**É a única coisa que todas as arquiteturas têm em comum.** O padrão muda quem decide — o código, o modelo, o avaliador —, mas **toda etapa, em todo padrão, recebe alguma coisa e devolve alguma coisa**. É por esse contrato que as etapas se encaixam, e é ele que sobrevive quando você troca o padrão de uma delas.

Três consequências práticas, e a terceira é a que costuma surpreender:

**Etapa cuja saída você não sabe escrever é etapa que você ainda não projetou.** Se o item 3 do seu fluxo sai como *"a análise"*, ele não existe — falta dizer análise de quê, em que formato.

**Sem saída declarada, não há teste.** A Aula 11 vai avaliar este sistema, e ela avalia a saída de cada etapa. O que você escrever aqui é o embrião daquele conjunto de avaliação.

**A saída de uma etapa é a entrada da seguinte — e é aí que o sistema quebra.** Se a triagem devolve `rota` e a análise espera `categoria`, o desenho está errado e você descobre agora, no papel, em vez de na semana 10.

> Repare no que **não** deve atravessar as etapas: o texto bruto que o usuário escreveu, repetido em toda chamada. Se a etapa 2 já extraiu o que importa, a 3 recebe o extraído — não a mensagem inteira de novo.

## E uma justificativa curta

Ao lado do esboço, para cada padrão escolhido, **duas linhas**: por que ele, e por que o padrão mais simples não resolveria.

A tabela de decisão da [nota 01-6](../notas-de-aula/01-6-qual-padrao-usar.md) é lida de cima para baixo, parando no primeiro que resolve. Mostre onde você parou.

> A regra da disciplina é **usar a menor autonomia que resolve**. Se o seu esboço não tem nenhuma etapa que exija decisão em tempo de execução, o seu sistema é um *workflow* — e isso é uma resposta legítima, desde que você diga onde a decisão entra quando ele crescer.

## Entrega

Um documento em `docs/arquitetura-v1.md`, no repositório do trabalho.

O `v1` é proposital: esta arquitetura vai mudar quando encontrar o código, e a **Parte 2 do trabalho** pede a versão que sobreviveu, com o que mudou e por quê. Manter as duas lado a lado é o que torna a mudança visível.

## Dicas

- **Comece pela entrada.** Se ela for homogênea, metade das dúvidas sobre *Router* desaparece antes de você tê-las.
- **Escreva a saída final antes do meio.** Sabendo o que o usuário precisa ler, as etapas que faltam aparecem sozinhas — e as que sobram também.
- **Se você não consegue escrever a saída de uma etapa, ela ainda não é uma etapa.** É um nome.
- **O esboço cabe numa folha.** O que não couber numa folha provavelmente não cabe no semestre.
- **Se todas as etapas forem `[AGENTE]`, releia a tabela de decisão.** O erro de arquitetura mais comum não é escolher o padrão errado — é escolher um padrão onde nenhum era necessário.
- **Não tente acertar.** É a `v1`, e ela existe para estar errada de um jeito que você consiga enxergar depois.
