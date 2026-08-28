# APIs — contrato, execução e comparação

Trilha essencial da oficina do módulo 2, executada em `laboratorios/plataforma-hospitalar`.
macOS, Python 3.12.14, Node v24.18.0, Spectral CLI 6.16.1, Bruno CLI 4.0.0. Não fiz a
extensão opcional do Ocelot.

## Duas coisas que não bateram com o roteiro

Registro antes do resto porque mudam os comandos de quem for repetir.

**O `pip install -e ".[dev]"` não deixa a suíte executável.** `tests/test_api_contract.py`
faz `import yaml` na linha 5, mas `PyYAML` não está declarado nem em `dependencies` nem
em `[project.optional-dependencies].dev` do `pyproject.toml`. Seguindo o roteiro à
risca, o pytest nem chega a coletar:

```
tests/test_api_contract.py:5: in <module>
    import yaml
E   ModuleNotFoundError: No module named 'yaml'
!!!!!!!!!!!!!!!!!!! Interrupted: 2 errors during collection !!!!!!!!!!!!!!!!!!!!
```

Resolvi com `pip install pyyaml` (6.0.3) na mesma `.venv`. Vale notar o que isso é: o
`pyproject.toml` é um contrato de dependências, e ele declara menos do que o código
usa. É a mesma classe de problema que a oficina inteira investiga em OpenAPI, só que
num contrato diferente.

**São sete testes de contrato, não seis.** O roteiro diz "procure especificamente os
seis testes de `test_api_contract.py`" e "o resumo mostra `6 passed`". O resumo mostrou
`7 passed`, e a suíte completa, `20 passed, 3 skipped`. Os sete estão em
`evidencias/testes-contrato.txt`; o que provavelmente entrou depois é
`test_health_endpoints_distinguish_process_liveness_from_traffic_readiness`, e ele
importa para a comparação mais adiante.

## Contrato explícito, contrato gerado e execução

Esta é a comparação que a oficina pede. Coloquei os três lado a lado extraindo
operações e status de `contratos/openapi.yaml` e de `app.openapi()`:

```
versão OpenAPI  explícito=3.1.0  gerado=3.1.0
operações       explícito=2      gerado=2

  GET /elegibilidades/{protocolo}   explícito=['200','404']  gerado=['200','404','422']
  POST /elegibilidades              explícito=['202','422']  gerado=['202','422']

schemas só no explícito: nenhum
schemas só no gerado   : ['HTTPValidationError', 'ValidationError']
```

Os dois concordam nas operações e divergem no resto, de dois jeitos opostos.

O `422` do `GET` existe só no gerado. O FastAPI o acrescenta porque a operação tem um
parâmetro de caminho e ele assume que validação pode falhar. Só que `protocolo` é `str`
sem restrição, então nenhuma entrada chega a ser rejeitada: testei com `%20` e com uma
barra a mais, e as duas devolveram `404`. É uma promessa que o contrato gerado faz e a
implementação nunca cumpre.

Os schemas `HTTPValidationError` e `ValidationError` também são invenção do framework.
O contrato explícito modela erro com `ErroAPI`, e é `ErroAPI` que o servidor devolve de
fato — confira `evidencias/http/post-422.txt`. O gerado carrega dois schemas que nenhuma
resposta real usa.

Na direção contrária, `/health/live` e `/health/ready` existem, respondem e são
cobertos por teste, mas estão registrados com `include_in_schema=False`. Não aparecem em
nenhum dos dois documentos. Quem só lê o OpenAPI não sabe que existem; quem opera o
serviço depende deles.

## A divergência que os testes não pegam

`test_application_and_explicit_contract_agree_on_operations_and_models` é o teste que
mais se aproxima disso, e ele passa. Lendo o que ele compara: a presença dos dois
métodos, o conjunto `required` de `PedidoElegibilidade`, e a descrição e o header
`Location` do `202`. Não compara o conjunto de status por operação, nem a lista de
schemas, nem rotas fora do schema.

Ou seja, as três divergências acima passam pelos sete testes sem disparar nada. Não é
descuido: cada asserção escolhe uma amostra, e amostra não é cobertura. O teste prova
que a implementação atende ao que ele resolveu olhar.

## A falha deliberada do Spectral

`evidencias/openapi-experimento.yaml` é uma cópia do contrato com `cpf` alterado de
`'12345678901'` para `'123'` no exemplo de mídia da requisição. O lint reprova:

```
 35:24  error  oas3-valid-media-example  "cpf" property must match pattern "^\d{11}$"
        paths./elegibilidades.post.requestBody.content.application/json.examples.pedidoValido.value.cpf
✖ 1 problem (1 error, 0 warnings, 0 infos, 0 hints)
```

Código de saída `1`, contra `0` do contrato original em `evidencias/spectral-valido.txt`.
Isso é o resultado pretendido do experimento, não um defeito a corrigir: o arquivo existe
para mostrar que o exemplo é verificável contra o schema, sem que ninguém suba servidor.

O que achei mais interessante foi a simetria. O mesmo valor `'123'` enviado ao servidor
devolve `422` com `"String should match pattern '^\\d{11}$'"`
(`evidencias/http/post-422-cpf-curto.txt`). O mesmo `^\d{11}$` reprova em dois momentos
diferentes — no documento, antes de existir requisição, e na execução, com requisição
real. O linter erra cedo e barato; o servidor erra tarde e caro. Nenhum dos dois torna o
outro dispensável, porque o linter não sabe se o servidor obedece ao documento.

## O consumidor

A coleção em `evidencias/bruno/` foi gerada a partir de `contratos/openapi.yaml` pelo
conversor `openApiToBruno` — o mesmo que o Bruno usa no **Import Collection → OpenAPI**.
Saíram as duas requisições esperadas, com `{{baseUrl}}` como variável e o environment
`Local` apontando para `http://127.0.0.1:8000`.

Executei a coleção pelo Bruno CLI contra o servidor no ar
(`evidencias/bruno-execucao.txt`):

```
Elegibilidades/Aceita uma consulta de elegibilidade (202 Accepted) - 6 ms
Elegibilidades/Consulta uma elegibilidade aceita (200 OK) - 1 ms
Status ✓ PASS   Requests 2 (2 Passed)
```

Repare na linha `Tests 0/0, Assertions 0/0`. A coleção importada dispara as duas
chamadas e confere só que houve resposta — ela não afirma nada sobre o corpo. Como
regressão isso vale pouco; como leitura do contrato pela perspectiva de quem consome,
vale bastante, porque o que veio do OpenAPI foi suficiente para montar as duas chamadas
sem ler uma linha do código do servidor.

## As respostas HTTP

Em `evidencias/http/`. O `POST` devolveu `202` com
`location: /elegibilidades/c7a6d857-c672-4f8e-8640-bfdf5f603127` e corpo com `protocolo`
e `situacao: recebida`. Segui esse `Location` para o `GET` em vez de montar a URL pela
convenção, e recebi `200` com o mesmo protocolo e o mesmo `criado_em`. O `POST` sem
`cpf` devolveu `422` com `codigo: dados_invalidos` e `campo: body.cpf`, e um protocolo
inexistente devolveu `404` com `codigo: elegibilidade_nao_encontrada`.

Vale dizer o que essas capturas não provam. Todos os protocolos vivem na memória do
processo: reiniciar o Uvicorn apaga tudo, e nenhuma delas demonstra persistência,
autenticação, idempotência ou integração externa.

## As questões exploratórias

**1. O que o `202` permite ao provedor mudar sem quebrar o consumidor?**
O `202` afirma que o pedido foi aceito, não que foi decidido. Isso libera todo o "como"
e o "quando": a plataforma pode responder na hora, consultar a operadora depois,
enfileirar, tentar de novo ou trocar o mecanismo interno, sem alterar o contrato. Um
`200` com veredito prenderia o provedor ao tempo da operadora. O preço é que o consumidor
precisa acompanhar o protocolo — o `202` transfere para ele o trabalho de descobrir o
desfecho.

**2. Por que `Location` é melhor que pedir ao consumidor para montar a URL?**
Porque troca convenção por dado. Com `Location`, quem sabe onde o recurso vive é quem o
criou, e o consumidor só segue o que recebeu — foi o que fiz na captura do `GET`. Se
amanhã o caminho virar `/v2/elegibilidades/{id}` ou apontar para outro host, o cliente
que segue o header continua funcionando e o que concatena strings quebra. A URL montada
por convenção transforma um detalhe do provedor em acoplamento do consumidor.

**3. Qual divergência entre OpenAPI e aplicação os testes atuais ainda não detectam?**
Três, documentadas acima: o `422` do `GET` que o contrato gerado promete e a
implementação nunca produz; os schemas `HTTPValidationError` e `ValidationError`, que só
existem no gerado; e `/health/live` e `/health/ready`, que existem e funcionam mas estão
fora dos dois documentos. As três atravessam os sete testes sem falhar, porque a
comparação existente olha operações, `required` e o `202` — não o conjunto de status,
nem a lista de schemas, nem rotas com `include_in_schema=False`.

**4. Quando uma chave de idempotência passaria a ser necessária?**
Quando repetir o mesmo `POST` deixar de ser inofensivo. Hoje cada chamada cria um
protocolo novo e nada acontece fora do processo, então duplicar é só ruído. Vira problema
quando o aceite dispara efeito externo — abrir uma autorização na operadora, agendar
coleta, cobrar. Aí o cliente que sofre timeout e repete não sabe se o primeiro pedido
chegou, e sem chave o provedor não sabe se está diante de um pedido novo ou do mesmo de
novo. A necessidade não nasce do volume, nasce do momento em que o efeito colateral passa
a ser observável fora da API.

**5. Que parte do experimento deixaria de funcionar com duas instâncias e memória
separada?**
A recuperação pelo `GET`. O dicionário de protocolos vive no processo, então o `POST`
atendido pela instância A cria um protocolo que a instância B desconhece — e o `GET`
balanceado para B devolveria `404` para um recurso que existe. É o cenário mais
traiçoeiro do conjunto, porque o `404` é uma resposta legítima do contrato: nem o
consumidor nem o monitoramento conseguem distinguir "protocolo inexistente" de "protocolo
na outra instância". O `202` continuaria correto e o `422` também, porque nenhum dos dois
depende de estado compartilhado. Só o que foi aceito precisa ser encontrado depois.
