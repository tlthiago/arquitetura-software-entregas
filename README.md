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
