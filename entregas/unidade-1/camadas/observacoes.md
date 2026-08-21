# Camadas — agenda clínica

Executado com `python3 main.py` dentro desta pasta. macOS, Python 3.12.14.

## A condição alterada

Em `repositorios.py`, no `listar_por_medico` de `InMemoriaRepositorioConsulta`,
apaguei a terceira condição da compreensão de lista:

```
             if c.medico.id == medico_id
             and c.horario.inicio.date() == data.date()
-            and c.status == "agendada"
```

Só isso. Os outros quatro arquivos seguem idênticos ao original.

## O que a saída revelou

O diff entre as capturas tem uma linha, no cenário 7 ("Agenda após alterações"):

```
- HTTP 200 OK → []
+ HTTP 200 OK → [{'id': 1, ..., 'status': 'realizada'}, {'id': 2, ..., 'status': 'cancelada'}]
```

Todo o resto saiu igual, inclusive o 409 do cenário 2 e o 400 do cenário 6. Mexi na
camada mais baixa e o efeito apareceu na resposta HTTP, sem tocar em apresentação nem
em serviço.

## A responsabilidade que isso expõe

`status == "agendada"` não é detalhe de armazenamento: é a regra que define o que
ocupa a agenda de um médico. Estava escrita no repositório — justamente a camada que
o cenário 9 do `main.py` apresenta como substituível "sem impacto", apoiado na ABC
`RepositorioConsulta`.

A saída mostra que a promessa é mais estreita do que parece. O ABC garante a
assinatura de `listar_por_medico`, não a semântica do que ela devolve. Uma
implementação Postgres que esqueça esse filtro no SQL não quebra nada: passa pelo
mesmo serviço e responde outra coisa, como na linha 41 acima.

O mesmo método alimenta dois consumidores — `RelatorioServico.agenda_diaria` e a
verificação de conflito em `AgendamentoServico.agendar`. A regra de conflito mora no
domínio (`Horario.conflita_com`), mas *quais* horários entram na comparação foi
decidido no repositório.

## Limite desta evidência

O efeito sobre o conflito não aparece na saída: o roteiro do `main.py` não tenta
agendar nada depois do cancelamento da consulta #2. Que a consulta cancelada voltaria
a bloquear o horário 10:00–10:30 eu deduzi lendo `agendar()`, não observei. Provar
isso exigiria acrescentar um POST ao cenário, o que já seria estendê-lo em vez de
alterar uma condição existente.

## As perguntas do roteiro

**Onde a entrada vira chamada ao serviço e onde a resposta HTTP é formatada?**
As duas coisas em `apresentacao.py`, dentro de `AgendaController`. Em
`post_consulta`, o dicionário cru vira um `SolicitacaoAgendamento` — é ali que
`datetime.fromisoformat` converte as strings — e só então `self._agendamento.agendar(...)`
é chamado. A resposta volta como um `Resposta(201, {...})`, e o texto
`HTTP 201 CREATED → ...` que aparece na saída é o `__str__` dessa dataclass.

O mapeamento de erro para status também mora só aí: `ConflitodeAgendaError` vira 409,
e `EntidadeNaoEncontradaError`, `KeyError` e `ValueError` viram 400. Nem o serviço nem
o domínio mencionam HTTP em lugar nenhum — eles levantam exceções de negócio, e quem
decide o código de status é o controller. Foi por isso que a linha 41 da saída mudou
sem que nenhum arquivo dessa camada fosse tocado no experimento.

**Qual regra impede o conflito e que objeto do domínio a expressa?**
`Horario.conflita_com`, em `dominio.py`:

```python
return self.inicio < outro.fim and self.fim > outro.inicio
```

`Horario` é um value object (`@dataclass(frozen=True)`) que também valida a si mesmo
no `__post_init__` — recusa fim anterior ao início. Quem aplica a regra é
`AgendamentoServico.agendar`, percorrendo as consultas devolvidas por
`listar_por_medico` e levantando `ConflitodeAgendaError` no primeiro conflito. A
comparação em si é do domínio; a orquestração é do serviço.

**Que dependência mudaria para trocar o armazenamento, e qual camada ficaria estável?**
Só a implementação concreta: trocar `InMemoriaRepositorioConsulta` por, digamos, um
`PostgresRepositorioConsulta` que herde da mesma ABC `RepositorioConsulta`. O ponto de
troca é a composição em `main.py`, onde os repositórios são instanciados e injetados
no construtor de `AgendamentoServico`. Serviço e domínio ficariam intactos, porque
importam a ABC e nunca a implementação.

É essa a resposta que o cenário 9 do `main.py` sugere, e é onde meu experimento
incomoda: a estabilidade vale para a assinatura, não para o comportamento. Uma
implementação Postgres que traduza `listar_por_medico` para um `SELECT` sem o
`WHERE status = 'agendada'` compila, satisfaz a ABC, passa pelo mesmo serviço — e
devolve a agenda errada, exatamente como na minha `saida-depois.txt`.
