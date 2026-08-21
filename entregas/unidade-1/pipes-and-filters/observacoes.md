# Pipes and Filters — triagem de currículos

Executado com `python3 main.py` dentro desta pasta. macOS, Python 3.12.14.

## A condição alterada

Comentei uma linha na composição do pipeline, em `main.py`:

```
         .adicionar(ValidadorDeCurriculo())
-        .adicionar(NormalizadorDeCampos())
         .adicionar(FiltroPorExperienciaMinima(vaga))
```

Nenhum filtro foi editado, e os dados de entrada seguem os originais. Queria testar a
terceira pergunta do roteiro — que efeito teria reorganizar filtros — no caso mais
simples: tirar um do meio.

## O que a saída revelou

O pipeline não quebrou. Manteve os mesmos três descartes (`id=3` sem nome, Bruno por
experiência, Clara por pretensão) e continuou aprovando três candidatos. O relatório
final, porém:

```
- 1. Ana Lima        Score: ██████████ 100%   python, docker, postgresql, rest
+ 1.   ana lima      Score: ░░░░░░░░░░   0%   —
- 2. Elena Souza     Score: ██████████ 100%   python, docker, postgresql, rest
+ 2. Diego Faria     Score: ░░░░░░░░░░   0%   —
- 3. Diego Faria     Score: ███████░░░  75%   python, postgresql, rest
+ 3. Elena Souza     Score: ░░░░░░░░░░   0%   —
```

Todos os scores foram a zero. `CalculadorDeScore` guarda
`self._requeridas = {h.lower() for h in vaga.habilidades_requeridas}` e faz
`set(c.habilidades) & self._requeridas`; os currículos brutos trazem `"Python"`,
`"PostgreSQL"`, `"Docker"`, `"REST"` com maiúsculas. Quem alinhava os dois lados era o
`NormalizadorDeCampos`, com `h.lower().strip()`. Sem ele a interseção é vazia, o score
é `0/4` e a coluna de compatíveis vira `—`. O nome também voltou ao bruto,
`"  ana lima  "`, porque o `.strip().title()` estava no mesmo filtro.

Não tinha previsto a troca entre 2º e 3º lugar. Com todos empatados em `0.0`, o
`sorted(..., key=score, reverse=True)` do consumer não tem o que ordenar, e por ser
estável devolve a lista na ordem de chegada (ids 1, 5, 6). O ranking não ficou errado
— deixou de existir, e no lugar dele apareceu a ordem de leitura.

## A responsabilidade que isso expõe

`Pipeline.executar` faz só isto:

```python
for filtro in self._filtros:
    resultado = filtro.processar(resultado)
```

O contrato de `Filtro` é `processar(dados) -> dados`, sem tipo nem pré-condição. No
nível da assinatura os filtros são mesmo independentes, e foi por isso que o programa
rodou até o fim sem erro.

Só que existe um acoplamento real entre `NormalizadorDeCampos` e `CalculadorDeScore`:
o segundo assume habilidades já em minúsculas. Essa dependência não está declarada em
lugar nenhum — existe apenas como a ordem das chamadas `.adicionar(...)` em `main.py`.
É uma pré-condição de dados sustentada por convenção, não por código.

O que mais me chamou atenção foi a falha ser silenciosa. Sem exceção, sem aviso, sem
contagem diferente: a última linha continuou dizendo "3 candidato(s) encaminhado(s)
para entrevista", agora com três pessoas de 0% de aderência. Separar bem as
responsabilidades de processamento não separa, sozinho, as de formato.

## Uma nota sobre reprodutibilidade

A ordem dentro da linha "Habilidades compatíveis" muda a cada execução — numa rodada
sai `postgresql, python, docker, rest`, na seguinte `docker, python, rest, postgresql`.
A causa é `list(set(c.habilidades) & self._requeridas)`: a iteração de um `set` de
strings depende do hash randomization, que o Python sorteia por processo. Testei em
3.9.6 e em 3.12.14 e varia nas duas.

Isso não altera score, aprovados nem ranking, mas significa que as capturas deste
diretório não são byte a byte reproduzíveis nessa linha. Um `sorted()` no consumer
resolveria — o pipeline não define ordem para esse campo em lugar nenhum.

## As perguntas do roteiro

**Qual parte recebe os dados brutos e qual apresenta o resultado final?**
O producer `LeitorDeCurriculos` recebe. Ele é o único filtro que ignora o que vem
pelo pipe — a assinatura é `processar(self, _)` — porque a fonte dele é a lista
`CURRICULOS_BRUTOS`, passada no construtor. É ali que os dicionários viram objetos
`Curriculo`. Por isso a chamada final é `pipeline.executar(None)`: não há nada para
alimentar o primeiro estágio.

Quem apresenta é o consumer `RelatorioDeTriagem`. Ele é o único filtro que imprime o
bloco final, ordena por score e devolve a lista de aprovados, que o `main.py` usa só
para contar. Todo o resto entre os dois trabalha em silêncio, exceto as linhas de
descarte.

**Em que etapas os itens deixam de seguir, e em qual são transformados sem descarte?**
Os três testers descartam, e cada um imprime a própria linha:
`ValidadorDeCurriculo` tirou o currículo `id=3` por nome ausente,
`FiltroPorExperienciaMinima` reprovou Bruno (1 ano contra o mínimo de 3) e
`FiltroPorPretensaoSalarial` reprovou Clara (R$22.000 contra o teto de R$18.000). Dos
seis currículos de entrada sobraram três.

Os dois transformers não descartam ninguém. `NormalizadorDeCampos` altera os objetos
no lugar e devolve a mesma lista. `CalculadorDeScore` devolve uma lista do mesmo
tamanho, mas de outro tipo: cada `Curriculo` vira um `ResultadoTriagem` que embrulha o
currículo e acrescenta score e habilidades compatíveis. Vale reparar que ele muda o
tipo do que trafega no pipe sem que o framework saiba — e é por isso que o normalizador
não poderia simplesmente ser movido para depois dele: receberia `ResultadoTriagem` e
quebraria no primeiro `c.candidato_nome`. Tirá-lo do fluxo foi a única forma de mexer
na ordem sem provocar erro.

**Por que o ranking pertence ao fim, e que efeito teria reorganizar filtros?**
Porque a chave da ordenação é o `score`, que não existe antes do `CalculadorDeScore`.
O consumer faz `sorted(..., key=lambda r: r.score, reverse=True)` e não tem como
verificar se aquele número significa alguma coisa — ele ordena o que recebeu.

O efeito de reorganizar foi justamente o que este experimento mediu, e ele não é
uniforme. Trocar dois testers de posição mudaria só a ordem das linhas de descarte,
porque cada um lê um campo diferente e nenhum depende do outro. Já tirar um transformer
do meio manteve a mesma quantidade de aprovados e destruiu o critério do ranking.
Reordenar é seguro para filtros que só leem dados de entrada, e não é para os que
dependem do que outro filtro escreveu antes.
