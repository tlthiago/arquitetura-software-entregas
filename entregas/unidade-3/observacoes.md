# Serviços — dois bancos, uma fronteira e uma falha parcial

Oficina do módulo 3 executada em `laboratorios/plataforma-hospitalar`. macOS, Docker
Engine 29.6.2, Docker Compose v5.3.1, Python 3.12.14. Portas `18001` e `18002`, como o
roteiro sugere, para não disputar as padrão. Tudo foi removido ao final com `down -v`.

Desta vez o roteiro bateu com o repositório em todos os pontos verificáveis: os quatro
serviços no `config --services`, os quatro `healthy`, o `201`, o `503` e os `4 passed`.

## A questão exploratória

*Qual dependência permanece saudável e qual capacidade deixa de ser concluída?*

Com `elegibilidade` parado, o `ps -a` mostra:

```
db_elegibilidade   Up 25 seconds (healthy)
db_exames          Up 25 seconds (healthy)
elegibilidade      Exited (0) Less than a second ago
exames             Up 19 seconds (healthy)
```

Três dos quatro contêineres continuam saudáveis. O que deixa de ser concluído é uma
capacidade só: criar solicitação de exame, a única operação que atravessa a fronteira.

O detalhe que me pareceu mais interessante é `db_elegibilidade`. Ele continua `healthy`
o tempo todo — o banco não caiu, não perdeu dado, responde `pg_isready` normalmente.
Só que ninguém consegue mais chegar até ele, porque o único contêiner com rota até
aquela rede era o serviço que parou. Há dois níveis de "saudável e inútil" aqui: Exames,
que está vivo mas não conclui a operação, e o banco de Elegibilidade, que está perfeito
e inalcançável.

## As duas respostas contraditórias

Capturei o `POST` e o `GET /health` do mesmo serviço em sequência
(`evidencias/falha-parcial.txt`). Os dois carimbos de data são idênticos:

```
POST /exames  → HTTP/1.1 503 Service Unavailable   date: ... 22:50:34 GMT
                {"detail":{"codigo":"dependencia_indisponivel"}}

GET /health   → HTTP/1.1 200 OK                    date: ... 22:50:34 GMT
                {"status":"ok","servico":"exames"}
```

No mesmo segundo, o mesmo processo afirma que está bem e que não consegue trabalhar.
As duas respostas estão corretas, e é essa a definição prática de falha parcial: não há
um estado único "no ar / fora do ar" para reportar.

A razão está no health check declarado no Compose:

```yaml
test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
```

Ele chama `localhost` — o próprio contêiner. Exames se pergunta se está bem e responde
que sim, porque pela definição dele está mesmo: o processo responde e a base própria está
acessível. Ele não tem como saber que o vizinho sumiu, e o roteiro avisa isso antes de
acontecer.

Vale pensar no que mudaria se o health check consultasse as dependências. Exames viraria
`unhealthy` e o Docker poderia reiniciá-lo — sem efeito nenhum, porque o problema não é
dele, e ainda derrubando as operações que continuavam funcionando. A falha de um serviço
viraria a falha de dois. O health check auto-referente não é preguiça de quem escreveu:
é o que impede que a indisponibilidade se propague por reinício em cascata.

## A fronteira está no YAML, não no Python

As três redes são o mecanismo:

```yaml
networks:
  application-net:
  elegibilidade-db-net:
    internal: true
  exames-db-net:
    internal: true
```

`exames` está em `application-net` e `exames-db-net`. Não está em
`elegibilidade-db-net`. Não existe rota — uma consulta ao banco alheio falharia por
ausência de caminho, não por falta de senha ou por revisão de código.

O `ps` torna isso visível de outro jeito. Os dois bancos aparecem com `5432/tcp` puro,
enquanto as aplicações aparecem com `0.0.0.0:18001->8000/tcp`. A ausência do prefixo é
a evidência de que nem eu, do meu terminal, alcanço os bancos.

Há uma segunda camada de garantia que não é de rede:
`test_exames_source_cannot_access_eligibility_table_directly` lê o código-fonte de Exames
e falha se encontrar SQL contra a tabela de Elegibilidade. A infraestrutura impede que
funcione; o teste impede que alguém escreva. São dois mecanismos para a mesma regra,
atuando em momentos diferentes.

## A tradução de erros, exercitada

O roteiro apresenta uma tabela de tradução, então resolvi exercitá-la em vez de só ler.
Subi os serviços de novo e observei os dois lados de cada caso
(`evidencias/traducao-de-erros.txt`):

```
paciente-001   Elegibilidade → 200 {"elegivel":true}                         Exames → 201
paciente-002   Elegibilidade → 200 {"elegivel":false}                        Exames → 422 beneficiario_inelegivel
paciente-999   Elegibilidade → 404 {"codigo":"beneficiario_nao_encontrado"}  Exames → 422 beneficiario_desconhecido
(parado)       Elegibilidade → sem resposta                                  Exames → 503 dependencia_indisponivel
```

Quatro das cinco linhas da tabela com evidência. Duas me chamaram atenção.

Em `paciente-999`, Elegibilidade devolve `404` e Exames devolve `422`. O status não é
repassado, e o código muda de `beneficiario_nao_encontrado` para
`beneficiario_desconhecido`. Faz sentido: para quem chamou `POST /exames`, o recurso
`/exames` existe — um `404` ali significaria que a própria rota sumiu. Repassar o status
do vizinho contaria ao consumidor que existe um vizinho, e amarraria o contrato externo à
topologia interna.

Em `paciente-002`, Elegibilidade responde `200`. Tecnicamente nada falhou: a pergunta foi
feita e respondida com sucesso. A resposta é que foi negativa, e isso vira `422`, não
`503`. Decisão de negócio e falha de infraestrutura ocupam faixas diferentes do contrato,
e só quem lê o corpo distingue `beneficiario_inelegivel` de `beneficiario_desconhecido`
— os dois são `422`.

## O que os testes provam, e o que não provam

`4 passed`, como o roteiro prevê. Mas a seção se chama "verificar as fronteiras sem
depender do Compose", então testei a afirmação: rodei de novo **depois** do `down -v`,
com nenhum contêiner, rede ou volume existindo (`evidencias/testes-sem-compose.txt`).
Passaram igual, em 0,13s.

Isso é bom e é limitado ao mesmo tempo. Bom porque a fronteira pode ser verificada em
qualquer máquina, sem Docker, num pipeline de CI. Limitado porque
`test_exames_makes_partial_failure_observable_when_dependency_is_down` passa sem que
exista falha parcial alguma — ele simula a indisponibilidade e confere que o código
traduz para `503`. O teste prova que o tratamento está escrito, não que o sistema se
comporta assim quando um contêiner cai de verdade.

Quem produziu essa evidência foi o Docker, nos dois carimbos de `22:50:34`. É a mesma
distinção do módulo anterior entre contrato, teste e execução, aparecendo agora entre
teste unitário e demonstração distribuída: cada instrumento vê um pedaço, e nenhum
substitui o outro.

## Limites desta demonstração

Não demonstrei repetição de chamada, disjuntor, fila, consistência eventual nem
recuperação parcial. O `timeout=2.0` do cliente httpx evita que Exames fique pendurado,
mas não há nova tentativa: a primeira falha vira `503` na hora.

Os dados também não sobrevivem. O `solicitacao_id` saiu `1` na primeira solicitação e
chegou a `4` ao longo dos testes da tabela; depois do `down -v` a contagem reinicia,
porque o volume vai junto. O `201` afirma que o recurso passou a existir, e ele existe
mesmo — só que dentro de um volume que a limpeza apaga.
