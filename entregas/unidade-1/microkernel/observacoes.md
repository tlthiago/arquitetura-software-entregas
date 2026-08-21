# Microkernel — faturamento por plugins

Executado com `python3 main.py` dentro desta pasta. macOS, Python 3.12.14.

## A condição alterada

Uma linha em `nucleo.py`, dentro de `CoreFaturamento`:

```
- ORDEM_CATEGORIAS = ["impostos", "frete", "notificacao"]
+ ORDEM_CATEGORIAS = ["notificacao", "impostos", "frete"]
```

Nenhum plugin foi tocado, nenhum registro em `main.py` mudou, nenhuma fatura foi
alterada. Escolhi essa condição porque é a única decisão de comportamento que sobra
dentro do núcleo — todo o resto ele delega.

## O que a saída revelou

O diff tem quatro linhas, e todas são a mensagem de e-mail de uma fatura:

```
#1001  e-mail: R$13.400,00 → R$12.000,00     (o bloco TOTAL continuou R$13.400,00)
#1002  e-mail: R$10.080,00 → R$ 8.400,00     (TOTAL continuou R$10.080,00)
#1003  e-mail: R$84.000,00 → R$75.000,00     (TOTAL continuou R$84.000,00)
#1004  e-mail: R$53.100,00 → R$45.000,00     (TOTAL continuou R$53.100,00)
```

Em todos os casos o valor anunciado passou a ser o bruto. `NotificacaoEmailPlugin` lê
`resultado.valor_total`, propriedade calculada de `ResultadoEmissao`
(`valor_bruto + total_impostos + frete`); executando primeiro, encontra
`impostos = {}` e `frete = 0.0`. Na #1001 isso é um e-mail dizendo R$12.000,00 ao
cliente enquanto a fatura fecha em R$13.400,00 — R$1.400,00 de diferença, sem erro na
tela. O registro de plugins saiu idêntico, o que faz sentido: registrar e executar são
momentos separados.

## O que não mudou, e por que importa

O frete também trocou de posição — era o segundo, virou o último — e não mudou de
valor em nenhuma fatura. `FreteCorrespondenciaPlugin` decide a isenção com
`resultado.valor_bruto`, que o núcleo preenche na criação do `ResultadoEmissao` e
ninguém mais altera. Os plugins de imposto, do mesmo jeito, leem `fatura.itens` ou
`valor_bruto`. Nenhum depende do que outro escreveu antes.

Dos cinco plugins registrados, só um é sensível à ordem: o de notificação, o único que
lê estado acumulado em vez de dados de entrada. Isso me fez reler o comentário logo
acima da linha que alterei:

```python
# Ordem de execução garante consistência (impostos antes de frete, etc.)
```

A evidência não sustenta essa justificativa. Impostos antes de frete é indiferente
aqui, porque o frete não olha para os impostos. A dependência real é outra —
notificação depois de tudo — e o comentário aponta para a relação errada.

## A responsabilidade que isso expõe

O contrato é minúsculo: `PluginFaturamento` exige `nome` e
`processar(fatura, resultado) -> resultado`, e `emitir()` é um laço duplo sobre
categorias e plugins. Essa estreiteza é o que dá a extensibilidade demonstrada no fim
do `main.py`, onde `ImpostoMGPlugin` entra sem que uma linha do núcleo mude.

O preço aparece aqui. Como o contrato não diz nada sobre o que um plugin lê ou escreve
no `ResultadoEmissao`, o núcleo não tem como saber que a notificação precisa vir por
último. Nenhum plugin declara essa dependência e o registry não teria como validá-la.
A consistência do resultado depende inteiramente de uma lista literal dentro do
núcleo, cuja correção nada no sistema verifica.

E de novo a falha é silenciosa: os quatro e-mails foram "enviados", os quatro totais
impressos, nenhuma exceção. A divergência entre valor notificado e valor faturado só
aparece comparando as duas capturas.

## As perguntas do roteiro

**Que contrato o núcleo conhece e o que ele deixa para os plugins?**
`PluginFaturamento`, um `Protocol` com duas coisas: a propriedade `nome` e o método
`processar(fatura, resultado) -> resultado`. Nada além disso. O registry usa
`isinstance` contra esse protocolo (é `@runtime_checkable`) e recusa qualquer objeto
que não o satisfaça.

Fica de fora do núcleo tudo que é regra: as alíquotas, a condição que decide se a
regra se aplica (`if fatura.cliente.estado != "SP": return resultado`), a tabela de
frete, o limite de isenção e o canal de notificação. O núcleo também não sabe se um
plugin vai escrever no resultado ou apenas lê-lo — e é exatamente essa ignorância que
o experimento explora.

**Como a ordem por categoria afeta o total e a notificação?**
O total impresso não mudou em nenhuma das quatro faturas; a notificação mudou nas
quatro. A razão é que `valor_total` é uma propriedade calculada, avaliada no momento
em que alguém a lê. O bloco de impressão do `main.py` lê depois que `emitir()`
terminou, então sempre vê o valor final. O plugin de notificação lê durante a
execução, e passou a ver um resultado ainda vazio. A ordem não altera o que é
calculado — altera o instante em que cada plugin observa o cálculo.

**Quais regras contribuem para a fatura de SP, a de RJ e a de valor alto?**

Na #1001 (SP, R$12.000 brutos) contribuem dois plugins de imposto. O `ICMS-SP` aplica
alíquota por categoria: 12% sobre os R$10.000 de eletrônico e 5% sobre os R$2.000 de
serviço, somando os R$1.300 da saída. O `ISS-SP` incide só sobre serviço, 5% de
R$2.000, e por isso aparece como R$100 numa linha separada. O `ICMS-RJ` estava
registrado e rodou, mas devolveu o resultado intocado no `if` de estado, e por isso
não deixa linha nenhuma no relatório.

Na #1002 (RJ, R$8.400 brutos) acontece o inverso: os dois plugins paulistas saem pela
guarda de estado e só o `ICMS-RJ` contribui, com alíquota única de 20% sobre o bruto —
R$1.680. Repare que ele usa `resultado.valor_bruto`, enquanto o `ICMS-SP` percorre
`fatura.itens`. Dois plugins da mesma categoria lendo fontes diferentes, e o núcleo
não distingue um do outro.

Na #1003 (SP, R$75.000 brutos) contribui só o `ICMS-SP`, com R$9.000 — 12% sobre um
item inteiramente eletrônico. O `ISS-SP` não deixa linha porque a fatura não tem item
de serviço, e o próprio plugin só grava a chave `if iss > 0`.

Sobre o frete, a saída mostra R$0,00 nas quatro faturas. Todas passam do limite de
isenção de R$5.000, então a `TABELA` por estado do `FreteCorrespondenciaPlugin` nunca
chega a ser consultada neste cenário. A regra de frete está registrada e é executada
toda vez, mas sua única contribuição observável no relatório é o zero da isenção.
