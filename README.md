# Teste de Performance com JMeter – YouTube

Exercício do curso **Engenheiro de Qualidade de Software (EBAC)**: criação de um script de teste de performance com Apache JMeter, usando massa de dados em arquivo `.csv`.

## Objetivo

Simular 20 usuários virtuais pesquisando termos diferentes no YouTube durante 3 minutos e analisar o comportamento (tempo de resposta, taxa de erro e vazão).

## Requisitos do exercício

| Item | Valor |
|---|---|
| Ferramenta | Apache JMeter 5.6.3 |
| Endpoint | `www.youtube.com` |
| Massa de dados | Arquivo `.csv` com 10 palavras |
| Número de usuários | 20 |
| Ramp-up | 60 segundos |
| Duração | 3 minutos (180 segundos) |

## Arquivos do repositório

| Arquivo | Descrição |
|---|---|
| `Usuarios Youtube.jmx` | Plano de teste do JMeter |
| `dados.csv` | Massa de dados: 10 palavras, uma por linha, sem cabeçalho |

## Estrutura do plano de teste

```
Test Plan
└── Usuarios Youtube (Thread Group)
    ├── CSV Data Set Config
    ├── HTTP Cookie Manager
    ├── HTTP Header Manager
    ├── Pesquisa Youtube (HTTP Request)
    ├── View Results Tree   (desativado na execução final)
    ├── Summary Report
    └── Aggregate Report
```

### Configurações principais

**Thread Group**
- Number of Threads: `20`
- Ramp-up period: `60` s (1 novo usuário a cada 3 s)
- Loop Count: `Infinite`
- Specify Thread lifetime: marcado, Duration `180` s

**CSV Data Set Config**
- Filename: `dados.csv` (caminho relativo)
- Encoding: `UTF-8`
- Variable Names: `palavra`
- Recycle on EOF: `True` | Stop thread on EOF: `False`
- Sharing mode: `All threads`

**HTTP Request (Pesquisa Youtube)**
- Protocolo: `https` | Servidor: `www.youtube.com`
- Método: `GET` | Path: `/results`
- Parâmetro: `search_query = ${palavra}`

**HTTP Cookie Manager / HTTP Header Manager**
- Cookie Manager com configuração padrão
- Header `User-Agent` simulando um navegador Chrome

## Como executar

1. Instale o [Apache JMeter](https://jmeter.apache.org/) (e o Java necessário).
2. Clone este repositório e mantenha o `.jmx` e o `dados.csv` **na mesma pasta**.
3. Abra o `Usuarios Youtube.jmx` no JMeter.
4. Limpe os listeners (botão da vassoura) e clique em **Start** (▶).
5. Aguarde os 3 minutos completos e consulte o **Summary Report** e o **Aggregate Report**.

## Resultado da execução final

Execução completa de 3 minutos, 20 usuários, 0% de erro.

| Métrica | Valor |
|---|---|
| Samples | 2889 |
| Average | 1050 ms |
| Median | 859 ms |
| 90% Line | 1511 ms |
| 95% Line | 1995 ms |
| 99% Line | 5371 ms |
| Min / Max | 433 ms / 16837 ms |
| Error % | 0,00% |
| Throughput | 16,0 req/s |
| Received KB/sec | ~23.911 |
| Avg. Bytes | ~1,5 MB por resposta |

### Análise

- A maioria das requisições foi rápida e estável: mediana de ~0,86 s e 95% das respostas em até ~2 s.
- A média (1050 ms) ficou acima da mediana (859 ms) porque poucas respostas lentas, com máximo de ~16,8 s, elevam o valor. Por isso os percentis descrevem melhor a experiência típica do que a média.
- Não houve nenhum erro durante os 3 minutos.
- Cada resposta tem ~1,5 MB, e o teste baixou ~23 MB/s. O gargalo pode estar na rede e na máquina que executou o teste, e não no servidor. Os tempos representam a resposta do YouTube **para esta máquina, nesta rede**, e não a capacidade do servidor.

## Problemas encontrados durante o exercício

### 1. Taxa de erro de 59% (loop de redirects)

Na primeira execução, 59% das requisições falharam com:

```
Non HTTP response code: java.io.IOException
Exceeded maximum number of redirects: 20
```

**Causa:** o JMeter não guarda cookies por padrão. Sem eles, o servidor redirecionava cada requisição repetidamente até estourar o limite de 20 redirects.

**Solução:** adicionar o **HTTP Cookie Manager** (e, por boa prática, um **HTTP Header Manager** com `User-Agent` de navegador). Com isso, o erro caiu de 59% para 0%.

### 2. Teste encerrando em cerca de 57 segundos (OutOfMemoryError)

O teste parava antes dos 3 minutos. O log do JMeter mostrou:

```
java.lang.OutOfMemoryError: Java heap space
```

**Causa:** o listener **View Results Tree** guarda todas as respostas na memória. Com respostas de ~1,5 MB, a memória do JMeter se esgotou e as threads morreram.

**Solução:** desativar o View Results Tree na execução final. Os listeners Summary Report e Aggregate Report guardam apenas números e não causam o problema. Após o ajuste, o teste rodou os 3 minutos completos.

## Aprendizados

- O JMeter não mantém cookies sozinho: sites que dependem deles exigem o HTTP Cookie Manager.
- Taxa de erro alta pode vir do próprio script, e não do servidor. É preciso ler a mensagem do erro antes de concluir.
- O View Results Tree é útil para depurar, mas deve ficar desativado em testes com muitas requisições ou respostas grandes.
- Percentis (90%, 95%, 99%) mostram melhor o comportamento real do que a média isoladamente.
- Os resultados dependem do ambiente de execução (rede e máquina), e não só do sistema testado.
