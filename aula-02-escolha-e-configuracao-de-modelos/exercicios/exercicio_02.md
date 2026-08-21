# Exercício 2 - O Extrator do Turno da Noite

## Contexto

A transportadora do exercício anterior colocou um extrator em produção. Todas as noites, às 3h, um script lê as mensagens que os clientes mandaram durante o dia, extrai categoria, número do pedido e um resumo, e grava tudo numa planilha para o time de atendimento abrir os tickets de manhã.

Funciona. **Quase.**

Toda manhã aparecem linhas em branco na planilha — em média um quinto delas. Não há erro nenhum no log: o script termina com código 0 e diz que processou tudo. Quando alguém pega a mensagem que ficou em branco e roda de novo, na mão, às vezes funciona. Às vezes vem diferente da vez anterior. Às vezes fica em branco de novo.

O estagiário que escreveu o script foi embora. Você recebeu a tarefa com um recado do gerente: *"não quero saber de trocar de modelo, quero esse script confiável."*

**A boa notícia:** todos os problemas desse script são de **configuração da chamada**, e você aprendeu todos eles na aula 02. Não falta modelo melhor, não falta prompt melhor, não falta framework. Falta engenharia.

## O caso

Este é o script que está em produção, exatamente como está:

```python
# extrator_noturno.py — versão em produção (com problemas)
import json
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()

client = OpenAI(
    base_url="https://api.mistral.ai/v1",
    api_key=os.environ.get("OPENAI_API_KEY"),
)

PROMPT = """Extraia da mensagem do cliente:
- categoria: entrega_atrasada, endereco_errado, produto_avariado, duvida ou elogio
- pedido: o número do pedido, somente dígitos, ou null se não houver
- resumo: uma frase curta descrevendo o problema

Responda em JSON.

Mensagem: {mensagem}"""


def extrair(mensagem):
    try:
        resposta = client.chat.completions.create(
            model="mistral-small-latest",
            messages=[{"role": "user", "content": PROMPT.format(mensagem=mensagem)}],
            temperature=1.4,
            max_tokens=60,
            frequency_penalty=1.9,
        )
        return json.loads(resposta.choices[0].message.content)
    except Exception:
        return {}          # "pra não derrubar a fila da madrugada"


for mensagem in MENSAGENS:
    print(extrair(mensagem))
```

Ele tem **cinco** problemas. Todos os cinco estão nas notas 02 e 03 desta aula.

## Objetivo

Entregar um `08-extrator-confiavel.py` que faz a mesma coisa, mas de forma confiável — e um `diagnostico.md` que **prova**, com números, que cada correção resolveu algo.

O que se avalia aqui não é o script final: é o **diagnóstico**. Consertar por tentativa e erro até parar de dar erro não vale nota. Você tem que saber dizer qual configuração causava qual sintoma, e mostrar a medição que sustenta a afirmação.

## Os dados

Use estas 8 mensagens. O gabarito é o que o extrator deveria ter produzido:

```python
# (mensagem, categoria correta, pedido correto)
MENSAGENS = [
    ("Meu pedido 48219 era pra chegar terça e até hoje nada. Já são 5 dias!",
     "entrega_atrasada", "48219"),

    ("bom dia, o entregador deixou na rua de tras, numero 45. o meu é 145",
     "endereco_errado", None),

    ("A caixa do pedido 77310 chegou toda amassada e a tampa do liquidificador trincou",
     "produto_avariado", "77310"),

    ("vocês entregam no sábado?",
     "duvida", None),

    ("Só pra dizer que o rapaz da entrega foi super educado, obrigada!",
     "elogio", None),

    ("Pedido 90021 cancelado e recomprado como 90455, o 90455 não chegou",
     "entrega_atrasada", "90455"),

    ("PEDIDO 12 MIL 340 ENTREGUE NO CEP ERRADO, EU MORO NO 04567890 E FOI PRO 04567089",
     "endereco_errado", "12340"),

    ("Recebi o pedido n 55.102 com a tela rachada. Quero trocar.",
     "produto_avariado", "55102"),
]

CATEGORIAS = ["entrega_atrasada", "endereco_errado", "produto_avariado", "duvida", "elogio"]
```

Repare que quatro delas têm armadilha: número que **não** é pedido (número de casa, CEP), número **por extenso**, número **com ponto** e **dois** pedidos na mesma mensagem. Regra de desempate: quando há mais de um número de pedido, vale aquele de que o cliente está reclamando.

## Como medir

Toda afirmação sua tem que vir com número. Rode as 8 mensagens **3 vezes** (24 saídas) e meça três coisas:

| Métrica | Definição |
| --- | --- |
| **Válidas** | deu `json.loads`, é um objeto e tem as 3 chaves (`categoria`, `pedido`, `resumo`) — de 24 |
| **Corretas** | é válida **e** a categoria e o pedido batem com o gabarito — de 24 |
| **Estáveis** | mensagens em que as 3 execuções deram a mesma categoria e o mesmo pedido — de 8 |

**Estáveis** é a métrica que explica o bilhete do gerente. Um extrator que devolve resultados diferentes para a mesma mensagem é pior que um extrator que erra sempre igual: o erro consistente você descobre e trata, o inconsistente aparece uma vez a cada cinco madrugadas.

## Requisitos

### 1. Linha de base

Faça o script rodar como está (você vai precisar acrescentar o laço de medição — o `MENSAGENS` original não tinha gabarito) e registre as três métricas. Esse é o seu ponto de partida, e ele **tem** que estar no relatório: sem linha de base não existe prova de melhora.

Guarde também as saídas cruas dessa primeira rodada. Você vai citá-las como evidência.

### 2. Diagnóstico

Para cada um dos cinco problemas, escreva no `diagnostico.md`:

- **qual configuração** está errada;
- **qual sintoma** ela causa (o que aparece na planilha de manhã);
- **por que** ela causa isso — em uma ou duas frases, com a seção da nota como referência;
- **a evidência**: uma saída crua da linha de base em que dá para ver o problema acontecendo.

Um deles não é um parâmetro do modelo, e é o que transforma três dos outros quatro em silêncio. Ache esse primeiro.

### 3. Correção incremental — a parte central

Corrija **uma coisa por vez**, medindo depois de cada mudança, e monte esta tabela:

```
versão                              válidas  corretas  estáveis
0. como está em produção              14/24     9/24      2/8
1. + ...                              ..../24   ..../24   ..../8
2. + ...                              ..../24   ..../24   ..../8
3. + ...                              ..../24   ..../24   ..../8
4. + ...                              ..../24   ..../24   ..../8
5. + ...                              24/24    22/24      8/8
```

(os números são ilustrativos — os seus vão ser outros)

**Uma mudança por versão.** Se você corrigir tudo de uma vez, o script fica bom e você não aprendeu nada: não sabe qual mudança valeu 10 pontos e qual valeu 1. É essa tabela que responde a pergunta do gerente, e é ela que vale a nota.

A ordem em que você corrige é sua decisão — mas pense nela. Uma das cinco correções, feita primeiro, faz as outras quatro ficarem **visíveis**; feita por último, você passa o exercício inteiro trabalhando às cegas.

### 4. A versão final

O `08-extrator-confiavel.py` precisa, além das cinco correções:

- **checar `finish_reason`** e tratar `"length"` como falha, não como resposta;
- **não engolir exceção**: quando uma mensagem falhar, registrar qual mensagem, qual erro, e continuar as outras — a falha tem que **aparecer** no log da madrugada;
- **backoff exponencial** no `429`, porque a bateria são 24+ chamadas;
- imprimir, no fim, um resumo com quantas mensagens foram processadas, quantas falharam e quais.

### 5. Um teste que não seja mentira

Escreva 3 asserções que verifiquem a saída do extrator **por propriedade**, não por igualdade de string (nota 02, §6). Explique em uma frase por que um `assert resposta == "..."` seria um teste inútil aqui — mesmo com a configuração corrigida.

### 6. A conta

Compare o custo médio por mensagem da versão 0 e da versão final, usando `usage` e os preços de <https://mistral.ai/pricing> (anote a data da consulta).

Uma das correções **aumenta** o custo. Diga qual, quanto, e por que ainda vale a pena — em termos de negócio, não de código.

## Saída esperada

```
=== LINHA DE BASE (versão 0) ===
[1/8] {"categoria": "entrega_atrasada", "pedido": "48219", "resumo": "Cliente relata que o pe
      -> INVÁLIDA: JSON truncado (finish_reason=length)
[2/8] {}
      -> INVÁLIDA: exceção engolida pelo except
...
válidas 14/24 | corretas 9/24 | estáveis 2/8

=== TABELA DE CORREÇÃO ===
(sua tabela do requisito 3)

=== VERSÃO FINAL ===
processadas: 8 | falhas: 0
custo médio por mensagem: US$ 0,0000xx (era US$ 0,0000yy)
```

## Desafios opcionais

- **Reproduza um problema de propósito.** Pegue a versão final e sabote **um só** parâmetro para provocar cada sintoma de novo, um por vez. Se você consegue causar o sintoma à vontade, você entendeu a causa.
- **A pergunta do `seed`.** Com a versão final (`temperature=0`), rode a mesma mensagem 10 vezes. Saiu idêntico sempre? Acrescente `extra_body={"random_seed": 42}` e rode 10 vezes de novo. Compare, e explique o que isso significa para um teste automatizado.
- **O que o schema não protege.** Encontre (ou construa) uma mensagem em que a saída passa em todas as suas validações e ainda assim está **errada**. Nota 02, §7.4.

## Entrega

- `08-extrator-confiavel.py`
- `diagnostico.md` com: os cinco diagnósticos (requisito 2), a tabela de correção (requisito 3), a justificativa da ordem que você escolheu, os testes por propriedade e a conta de custo
- Cópia do terminal da linha de base e da versão final

## Dicas

- **Não comece corrigindo.** Comece rodando e **olhando as saídas cruas**. Metade do exercício é ver o problema acontecer.
- O `except Exception: pass` do estagiário não foi maldade — foi medo de a fila da madrugada parar. Pense no que ele deveria ter feito para atender a mesma preocupação **sem** esconder a falha.
- Se uma correção sua não mudou nenhuma das três métricas, isso é um resultado válido e interessante: registre e explique. Talvez o parâmetro não fizesse o que você achava (nota 02, §2).
- Compare os seus dois piores casos com o gabarito **lendo**, não só contando. Erro de categoria e erro de extração de número têm causas diferentes.
- A mensagem com dois pedidos e a com número por extenso continuam difíceis mesmo com tudo configurado certo. Se elas sobrarem erradas no fim, isso não é falha do exercício — é o assunto do último desafio opcional.
