# 📝 Low-Code Speed, Bare-Metal Performance: Como convertemos 126 workflows do n8n para Node.js/Fastify com latência de 5ms e 0-deps

*Por Rogério Maciel & Antigravity (CoreAutoCRM Team)*

> 🌐 **Estudo de Caso & Teardown Oficial no Ar:**  
> Acesse a análise técnica interativa e benchmarks de produção em:  
> [rogeriomaciel.com.br/solucoes/transpilar-n8n-microsservicos-fastify/](https://rogeriomaciel.com.br/solucoes/transpilar-n8n-microsservicos-fastify/)

---

## 🎯 1. O Paradoxo do MVP Moderno

No ecossistema de startups e engenharia de software moderno, os times enfrentam um dilema cruel ao escolher sua pilha tecnológica inicial:

- **A Abordagem Low-code / No-code (ex: n8n, Make, Flowise)**: Entrega uma velocidade de entrega imbatível no início. Você desenha fluxos visuais em minutos, integra APIs e valida a proposta de valor da startup em tempo recorde.
- **O Teto Técnico em Produção**: Conforme o produto ganha tração e milhares de requisições por minuto começam a entrar, o teto aparece: latências de 150ms–500ms por nó devido à serialização pesada de JSONs, consumo explosivo de RAM, falta de testes de regressão unitários e a impossibilidade de usar versionamento real com Git diffs e code reviews.

A maioria das empresas responde a esse dilema com uma decisão dolorosa: **reescrever tudo do zero em código tradicional**. Essa reescrita costuma durar de 6 meses a 1 ano, paralisa o lançamento de novas funcionalidades e destrói o ritmo de inovação da empresa.

> **E se não precisássemos escolher entre velocidade e escala?**  
> Foi para resolver esse paradoxo que desenvolvemos o **CoreAuto Transpiler**: uma máquina de compilação que lê a representação declarativa dos workflows n8n (`.json`) e os transpila automaticamente em uma **aplicação Node.js / TypeScript nativa e autônoma**, rodando sobre o framework Fastify.

---

## 🛑 2. O Calcanhar de Aquiles das Ferramentas Visuais: O Versionamento em Git

Desenvolver sistemas críticos em interfaces visuais sem arquivos locais é um risco operacional enorme:
1. **Sem Histórico de Diff Real**: Como saber quem alterou a condição de um `Switch` ou a query SQL de um nó de banco de dados há duas semanas?
2. **Sem Esteira de CI/CD**: Como impedir que uma alteração em staging quebre um subworkflow silenciosamente em produção?
3. **Branches Concorrentes**: Como dois desenvolvedores podem trabalhar em features diferentes do mesmo workflow ao mesmo tempo?

Ao trazer a especificação `.json` dos workflows para dentro do repositório Git, estabelecemos o verdadeiro **DevOps para Low-code**:
- Cada alteração visual gera um arquivo JSON versionável no Git.
- Revisões de código (Pull Requests) reais são feitas antes do merge.
- O pipeline de CI (GitHub Actions) intercepta o commit e executa o transpilador nativo.

---

## 🧠 3. Sob o Capô: A Engenharia do Transpilador

O transpilador opera como um compilador de linguagem clássico estruturado em cinco etapas:

```
[n8n JSON Graphs] ➔ [DAG Parser (Kahn)] ➔ [Expression Parser] ➔ [Node Mappers] ➔ [Fastify Native TS]
```

### A. Ordenação Topológica por DAG (Algoritmo de Kahn)
Workflows visuais são Grafos Direcionados Acíclicos (DAGs). O transpilador mapeia as conexões `main` entre nós e executa a Ordenação Topológica via Algoritmo de Kahn:
- Garante que nós predecessores (como autenticação JWT ou busca de dados no banco) sempre executem antes dos nós consumidores.
- Posiciona nós do tipo **SINK** (como `Respond to Webhook`) no momento exato em que o payload da resposta HTTP está consolidado, permitindo que a resposta ao cliente seja devolvida em **< 5ms**, enquanto tarefas assíncronas secundárias prosseguem em background.

### B. Execução de Subworkflows em Memória (`Global Singleton Registry`)
No n8n, o nó `Execute Workflow` aciona subworkflows via eventos de rede ou subprocessos internos.  
No nosso ambiente transpilado, eliminamos 100% da latência de rede atrelando todas as funções de subworkflow a um **Global Singleton Registry** em memória (`globalThis.__COREAUTO_WORKFLOWS_REGISTRY__`):

```ts
// Chamada direta de função assíncrona em memória
const result = await executeSubworkflow("COREAUTOCRM-PANEL-ACTION-GET-OS-DETAILS", inputItems, ctx);
```
Sem chamadas HTTP internas, sem re-autenticação e com overhead zero de transporte.

### C. Parser de Expressões `={{ ... }}` e Safe Chaining (`?.`)
Expressões dinâmicas do n8n como `={{ $('Pessoa').item.json.user.loja_id }}` são convertidas sintaticamente para encadeamento opcional profundo nativo:

```ts
(ctx.getNode('Pessoa').first().json?.user?.loja_id)
```
Isso blinda a aplicação contra exceções runtime (`TypeError: Cannot read property of undefined`), garantindo resiliência de execução.

---

## 🧪 4. O Santo Graal: Testes Unitários de Regressão Automáticos & Autocorreção

Para garantir que a aplicação TypeScript gere exatamente a mesma resposta entregue pelo n8n, o pipeline conta com um gerador automatizado de testes unitários (`TestGenerator`):

1. **Captura de Gabarito via API do n8n**: A API REST do n8n (`GET /api/v1/executions`) fornece os payloads de entrada e os resultados JSON reais de cada nó.
2. **Geração de Testes Unitários Nativos**: O gerador cria arquivos `.test.ts` que executam as funções TypeScript geradas usando o **Bun Test / Vitest**.
3. **Validação de Equivalência**: O teste garante que o retorno do Fastify seja **100% equivalente, campo a campo**, ao gabarito do n8n antes de autorizar o deploy em produção.

---

## 📈 5. Resultados e Métricas de Impacto (Dados Reais em Produção)

Após submeter ambos os ambientes a um teste de carga contendo 10 requisições simultâneas em paralelo (simulando acessos concorrentes ao painel de OS) originadas do Brasil contra o servidor em Ohio (EUA):

| Métrica | n8n Tradicional em VPS | Aplicação Transpilada Fastify | Ganho de Eficiência |
| :--- | :--- | :--- | :--- |
| **Tempo Total (10 Req Paralelas)** | 6.852 ms (6,8s) | **998 ms (0,9s)** | **~6.87x mais rápido na ponta do cliente** |
| **Latência de Processamento Interno** | ~400ms – 600ms | **~16ms** | O gargalo passa a ser apenas o ping físico da rede |
| **Consumo de Memória RAM (VPS)** | > 1.2 GB (picos) | **~25 MB** (PM2 Cluster) | **Redução de ~98%** |
| **Sobrecarga de Concorrência** | Enfileiramento severo | **Assíncrono Não-bloqueante** | Escala horizontal ilimitada |
| **Tempo de Transpilação (126 WFs)** | N/A | **~77 milissegundos** | Instantâneo |

---

## 💡 Conclusão: A Startup "1000% Evolutiva"

A arquitetura desenvolvida no **CoreAutoCRM** prova que o futuro do desenvolvimento de software não está na disputa entre Low-code e Código Nativo, mas sim na sua **integração simbiótica**:

> **Desenvolva visualmente a 100 km/h no n8n. Transpile e execute em Bare-Metal Fastify a 1.000 km/h em produção.**

Essa abordagem entrega a velocidade de inovação de um MVP com a solidez de engenharia de um unicórnio.

---

## 💼 Consultoria & Assessoria de Arquitetura

Precisa eliminar gargalos de memória e latência nas automações da sua empresa ou deseja implantar essa arquitetura no seu negócio?

👉 **[Trocar uma ideia diretamente com Rogério Maciel no WhatsApp](https://wa.me/5562996232227?text=Ol%C3%A1%20Rog%C3%A9rio%2C%20vi%20o%20reposit%C3%B3rio%20coreautocrm-architecture%20no%20GitHub%20e%20gostaria%20de%20conversar%20sobre%20arquitetura%20de%20automa%C3%A7%C3%B5es.)**  
👉 **[Conheça o Portal Oficial de Soluções: rogeriomaciel.com.br](https://rogeriomaciel.com.br/solucoes/n8n-travando-memoria-solucao/)**

