# Eventos — RabbitMQ, consumidor idempotente e dead-letter queue

Oficina do módulo 5. macOS, Docker Engine 29.6.2, Docker Compose v5.3.1, Python 3.12.14,
RabbitMQ 4 com management plugin. AMQP em `15672`, management em `15673`, como o roteiro
sugere. Tudo removido ao final com `down -v`.

Os sete arquivos vieram de `oficinas/modulo-5/` do repositório da disciplina, que a
própria página indica como byte a byte igual ao código publicado. Estão em `codigo/`
sem nenhuma alteração — conferi com `diff -r` contra o original.

## Configuração validada

`docker compose -f infra/compose.eventos.yml config --quiet` terminou com código `0` e
sem texto (`evidencias/compose-config.txt`). O `up -d --build --wait` retornou com o
contêiner saudável:

```
SERVICE    STATUS                   PORTS
rabbitmq   Up 8 seconds (healthy)   0.0.0.0:15672->5672/tcp, 0.0.0.0:15673->15672/tcp
```

O mapeamento confirma a distinção que o roteiro pede para observar: `15672` chega na
porta AMQP interna `5672`, usada pelo `aio-pika`; `15673` chega na `15672` interna, que
é o management e serve só para inspeção.

## Duas entregas, uma cobrança

A sequência publica o mesmo `event_id` sintético duas vezes
(`evidencias/sequencia-idempotencia.txt`):

```
consumidor --once    (declara a fila; vazia, nada é impresso)
publicador           Publicado: ... event_id=3fa85f64-5717-4562-b3fc-2c963f66afa6
consumidor --once    ... processed=True attempts=1
publicador           Publicado: ... event_id=3fa85f64-5717-4562-b3fc-2c963f66afa6
consumidor --once    ... processed=False attempts=2
```

O banco mostra o que interessa (`evidencias/consulta-sqlite.txt`):

```
processed_events:  ('3fa85f64-5717-4562-b3fc-2c963f66afa6', 2)
billing_effects — contagem: 1
billing_effects: ('3fa85f64-...', 'exam-sintetico-001', 'patient-sintetico-001',
                  'resultados/exam-sintetico-001', '2026-09-20 22:03:32')
```

Duas tentativas registradas, uma linha de efeito. O `processed=False` da segunda entrega
não é falha: ele afirma que aquela entrega não produziu efeito novo, que é precisamente
o comportamento desejado. Confundir isso com erro é o equívoco que os exercícios do
módulo descrevem — `attempts` conta entregas vistas, `billing_effects` conta
consequências de negócio, e só a segunda precisa ser única.

O teste automatizado sem broker deu `2 passed, 1 skipped`; o terceiro é opt-in e exige
`COMPOSE_LIVE=1`. Com o broker no ar e a variável ligada, `3 passed`
(`evidencias/testes-compose-live.txt`).

## A mensagem inválida fica visível

Publicando um payload sem `result_reference`, o consumidor não imprime linha de
processamento — imprime `Mensagem rejeitada para DLQ: schema inválido (1 erro)`. A
consulta ao management confirma (`evidencias/dlq-management.txt`):

```
billing.resultados.v1.dlq     mensagens: 1   prontas: 1   durável: True
billing.resultados.v1         mensagens: 0
```

A ordem das duas decisões do consumidor importa e aparece aqui: ele valida o schema
antes de consultar o registro de idempotência. Por isso o `event_id` da mensagem
inválida não entra em `processed_events` — gravar identificador de mensagem que nunca
deveria ter passado sujaria o registro que sustenta a idempotência.

A topologia final (`evidencias/topologia.txt`) mostra as quatro peças: a exchange
`hospital.events` do tipo topic, a fila de trabalho `billing.resultados.v1`, a exchange
de dead-letter `hospital.events.dlx` do tipo direct, e a fila
`billing.resultados.v1.dlq` segurando a mensagem recusada.

## Uma armadilha de observabilidade

Ao capturar a topologia, os dois endpoints do management API discordaram sobre a mesma
fila. O endpoint de listagem, `/api/queues/%2F`, devolveu `mensagens=0` para a DLQ;
o endpoint da fila específica, `/api/queues/%2F/billing.resultados.v1.dlq`, devolveu
`mensagens=1` de forma estável em três leituras seguidas. A mensagem estava lá: o
primeiro endpoint serve estatística agregada, com atraso de amostragem.

Registro isso porque quase virou conclusão errada. Eu tinha acabado de rodar o teste
live e supus que ele havia drenado a DLQ; republiquei a mensagem inválida para
"corrigir" a evidência, e o zero persistiu. Só a consulta por fila mostrou o estado
real. É exatamente o alerta do exercício de investigação do módulo — uma contagem não é
um diagnóstico, e vale saber qual instrumento produziu o número antes de agir sobre ele.
As capturas deste diretório usam o endpoint por fila.

## Por que entrega pelo menos uma vez com idempotência, e não exactly-once

O broker não consegue distinguir "o consumidor processou e o ack se perdeu" de "o
consumidor nunca processou". As duas situações são idênticas vistas de fora, porque o
RabbitMQ não tem acesso ao SQLite do consumidor — ele só sabe se recebeu ou não uma
confirmação. Diante da ambiguidade, reentregar é a escolha segura: troca duplicidade
visível, que dá para tratar, por perda silenciosa, que ninguém detecta.

Exactly-once exigiria que a gravação do efeito e o ack ao broker acontecessem na mesma
transação. São dois sistemas diferentes, com armazenamentos diferentes, e não existe
transação que os cubra. O que existe é o arranjo demonstrado aqui: o broker entrega pelo
menos uma vez, e o consumidor torna o efeito idempotente gravando `event_id` como chave
durável. O resultado observável se parece com exactly-once — uma cobrança para duas
entregas — mas o mecanismo é outro, e a diferença aparece no momento em que alguém
tenta remover o registro de idempotência.

Vale dizer por que o registro é o SQLite e não uma estrutura em memória. Um `set()` no
processo zera a cada reinício, e uma segunda réplica do consumidor começaria com o seu
próprio conjunto vazio, sem saber o que a primeira processou. Com `event_id` como chave
primária durável, a garantia passa a ser propriedade do dado, compartilhável entre
réplicas que apontem para o mesmo banco.

## Em que cenário Kafka valeria como extensão

Não pelo volume. O desenho atual atende bem um fato consumido por poucos destinos que
processam e seguem adiante. O gatilho é releitura: quando um consumidor precisar
reprocessar uma janela histórica por vontade própria, sem pedir republicação à origem.

No caso hospitalar isso aparece quando Auditoria ou Qualidade precisam recalcular os
últimos noventa dias porque uma regra de retenção ou a definição de um indicador mudou.
Numa fila, ler remove: reprocessar exige que Resultados publique tudo de novo, o que
acopla o consumidor à origem justamente no momento em que ele deveria ser autônomo.
Num log com retenção, ler não consome e cada consumidor guarda a própria posição, de
modo que rebobinar é decisão local.

O requisito mensurável seria esse: existe consumidor que precisa reler uma janela
definida sem intervenção da origem, e com que frequência. Sem isso, Kafka acrescenta
particionamento, offsets, retenção e evolução de esquema a uma equipe que hoje opera
filas — e o caso do LinkedIn é explícito ao dizer que a adoção move trabalho em vez de
eliminá-lo, tanto que exigiu construir Cruise Control, Brooklin e Bean Counter.

E Kafka não dispensaria nada do que foi demonstrado aqui. A entrega continuaria sendo
pelo menos uma vez, o `event_id` continuaria necessário para tornar o efeito idempotente,
e a versão do contrato continuaria sendo o que protege consumidores antigos. Mudaria
quem guarda a posição de leitura e por quanto tempo o fato fica disponível — não a
necessidade de tratar repetição.

## Limites

O Compose não é produção: um broker só, sem cluster, TLS, credenciais próprias, backup
ou política de retenção. As credenciais `guest:guest` valem apenas neste ambiente
descartável, e todos os identificadores são sintéticos. O `down -v` removeu contêiner,
rede e volume; `ps -a` terminou sem listar nada.
