# Arquitetura de Software — entregas

Disciplina de Arquitetura de Software (IEC / PUC Minas).
Material: https://marco-mendes.github.io/arquitetura-software/

## Unidade 1 — Oficina de ferramentas

[Roteiro da oficina](https://marco-mendes.github.io/arquitetura-software/modulo-1-visao-geral/oficina-de-ferramentas/)

Em `entregas/unidade-1/` estão as três cópias pedidas, cada uma com a execução antes
e depois da alteração e a nota de observação:

| Pasta | Estilo | Alteração |
| --- | --- | --- |
| [`camadas/`](entregas/unidade-1/camadas) | Camadas | removi `and c.status == "agendada"` de `listar_por_medico`, em `repositorios.py` |
| [`pipes-and-filters/`](entregas/unidade-1/pipes-and-filters) | Pipes and Filters | tirei `NormalizadorDeCampos` da composição do pipeline, em `main.py` |
| [`microkernel/`](entregas/unidade-1/microkernel) | Microkernel | reordenei `ORDEM_CATEGORIAS` para `["notificacao", "impostos", "frete"]`, em `nucleo.py` |

**As respostas estão nos arquivos `observacoes.md`** — um em cada pasta. Cada um traz a
condição que alterei, o que mudou entre `saida-antes.txt` e `saida-depois.txt`, a
responsabilidade arquitetural que a evidência expõe e as respostas às questões
exploratórias do roteiro.

Os `README.md` dentro das três pastas são os originais do repositório da disciplina,
copiados junto com o código. Não os alterei.

Em cada experimento mexi em uma linha de um arquivo só. Todo o resto de cada cópia
está idêntico ao original.

## Reproduzir

Apenas biblioteca padrão, sem dependências. A partir da raiz:

```bash
cd entregas/unidade-1/camadas && python3 main.py
```

O mesmo para `pipes-and-filters` e `microkernel`. Para voltar ao comportamento
original, basta desfazer a linha indicada na tabela.

Rodei em macOS com Python 3.12.14, e as capturas foram geradas nessa versão.

Uma ressalva sobre reprodutibilidade: em `pipes-and-filters`, a ordem dentro da linha
"Habilidades compatíveis" muda a cada execução, porque o filtro de score monta essa
lista a partir de um `set`. Isso não afeta score, aprovados nem ranking. Está
explicado na nota daquela pasta.

## Unidade 2 — Oficina de ferramentas: contrato, execução e comparação

[Roteiro da oficina](https://marco-mendes.github.io/arquitetura-software/modulo-2-apis/oficina-de-ferramentas/)

Trilha essencial executada em `laboratorios/plataforma-hospitalar`: API FastAPI de
elegibilidades, contrato OpenAPI 3.1, Bruno como consumidor, Spectral como linter e
`TestClient` como regressão. Não fiz a extensão opcional do Ocelot.

As evidências estão em [`entregas/unidade-2/evidencias/`](entregas/unidade-2/evidencias):

| Arquivo | O que mostra |
| --- | --- |
| `testes-contrato.txt` | `7 passed` em `test_api_contract.py` |
| `testes-todos.txt` | `20 passed, 3 skipped` na suíte completa |
| `spectral-valido.txt` | contrato original sem erros, código de saída `0` |
| `spectral-experimento.txt` | falha deliberada `oas3-valid-media-example`, código `1` |
| `openapi-experimento.yaml` | a cópia com `cpf: '123'` que provoca a falha |
| `openapi-gerado.json` | contrato que o FastAPI gera, para comparar com o explícito |
| `http/` | respostas cruas de `202`, `200`, `422` (duas formas) e `404` |
| `bruno/` | coleção gerada a partir do `openapi.yaml` |
| `bruno-execucao.txt` | a coleção rodando contra o servidor: `202` e `200` |

**A nota está em [`observacoes.md`](entregas/unidade-2/observacoes.md)** — a comparação
entre contrato explícito, contrato gerado e execução, a leitura da falha deliberada e as
respostas às cinco questões exploratórias.

Duas observações que valem antes de repetir a oficina: `pip install -e ".[dev]"` não
instala o `PyYAML` que `test_api_contract.py` importa, e os testes de contrato são sete,
não os seis que o roteiro indica. Ambas estão detalhadas na nota.
