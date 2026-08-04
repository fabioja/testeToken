# Plano: Sistema de QA Inteligente (Testes Unitários + E2E + Massa de Dados)

> Documento vivo — atualizar sempre que refinarmos ideias, decisões ou descartarmos abordagens.

## Contexto do projeto

- **Frontend:** Node.js, Angular e Next (React)
- **Backend:** Java
- **CI/CD:** GitHub + GitHub Actions
- **Infra:** AWS (ecossistema completo)
- **Ambientes:** dev → homologação → release → piloto → produção
- **Objetivo:** automatizar geração inteligente de testes unitários e E2E, com entendimento de regras de negócio, geração de massa de testes regressivos via MCP, e otimização de gasto de tokens de IA.

## Problema central

Não deve ser uma ferramenta única, e sim um conjunto de módulos que se comunicam (via MCP), cada um resolvendo uma dor específica do ciclo de QA.

---

## Forma de entrega da solução (product shape)

**Definição (2026-07-29):** a solução é um **agente de QA autônomo**, acionável pelo engenheiro, não apenas pela pipeline. O foco é automatizar **todo o processo de QA** e também ajudar ativamente o engenheiro na geração de testes unitários e E2E — não só sugerir, mas **atuar como agente de codificação**: escrever e incrementar testes (unitários, E2E e funcionais) quando necessário, e devolver relatórios/feedback sobre o estado do projeto.

**Interface primária: CLI.**
- Engenheiro aciona algo como `qa-agent gerar-testes --task=XPTO-123` ou `qa-agent incrementar --feature=checkout`.
- O mesmo CLI é reaproveitado pela pipeline (GitHub Actions chama o mesmo binário/comando) — sem duplicar lógica entre uso local e uso automatizado.
- Motivo da escolha: engenheiros já vivem em terminal/IDE; CLI integra nativamente com GitHub Actions; evita fricção de "sair do fluxo de trabalho para usar um portal".

**Plataforma web: camada secundária, não a interface de execução.**
- Não substitui o CLI. Consome os artefatos/relatórios que o agente produz (cobertura por regra, histórico, rastreabilidade regra→teste — isso é o Módulo 5).
- Entra depois do CLI estar validado, não no MVP.

**Capacidades esperadas do agente:**
1. Entender a solicitação do engenheiro (feature/task).
2. Buscar contexto (Módulo 1: camadas A/B/C).
3. Gerar ou **incrementar** testes existentes (unitários, E2E, funcionais) — como um agente de codificação de verdade, não um gerador que só propõe texto.
4. Gerar/gerir massa de dados quando necessário (Módulo 3).
5. Rodar os testes e reportar resultado.
6. Devolver **feedback/relatório** ao engenheiro: o que foi coberto, o que falhou, o que ficou incerto (marcações de confiança da Camada C se aplicável).

---

## Stack técnica do MVP (definida em 2026-08-03)

- **CLI:** Node.js/TypeScript. Motivo: mesma linguagem do frontend piloto (Angular/Next), SDKs de MCP mais maduros em TS, distribuição simples via npm/npx, reaproveitável direto como step de GitHub Actions sem duplicar lógica.
- **E2E:** Cypress — não Playwright, apesar de vantagens técnicas pontuais do Playwright (trace/codegen mais ricos). Decisão por **coerência com a suíte já existente**: o time já tem testes E2E em Cypress; gerar em outro framework criaria duas suítes paralelas, contra a própria lógica de "incrementar teste existente" e do conceito de skill técnica (seguir a convenção que o projeto já usa, não a que a ferramenta acha melhor).
- **Adapter de LLM model-agnostic:** interface própria (ex.: `LLMProvider.complete()`), com implementações plugáveis — desacopla toda lógica/prompt de um provedor específico, incluindo a ferramenta de IA interna da empresa (também via MCP), conforme já decidido.
- **Armazenamento de regra no MVP:** arquivos JSON locais em `.qa-context/regras/*.json`, versionados no próprio repo do projeto — sem `qa-knowledge-base` central Git ainda (isso é evolução pós-MVP). Antecipa o formato que o central usará depois, sem construir a infraestrutura cross-projeto agora.
- **`mcp-project-context`:** versão mínima do MCP server, servindo só `get_business_rules(feature)` a partir do `.qa-context/` local — sem índice SQLite/BM25 ainda (não é necessário no MVP: volume baixo, uma regra por vez, busca direta por task/arquivo já basta; índice de similaridade só se justifica quando o volume de regras crescer).

## Plano de execução (roadmap do MVP, definido em 2026-08-03)

1. **Esqueleto do CLI** — projeto Node/TS, comando `qa-agent gerar-testes --task=X` (parsing de argumento, sem lógica ainda).
2. **Camada A (task)** — CLI como client MCP consumindo a ferramenta de gerenciamento de projeto já disponível, busca o texto do task/critérios de aceite.
3. **Adapter de LLM** — interface model-agnostic + uma implementação real (o provedor que o time já usa hoje), suficiente pra desbloquear as próximas fases.
4. **Sessão de refinamento** — extrai regra(s) candidata(s) do task, sessão CLI síncrona em lote (formato já decidido), engenheiro confirma/ajusta, regra confirmada salva em `.qa-context/regras/*.json`.
5. **`mcp-project-context` mínimo** — MCP server local servindo `get_business_rules` a partir do `.qa-context/`.
6. **Geração do teste Cypress** — LLM gera o teste a partir da regra confirmada + exemplos de testes Cypress já existentes no repo (busca direta por convenção/pasta, sem RAG/índice ainda — desnecessário nesse volume).
7. **Execução + relatório de cobertura** — CLI roda o teste gerado (`cypress run`), captura resultado, e o agente compara o teste contra o enunciado completo da regra para emitir o relatório coberto/não coberto (mecanismo já decidido no Módulo 2).
8. **Validação ponta a ponta** — rodar o fluxo completo com 1 task real do projeto piloto (frontend Angular ou Next, ainda a escolher) como critério de aceite do MVP.

**Deliberadamente adiado para depois do MVP:** índice SQLite/BM25, `qa-knowledge-base` central em Git, bootstrap completo, detecção de conflito entre regras, skill técnica por projeto, canal WebView/SDK, integração via GitHub Actions na pipeline (Fase 8 valida local via CLI primeiro).

## Escopo do MVP (definido em 2026-08-03)

**Objetivo do MVP:** provar que dá pra ir de "regra de negócio validada" a "teste E2E regressivo executável" — o corte vertical fino já decidido, agora com escopo explícito.

**Dentro do MVP:**
- Um único projeto **web puro** (Angular ou Next) — **sem WebView** nessa primeira versão (WebView/SDK é evolução planejada, não descartada).
- Módulo 1 reduzido ao necessário para **uma regra nascer por vez**: task (Camada A) → sessão de refinamento com o engenheiro → regra confirmada. Sem bootstrap completo (sem varrer histórico de sprints passadas), sem `qa-knowledge-base` central cross-projeto, sem detecção de conflito entre regras — esses continuam no desenho geral, só não são pré-requisito do MVP.
- Módulo 2 reduzido a **geração de teste E2E regressivo novo** a partir da regra confirmada (Cypress/Playwright) — sem "incrementar" teste existente ainda (isso é evolução pós-MVP).
- CLI funcional de verdade, mesmo que com poucos comandos (ex.: `qa-agent gerar-testes --task=X`).

**Fora do MVP (evolução planejada, não descartada):**
- WebView/SDK própria (canal via bridge) — entra depois que o canal web direto estiver validado.
- Bootstrap completo, `qa-knowledge-base` central Git cross-projeto, detecção de conflito entre regras.
- Skill técnica por projeto.
- Incremento de teste existente (só geração nova no MVP).
- Testes unitários (Módulo 2 cobre só E2E no MVP; unitário é extensão natural depois).
- Massa de dados (Módulo 3), orquestrador de triagem (Módulo 4), dashboard (Módulo 5), plataforma web.

**Critério de sucesso do MVP (revisado em 2026-08-03):** o engenheiro pede geração de teste para 1 task real, passa pela sessão de refinamento, e recebe (1) um teste E2E **executável e correto** que valida a regra confirmada, e (2) um **relatório explícito de cobertura** mostrando quais condições da regra foram testadas e quais não foram — não basta o teste passar, precisa ficar claro o que da regra ainda não está coberto. Ver "Verificação de cobertura da regra de negócio" no Módulo 2.

---

## Módulo 1 — Knowledge Base do Projeto ("o cérebro")

**Problema:** a IA precisa entender regras de negócio e arquitetura sem reprocessar o repo inteiro a cada chamada (custo de tokens).

**Contexto real descoberto (2026-07-29):** o maior desafio não é código bem estruturado — é conhecimento **fragmentado, multi-projeto, sem padrão de arquitetura**, hoje concentrado na cabeça do time. Esse é o motivador principal da iniciativa: democratizar esse conhecimento. Por isso o Módulo 1 deixa de ser "componente de apoio" e passa a ser o produto central do esforço inicial.

### Estratégia: camadas de confiança (em vez de parser único por arquitetura)

Como não há padrão entre projetos, não dá para inferir regra de negócio "lendo o código" de forma genérica. A abordagem é hierárquica:

- **Camada A — alta confiança:** ferramenta de gerenciamento de projeto (via MCP já disponível). Histórias, critérios de aceite e regras descritas em texto são a fonte primária e mais barata de verdade.
- **Camada B — média confiança:** código-fonte tratado como **evidência**, não fonte de verdade — usado para linkar regra ↔ implementação (rastreabilidade: task → arquivos → PR).
- **Camada C — baixa confiança/inferida:** quando não há task nem doc, a IA infere a regra a partir do código, mas **marca explicitamente como "inferido, não confirmado"** — nunca vira verdade sem validação humana.

Consequência prática: o sistema não tenta entender "o projeto inteiro" (inviável sem padrão). Ele entende um recorte: **"a feature X, conforme task Y, tocando os arquivos Z"**.

### Fluxo de indexação

1. Começa sempre pelo **task** (via MCP da ferramenta de gerenciamento de projeto) — fonte primária.
2. Desce para o **código** a partir dos arquivos/PR vinculados ao task (Camada B).
3. Só recorre à inferência pura por código (Camada C) na ausência de task/doc.

### Infra e integração (ajustado ao contexto real)

- **Sem AWS por enquanto** (nada de pgvector/OpenSearch no piloto). Usar o que a ferramenta interna de IA já expõe via MCP (embeddings/busca semântica, se houver); senão, índice leve próprio (ex.: SQLite + busca por palavra-chave/BM25) — suficiente para o piloto, evita reinventar infra.
- **Model-agnostic desde o início:** nenhuma tool MCP deve depender de um provedor específico. Precisa manter compatibilidade com múltiplos modelos, **incluindo a ferramenta de IA interna da empresa** (também expõe MCP). Toda lógica/prompt deve ser desacoplada do modelo usado.
- Exposto via **MCP server próprio** (`mcp-project-context`), com tools como:
  - `get_business_rules(feature)` — busca primeiro na Camada A, cai para B/C se necessário, sempre indicando a camada/confiança da resposta.
  - `get_related_endpoints(entity)`
  - `get_test_history(module)`
  - `get_project_skill(area)` — retorna a skill técnica do projeto relevante à área (ex.: convenção de teste E2E, mock do SDK da WebView), complementar à regra de negócio. **Formato decidido (2026-08-02): Markdown com frontmatter (estilo `SKILL.md`)** — texto simples e diffável no Git, sem schema JSON próprio; o MCP só devolve o conteúdo como contexto, mantendo model-agnostic mesmo sem prender ao runtime específico do Claude Code.
- **Economia de tokens:** enviar ao modelo apenas o subgrafo relevante à feature (retrieval seletivo), não o repo inteiro nem o board inteiro da ferramenta de gerenciamento de projeto.
- **Teto rígido de contexto por operação (2026-08-02) — evita crescimento silencioso de token com a maturidade do projeto:**
  - **Teto por orçamento de token, não por contagem de item (2026-08-02):** em vez de fixar um K arbitrário (ex.: "top-10 regras"), o retrieval respeita um **orçamento máximo de tokens de contexto por operação** — corta regras candidatas da menos relevante pra mais até caber no orçamento. K vira consequência do orçamento, não uma constante escolhida a dedo (contagem de item não garante teto de custo real, já que regras têm tamanhos bem diferentes entre si).
  - **Valor inicial conservador, calibrado pelo piloto:** ainda não há dado real sobre o tamanho típico de regra/entidade nesse contexto, então o teto começa **propositalmente apertado** (fração pequena da janela de contexto do modelo em uso) e só é relaxado depois, com base em medição real no piloto — nunca o caminho inverso (começar permissivo e só perceber o estouro depois de consumido).
  - **Rede de segurança agregada no Módulo 4 (orquestrador):** além do teto por operação individual, um teto agregado diário/mensal de consumo de token corta ou alerta antes que um caso não previsto (ex.: bug no retrieval, entidade anômala) consuma o orçamento de um mês inteiro de trabalho de uma vez.
  - **Regra revogada não entra na comparação ativa:** toda regra com `substituida_por` preenchido é excluída por padrão do conjunto usado em comparação/conflito/geração de teste — só entra se alguém pedir histórico explicitamente (ex.: auditoria, Módulo 5). Sem isso, o conjunto "ativo" comparado a cada nova regra só cresceria.
  - **Nunca carregar `.qa-context/snapshot.json` inteiro num prompt:** toda leitura passa pelo índice local (SQLite/BM25) para filtrar antes de compor o prompt — o snapshot inteiro nunca é injetado bruto no contexto do modelo, independente do tamanho que o projeto atingir.
  - **Motivo:** a arquitetura de camadas + retrieval seletivo já evita reprocessar o projeto inteiro a cada operação, mas isso só se sustenta na prática se os limites acima forem regra explícita, não intenção implícita — sem teto formal, o consumo de token tende a crescer linearmente (não exponencialmente) conforme a base de regras amadurece.

### Inteligência incremental: bootstrap + atualização + detecção de conflito

**Fase 0 — Bootstrap (primeiro contato com o projeto):** ao rodar `qa-agent init` num projeto pela primeira vez, o agente varre o histórico existente (tasks fechadas/mescladas + código + testes já existentes) e monta uma **baseline versionada** de regras de negócio, usando as camadas A/B/C. Operação cara (mais tokens), mas acontece uma vez só por projeto. **Não é extração silenciosa:** o levantamento inicial é seguido de uma **sessão de refinamento com o engenheiro responsável**, cobrindo tudo o que o projeto já faz e o objetivo da entrega, para eliminar dúvidas que possam impactar negativamente o valor entregue.

**Regra geral de governança (2026-08-02):** o agente nunca extrai regra sozinho e some — **toda regra nova, em qualquer fase (bootstrap ou incremental), passa por uma sessão de refinamento com o engenheiro responsável** antes de ser confirmada na baseline. A IA propõe/extrai, o engenheiro valida/ajusta/decide em conversa — não é um "aprovar/rejeitar" de uma tela, é uma sessão de esclarecimento ativa. Esse é o mecanismo central de confiabilidade do Módulo 1: a regra só vira "fonte de verdade" depois de passar por esse refinamento humano, independente da camada (A/B/C) de onde veio.

**Fase 1 — Atualização incremental (cada nova solicitação):** cada regra de negócio é um **registro atômico versionado**, não texto solto:

```json
{
  "id": "regra-desconto-maximo",
  "entidade": "Pedido",
  "enunciado": "desconto máximo de 20% sem aprovação de supervisor",
  "camada": "A",
  "task_origem": "TASK-100",
  "vigente_desde": "2026-03-01",
  "substituida_por": null,
  "confianca": "alta"
}
```

Fluxo a cada nova solicitação/task:
1. **Extrai** regra(s) candidata(s) do novo task/diff.
2. **Recupera** regras existentes relacionadas à mesma entidade/domínio (busca por similaridade, recorte pequeno — mantém economia de tokens).
3. **Compara** regra nova vs. existentes recuperadas:
   - sem relação → adiciona como nova regra independente;
   - estende/detalha → vincula como refinamento;
   - **contradiz sem revogação explícita → flag de conflito**, reportado ao engenheiro (ex.: "TASK-450 propõe desconto de 30%, mas TASK-100 vigente define máximo de 20% sem revogação explícita — confirmar intenção").
4. **Sessão de refinamento com o engenheiro responsável** — sempre, não só em caso de conflito. É aqui que dúvidas são resolvidas e a regra é validada antes de virar baseline. **Formato decidido (2026-08-02):** CLI interativo síncrono como modo padrão, em **lote** (uma sessão cobre todas as regras candidatas da feature/task de uma vez, não pergunta por pergunta) — evita gargalo e dá visão de conjunto para notar contradição entre regras do mesmo lote. **Fallback assíncrono** via comentário estruturado no PR/task quando a geração é disparada pela pipeline sem engenheiro "ao vivo" no terminal (ex.: CI chamando o CLI no PR) — resposta em linguagem natural no próprio comentário.
5. Baseline só é atualizada após essa confirmação humana (ou revogação explícita no task), marcando a regra antiga com `substituida_por`. **Antes do commit final (2026-08-02):** o sync com o central roda **de novo** nesse ponto (não só no início da sessão) — se alguém mesclou uma regra semanticamente conflitante enquanto a sessão estava em andamento, isso reaparece como conflito (volta ao passo 3/sessão de refinamento) em vez de sobrescrever silenciosamente.

A comparação do passo 3 não exige modelo caro — é um diff semântico focado (regra nova vs. poucas regras candidatas da mesma entidade), não reanálise do projeto inteiro. Modelo caro só entra em caso de ambiguidade real.

Efeito colateral valioso: rastreabilidade histórica de "por que essa regra existe hoje" (alimenta Módulo 5) e proteção contra regressão de regra de negócio (reintrodução de comportamento antigo já revogado).

### Skills técnicas por projeto (complemento à regra de negócio)

**Ideia validada (2026-08-02):** além da regra de negócio (o "**quê**" de domínio), o agente também deve gerar e manter uma **skill técnica por projeto** — o "**como**" desse projeto especificamente: convenções de estrutura de teste, qual framework/padrão de mock é usado (ex.: o mock do SDK da WebView), onde ficam fixtures, como rodar testes localmente, padrão de nomenclatura. É um segundo tipo de conhecimento, com dinâmica diferente da regra:

| | Regra de negócio | Skill técnica do projeto |
|---|---|---|
| Pergunta que responde | O quê o sistema faz/deve fazer | Como esse projeto especificamente é construído/testado |
| Volatilidade | Alta — muda com frequência, pode ficar obsoleta | Baixa — convenção técnica muda raramente |
| Precisa de conflito/versionamento | Sim (Fase 1, passo 3) | Não tipicamente — mas precisa de **revalidação** quando o padrão do projeto muda de fato |
| Onde nasce | Task (Camada A) → código (Camada B) → inferência (Camada C) | Observação do código/estrutura existente durante o bootstrap |

**Por que isso economiza token de verdade:** sem skill gerada, o agente teria que **re-inferir a estrutura do projeto lendo código bruto** a cada geração de teste (ex.: "como esse projeto importa o mock do SDK? onde ficam os testes E2E existentes?"). A skill condensa isso uma vez, carregada como contexto pronto quando relevante — mesmo princípio de retrieval seletivo já aplicado à regra de negócio, agora aplicado a conhecimento procedural.

**Onde se encaixa no fluxo já desenhado:**
- **Gerada no bootstrap (`qa-agent init`)**, junto com a baseline de regras (Fase 0) — não é um mecanismo novo, é outro artefato do mesmo processo.
- **Passa pela mesma sessão de refinamento com o engenheiro** (governança já definida para regra de negócio) antes de virar "confiável" — evita que o agente aprenda um padrão errado do projeto e repita esse erro silenciosamente em toda geração futura.
- **Consumida pelo Módulo 2** (geração/incremento de teste) como contexto técnico, complementar à regra de negócio vinda do Módulo 1 — a regra diz "o que testar", a skill diz "como escrever esse teste do jeito que esse projeto já escreve".
- **Sujeita ao mesmo teto de contexto por operação** (orçamento de token) já definido para regra de negócio — não é uma exceção ao limite.

**Resolvido (2026-08-02):**
- **Formato de armazenamento:** Markdown com frontmatter (estilo `SKILL.md`) — ver detalhe na tool `get_project_skill` acima.
- **Onde versionar:** mesmo repositório central `qa-knowledge-base`, em pasta separada das regras (ex.: `regras/` vs. `skills/`) — reaproveita o mecanismo de sync (`git pull`) e a auditoria via commit já decididos para regra de negócio, sem criar infraestrutura nova. Efeito colateral positivo: abre caminho pra skills compartilhadas por stack (ex.: convenção comum a todo projeto Next.js), não só por projeto — sem precisar decidir isso agora.
- **Gatilho de revalidação:** a skill guarda **âncoras verificáveis** junto com cada afirmação (ex.: "usa Jest → `package.json`/`devDependencies.jest`"). No início de cada execução do Módulo 2, o agente confere a âncora — checagem barata, sem IA, sem revarredura periódica do projeto (mantém o princípio de economia de token já fechado). Se a âncora não bate mais (ex.: `jest` sumiu, `vitest` apareceu), a skill é marcada **stale** e entra pendente na próxima sessão de refinamento em lote — reaproveita o mesmo mecanismo de governança já usado para conflito de regra, sem criar um segundo processo paralelo.

### Onde armazenar o conhecimento: central + cache local (híbrido)

**Decisão (2026-07-29):** o conhecimento não fica só dentro de cada projeto — isso reproduziria a fragmentação que motivou a iniciativa (regra pode atravessar múltiplos repositórios, ex. frontend + backend).

- **Fonte de verdade = repositório/serviço central** (`qa-knowledge-base`), consolidando regras de todos os projetos. Permite detectar conflito de regra **entre projetos diferentes**, não só dentro do mesmo repo.
- **Cada projeto mantém `.qa-context/snapshot.json`** local, versionado no repo, como **cache/espelho somente-leitura** do subconjunto de regras relevantes àquele projeto — preserva auditabilidade via PR e uso offline, sem duplicar a fonte de verdade.
- O CLI sempre **sincroniza com o central antes de decidir** (evita detectar conflito com base em cache desatualizado).

Alternativas descartadas: (a) conhecimento só dentro de cada repo — reproduz fragmentação; (b) só central sem cache local — perde auditabilidade em PR e uso offline.

### Critérios do piloto inicial

- Escopo pequeno e bem delimitado (validação, não generalização ainda).
- Projeto com tasks bem preenchidas no de gerenciamento de projeto (para validar a Camada A primeiro).
- Volume pequeno: últimas 2-3 sprints, não histórico completo.
- Escolher **ou** frontend **ou** backend para o primeiro piloto, não os dois mundos ao mesmo tempo.
- **Decidido (2026-08-02): piloto no frontend (Angular/Next).** Testes regressivos gerados a partir de regras extraídas (E2E/componente via Cypress/Playwright/Jest), não backend Java como cogitado inicialmente na Camada B de exemplo.
- **Sem projeto piloto específico definido ainda (2026-08-02):** o conceito precisa funcionar para **qualquer** projeto Angular ou Next (genérico, não amarrado a um repo específico desde o desenho). A escolha de qual projeto validar primeiro fica para depois de o conceito estar desenhado.
- **Suporte a WebView nativa própria (2026-08-02, revisado no mesmo dia):** alguns frontends rodam empacotados dentro de uma WebView nativa (Android/iOS) de implementação própria do time (não Capacitor). A regra de negócio em si vive no código Angular/Next, **mas** dentro da WebView existe uma **camada de SDK/bridge própria que intermedia o acesso ao backend** (proteção/autenticação), diferente do fluxo web normal, que chama a API diretamente. Essa camada **pode alterar comportamento** (cache, permissões, rate limit etc.), não é só troca de transporte. **Consequência prática:** testar o frontend só em modo "web direto" não é suficiente — pode não pegar bugs que só se manifestam no caminho via SDK/WebView. O conceito de teste regressivo precisa contemplar **dois canais de comunicação com o backend** (direto vs. via SDK/WebView) como variável relevante, não assumir que WebView é um shell neutro.

---

## Módulo 2 — Agente de Geração/Incremento de Testes (Gerador de Casos de Teste Inteligente)

**Problema:** gerar E incrementar testes unitários, E2E e funcionais direcionados à feature entregue, não testes genéricos — atuando como agente de codificação, não só como sugestor.

**Ideia:**
- Acionável de duas formas: (1) diretamente pelo engenheiro via **CLI** (`qa-agent gerar-testes`/`qa-agent incrementar`), (2) pela pipeline no PR (GitHub Action chamando o mesmo CLI).
- Recebe o **diff** (ou a solicitação do engenheiro) + contexto do Módulo 1.
- Classifica o tipo de mudança (novo endpoint, alteração de regra, novo componente UI) e gera ou **incrementa testes existentes**:
  - **Backend (Java):** testes unitários JUnit/Mockito focados nos branches alterados (usar diff-coverage + Jacoco para saber o que falta cobrir).
  - **Frontend (Angular/Next):** testes de componente (Jest/Testing Library) + cenários E2E (Cypress/Playwright).
- **Redução de custo de IA:** gerar teste novo só para código sem cobertura no diff; reusar/adaptar testes existentes similares via RAG em vez de gerar do zero sempre.
- Output em uso local: código de teste escrito/incrementado diretamente + relatório de feedback ao engenheiro (o que foi coberto, o que falhou, o que ficou incerto/inferido).
- Output em uso via pipeline: PR de sugestão de testes — revisão humana ainda obrigatória nesse fluxo.

### Verificação de cobertura da regra de negócio (2026-08-03)

**Problema identificado:** gerar um teste E2E que passa não garante que ele cobre **todas as condições** da regra — uma regra como "desconto máximo de 20% sem aprovação de supervisor" tem múltiplas facetas escondidas (dentro do limite, exatamente no limite, acima sem aprovação, acima com aprovação). Um teste só do caminho feliz passaria sem validar a regra de fato.

**Decisão:** não decompor a regra em cenários previamente (nem na sessão de refinamento, nem por inferência automática da IA) — o mecanismo é **checagem no relatório final**, entregue junto com o teste gerado. Alternativas descartadas: (a) IA decompor a regra em cenários testáveis já na sessão de refinamento — mais robusto, mas adiciona uma etapa extra à sessão que já existe; (b) IA decompor sozinha sem confirmação humana — risco de a IA definir "o que é cobertura completa" sem validação, no mesmo espírito do porquê nenhuma regra é confirmada sem o engenheiro.

**Como funciona:** ao gerar o teste, o agente compara o teste produzido contra o **enunciado completo da regra** (não só o cenário principal) e relata explicitamente no feedback ao engenheiro:
- quais condições/branches da regra foram exercitadas pelo teste gerado;
- quais condições da regra **não** foram cobertas (gap explícito, não silencioso);
- o que ficou incerto/inferido (reaproveitando a mesma marcação de confiança já usada para Camada C).

O engenheiro decide, a partir desse relatório, se aceita a cobertura como está ou pede incremento — não é a IA que define sozinha "cobertura suficiente".

**Consequência no critério de sucesso do MVP:** passa a incluir não só "teste executável e correto", mas **relatório explícito de cobertura da regra** (coberto vs. não coberto) — ver seção "Escopo do MVP".

---

## Módulo 3 — Massa de Dados para Testes Regressivos

**Problema:** testes de regressão precisam de dados consistentes e realistas, próximos do produtivo.

**Decisão revisada (2026-08-03): reaproveitar o portal de massas já existente, não construir do zero.** O time já mantém um portal próprio que **gera massa de dados e faz seed de ambiente** — cobre exatamente as duas responsabilidades que o Módulo 3 se propunha a construir (`generate_mass` + `seed_environment` do desenho original). Construir um `mcp-test-data` novo do zero duplicaria capacidade que já existe e já é mantida por outro time.

**Estratégia: integrar, não reinventar (mesmo princípio já aplicado em "sem infra AWS no piloto").**
- O Módulo 3 passa a ser, na prática, um **wrapper MCP fino sobre o portal de massas existente** — expõe as tools que o agente de QA precisa (`generate_mass`, `seed_environment`, e o que mais for necessário) traduzindo para as chamadas reais do portal, em vez de implementar geração/seed próprios.
- **Se faltar recurso** que o agente de QA precise e o portal não oferecer hoje (ex.: reprodutibilidade determinística por seed fixo, snapshot/restore de dataset, geração de perfil que viola uma regra de propósito para teste negativo), o caminho é **solicitar aos mantenedores do portal**, não bifurcar em uma implementação paralela — mantém uma única fonte de verdade para massa de dados na empresa.
- IA só entra na conversa quando o dado sintético precisa respeitar uma regra de negócio específica do teste (ex.: gerar um pedido que *viole* a regra de desconto de propósito, para o teste negativo) — a geração determinística de volume continua sendo responsabilidade do portal.

**Efeito no roadmap:** o Módulo 3 deixa de ser "construir infraestrutura de dados" e vira "mapear o portal existente + abrir uma tool MCP de integração" — escopo bem menor do que o desenho original previa, e sem dependência de AWS própria para isso.

**Em aberto:** mapear com precisão o que o portal expõe hoje (API? apenas UI?), quais lacunas existem frente às necessidades do agente (determinismo, negativo de regra, snapshot/restore), e o canal certo para solicitar evolução aos mantenedores.

---

## Módulo 4 — Orquestrador de Execução (economia de tokens em runtime)

**Problema:** rodar tudo com IA em toda pipeline é caro.

**Estratégia em camadas:**
1. **Determinística (sem IA):** linters, testes existentes, cobertura — sempre roda, custo zero.
2. **Triagem leve (modelo pequeno/barato):** decide se a mudança precisa de novo teste E2E ou só unitário.
3. **Geração (modelo maior):** só entra se a triagem indicar necessidade real, usando contexto pré-filtrado do Módulo 1.
4. **Cache de decisões:** padrões de diff já vistos (ex.: getter/setter) não são reprocessados.

Implementação natural: GitHub Action com jobs condicionais, cada um chamando um agente diferente.

---

## Módulo 5 — Painel de Regressão e Rastreabilidade

**Problema:** saber o que está coberto, o que quebrou, por quê, e em qual ambiente.

**Ideia:**
- Dashboard cruzando:
  - Cobertura por feature/regra de negócio (não só % de linha).
  - Histórico de flakiness de testes E2E.
  - Status por ambiente na promoção da pipeline.
- Rastreabilidade regra de negócio → teste, retroalimentando o Módulo 1.

---

## Fluxo proposto (visão geral)

```
PR aberto → GitHub Action
   ├─ Job 1: lint/testes existentes (grátis)
   ├─ Job 2: triagem leve (modelo barato) → decide necessidade
   ├─ Job 3 (condicional): agente gera testes via MCP contexto + MCP massa de dados
   ├─ Job 4: roda testes gerados em ambiente dev (seed via MCP)
   └─ Job 5: relatório no PR + atualização do dashboard
```

Promoção dev → homolog → release → piloto → produção: cada estágio aciona o MCP de massa de dados com datasets diferentes (sintético em dev, anonimizado de produção em homolog/piloto).

---

## Decisões tomadas

- **Dor mais urgente identificada (2026-08-02):** regras de negócio perdidas/fragmentadas, especificamente no contexto de **testes regressivos** — garantir que uma feature nova não quebre/reintroduza comportamento de uma regra vigente. O valor percebido pelo time só aparece quando o teste regressivo é de fato **gerado/mantido**, não com um repositório de regras passivo nem só com o alerta de conflito isolado.
- **MVP é um corte vertical fino, não o Módulo 1 isolado:** Módulo 1 (mínimo necessário: extrair e versionar regras de um recorte pequeno) + Módulo 2 (mínimo necessário: gerar/manter teste regressivo a partir dessas regras). Entregar só o Módulo 1 completo (bootstrap + incremental + conflito + central) sem gerar nenhum teste não prova valor — é infraestrutura sem entrega perceptível ao engenheiro.
- ~~Começar pelo Módulo 1 isoladamente~~ (decisão anterior, revisada) — Módulo 1 continua sendo pré-requisito técnico, mas o corte de entrega/piloto deve incluir geração de teste regressivo desde o início, ainda que simplificado (ex.: sem bootstrap completo, sem central cross-projeto, sem detecção de conflito na primeira versão).
- **Escopo do MVP fechado (2026-08-03):** regra de negócio + teste **E2E regressivo**, projeto **web puro** (Angular/Next), sem WebView/SDK, sem incremento de teste (só geração nova), sem bootstrap completo/central/conflito/skill técnica. WebView entra como evolução planejada depois do canal web direto validado — ver seção "Escopo do MVP" para lista completa dentro/fora.
- **Fonte primária de regra de negócio = ferramenta de gerenciamento de projeto** (via MCP), não o código. Código é evidência (Camada B), não fonte de verdade.
- **Sem infra AWS no piloto** — usar MCP da ferramenta interna de IA e/ou índice leve próprio (SQLite/BM25).
- **Arquitetura model-agnostic obrigatória**, incluindo compatibilidade com a ferramenta de IA interna da empresa (também expõe MCP).
- **Piloto inicial pequeno e delimitado**: um projeto, um mundo só (frontend OU backend), últimas 2-3 sprints.
- **Forma de entrega: CLI como interface primária** (agente de codificação acionável pelo engenheiro), reaproveitado pela pipeline. Plataforma web é camada secundária de report (Módulo 5), não substitui o CLI e não é MVP.
- **Teto rígido de contexto por operação (2026-08-02):** top-K fixo na recuperação por similaridade (nunca "todas as regras da entidade"), regras revogadas (`substituida_por` preenchido) excluídas por padrão da comparação ativa, e `.qa-context/snapshot.json` nunca injetado inteiro num prompt — sempre filtrado via índice local antes. Motivo: sem esses limites explícitos, o consumo de token cresceria linearmente conforme a base de regras amadurece, mesmo com a arquitetura de retrieval seletivo já desenhada.
- **Formato da sessão de refinamento (2026-08-02):** CLI síncrono em lote (todas as regras candidatas da feature numa sessão só) como modo padrão + fallback assíncrono via comentário estruturado no PR/task quando não há engenheiro ativo no terminal (disparo pela pipeline). Alternativas descartadas: sessão pergunta-por-pergunta (risco de gargalo); só assíncrono via PR (perde o feedback imediato do fluxo CLI já decidido como interface primária).
- **O agente deve poder incrementar testes existentes** (não só gerar do zero) e devolver relatório/feedback ao engenheiro a cada execução.
- **Conhecimento é versionado como registros atômicos de regra** (não texto solto), com origem/camada/vigência/substituição — permite detecção de conflito entre regra nova e regra existente.
- **Armazenamento híbrido:** repositório central (`qa-knowledge-base`) como fonte de verdade cross-projeto + `.qa-context/snapshot.json` como cache local versionado em cada repo. Evita reproduzir a fragmentação por projeto isolado.
- **`qa-knowledge-base` central é um repositório Git** (não banco de dados): regras como arquivos JSON versionados, escrita via commit/PR, leitura via `git pull` + índice local (SQLite/BM25). Trade-off aceito: não escala bem para concorrência/queries relacionais complexas — irrelevante no piloto, revisitar se a base crescer muito.
- **Conhecimento técnico do projeto ("skill") é distinto de regra de negócio, mas passa pela mesma governança:** skill por projeto (convenções técnicas: estrutura de teste, mock do SDK, nomenclatura) complementa a regra de negócio (fatos de domínio). Gerada no bootstrap, validada na mesma sessão de refinamento com o engenheiro, consumida pelo Módulo 2 junto com a regra, sujeita ao mesmo teto de contexto por operação. Formato de armazenamento e gatilho de revalidação ainda em aberto.

## Próximos passos / pontos em aberto

- [ ] Detalhar tecnicamente o modo lote da sessão de refinamento (formato do prompt/tela que junta N regras candidatas de uma feature numa sessão só)
- [ ] Definir o formato do comentário estruturado que a IA posta no PR/task no fallback assíncrono (o que precisa estar nele pra permitir resposta em linguagem natural sem ambiguidade)
- [ ] Definir o que acontece se a regra fica pendente sem resposta no fallback assíncrono por muito tempo (bloqueia geração de teste? segue com confiança baixa/"inferido"? expira?)
- [ ] Definir o valor concreto do orçamento de token por operação (ponto de partida conservador) e o teto agregado diário/mensal do Módulo 4 — calibrar com dado real do piloto, não estimativa a priori
- [ ] Detalhar o schema de camadas de confiança (A/B/C) — como representar "confiança"/"origem" nos dados retornados pelo `mcp-project-context`
- [ ] Levantar quais tools o MCP da ferramenta de gerenciamento de projeto já expõe hoje (para não duplicar capacidade)
- [ ] Levantar quais tools o MCP da ferramenta interna de IA já expõe (embeddings? busca semântica? só chat?)
- [ ] Escolher o projeto piloto frontend específico (Angular ou Next, com tasks bem preenchidas, escopo pequeno) — só depois do conceito genérico estar desenhado
- [ ] Mapear o que exatamente a camada de SDK/proteção da WebView pode alterar (cache, permissões, rate limit, auth) — isso define o que precisa ser coberto além do teste "web direto"
- [ ] Desenhar como o agente gera/reusa o **mock do SDK da WebView** para testes regressivos: já existe mock respeitando o contrato frontend↔backend (usado nas fases iniciais dos projetos) — avaliar se dá pra reaproveitá-lo diretamente no Módulo 2, rodando o mesmo teste em dois modos (canal direto vs. canal mockado de SDK), reduzindo a necessidade de execução real em dispositivo/emulador
- [ ] Confirmar se o contrato frontend↔backend usado pelo mock é versionado/formalizado (ex.: schema/OpenAPI) — se sim, pode virar insumo direto do Módulo 1 (Camada B/evidência) para validar que o mock não ficou desatualizado em relação ao contrato real
- [ ] Definir o desenho de comandos do CLI (nomes, parâmetros, formato de saída/relatório)
- [x] ~~Definir stack do CLI~~ — resolvido em 2026-08-03: Node.js/TypeScript.
- [ ] Definir empacotamento/distribuição concreta do CLI (npm privado? GitHub Packages? binário standalone via pkg/nexe?)
- [ ] Levantar exemplos reais de teste Cypress do projeto piloto escolhido, para validar se "busca direta por convenção/pasta" (sem índice) é suficiente na Fase 6 do roadmap, ou se já precisa de algo mais estruturado
- [ ] Definir como o agente decide "gerar novo" vs "incrementar existente" na prática
- [x] ~~Definir onde/como hospedar o `qa-knowledge-base` central~~ — resolvido em 2026-08-02: repositório Git, escrita via commit/PR, permissão via controle de acesso do repo.
- [x] ~~Definir modelo de sincronização CLI ↔ central~~ — resolvido em 2026-08-02: `git pull`/`fetch` antes de cada decisão, sem TTL customizado.
- [x] ~~Definir resolução de divergência quando dois engenheiros geram regras conflitantes em paralelo~~ — resolvido em 2026-08-02: conflito **textual** (mesmo arquivo) já é resolvido pelo Git nativamente, sem lógica nova. Conflito **semântico** (regras em arquivos diferentes que se contradizem sem o Git perceber) é coberto reforçando o passo de sync-antes-de-decidir para rodar **de novo imediatamente antes do commit final**, não só no início da sessão — se algo mudou nesse intervalo, cai no mesmo fluxo de detecção de conflito já desenhado (Fase 1, passo 3) e volta pra sessão de refinamento em vez de auto-mergear.
- [ ] Definir formato exato do registro de regra versionada (schema completo, além do exemplo já esboçado)
- [ ] Detalhar formato do `.qa-context/` (schema do grafo de contexto, agora considerando as 3 camadas)
- [ ] Desenhar o fluxo task → arquivos/PR → evidência (Camada B) na prática
- [ ] Definir como fica marcada/exibida a informação "inferido, não confirmado" (Camada C) para quem consome
- [ ] Avaliar ferramentas de diff-coverage para Java (Jacoco + diff-cover) e frontend (Istanbul/nyc) — relevante para Módulo 2, mais à frente
- [ ] Definir política de dados anonimizados de produção para homolog/piloto (compliance/LGPD) — relevante para Módulo 3, mais à frente
- [ ] Mapear o que o portal de massas existente expõe hoje (API própria? só UI?) e como o wrapper MCP do Módulo 3 vai consumi-lo
- [ ] Levantar as lacunas do portal frente às necessidades do agente de QA (determinismo por seed fixo, snapshot/restore, geração de dado que viola regra de propósito) e o canal para solicitar evolução aos mantenedores
- [ ] Definir formato de armazenamento da **skill técnica por projeto** — arquivo próprio nosso, ou reaproveitar o formato nativo de skill do Claude Code (`SKILL.md`)?
- [ ] Definir o gatilho de **revalidação da skill técnica** quando a convenção real do projeto muda (ex.: troca de framework de teste) — como o agente percebe que a skill ficou desatualizada em vez de continuar aplicando um padrão morto
- [ ] Definir se a skill técnica fica versionada junto com as regras no `qa-knowledge-base` (mesmo repo Git) ou em local separado — e se o MCP `mcp-project-context` precisa de uma tool própria (ex.: `get_project_skill(area)`) além das já listadas

## Histórico de refinamento

- **2026-08-03:** revisado o Módulo 3 (massa de dados) — decidido **reaproveitar o portal de massas já existente** (gera massa e faz seed de ambiente) em vez de construir um `mcp-test-data` do zero. Módulo 3 passa a ser um wrapper MCP fino sobre o portal, com lacunas resolvidas via solicitação aos mantenedores em vez de implementação paralela. Mesmo princípio já usado em "sem infra AWS no piloto" — integrar em vez de reinventar.
- **2026-08-03:** definida a stack técnica do MVP e o roadmap de execução em 8 fases. CLI em Node.js/TypeScript (afinidade com frontend piloto, MCP maduro em TS, npm/npx pra distribuição e reuso em GitHub Actions). E2E em **Cypress**, não Playwright — decisão por coerência com a suíte já existente no time (evita duas suítes paralelas, alinhado ao conceito de skill técnica de seguir a convenção do projeto). Armazenamento de regra no MVP simplificado para JSON local em `.qa-context/regras/`, sem central Git ainda. `mcp-project-context` mínimo, sem índice SQLite/BM25 (adiado — só necessário com volume maior). Roadmap: esqueleto do CLI → Camada A via MCP do de gerenciamento de projeto → adapter de LLM model-agnostic → sessão de refinamento → MCP mínimo → geração de teste Cypress → execução + relatório de cobertura → validação ponta a ponta com task real do piloto.
- **2026-08-03:** identificado gap no critério de sucesso do MVP — teste E2E passar não garante cobertura de **todas** as condições da regra de negócio (regra pode ter múltiplas facetas: limite, acima, abaixo, exceção). Decidido não decompor a regra em cenários previamente (nem na sessão de refinamento nem por inferência automática) — o mecanismo é **relatório de cobertura no output final**, comparando o teste gerado contra o enunciado completo da regra e listando explicitamente o que foi coberto vs. não coberto, deixando a decisão de aceitar ou pedir incremento para o engenheiro. Critério de sucesso do MVP atualizado para incluir esse relatório, não só "teste passa".
- **2026-08-03:** fechado o escopo explícito do MVP: regra de negócio + teste E2E regressivo, projeto web puro (Angular/Next), sem WebView/SDK (evolução planejada, não descartada), sem incremento de teste (só geração nova), sem bootstrap completo/central Git/detecção de conflito/skill técnica — esses continuam no desenho geral do produto, só não são pré-requisito do MVP. Critério de sucesso definido: 1 task real → sessão de refinamento → teste E2E executável e correto.
- **2026-07-29:** primeira versão do plano criada, com os 5 módulos e fluxo geral definidos.
- **2026-07-29:** decidido iniciar pelo Módulo 1. Refinado com o contexto real (múltiplos projetos sem padrão, regras fragmentadas, ferramenta de gerenciamento de projeto com MCP, ferramenta de IA interna com MCP, sem AWS ainda, piloto pequeno). Introduzido o modelo de camadas de confiança (A: ferramenta de gerenciamento de projeto/task, B: código como evidência, C: inferido pela IA).
- **2026-07-29:** definida a forma de entrega do produto: agente de QA autônomo acionável via CLI (interface primária), atuando como agente de codificação capaz de gerar/incrementar testes unitários, E2E e funcionais, com relatório de feedback ao engenheiro. Plataforma web vira camada secundária de visibilidade (Módulo 5), não a interface de execução.
- **2026-07-29:** detalhado o mecanismo de inteligência incremental (bootstrap via `qa-agent init` + atualização contínua com detecção de conflito entre regras) e decidido o modelo de armazenamento híbrido: repositório central (`qa-knowledge-base`) como fonte de verdade cross-projeto + cache local versionado (`.qa-context/snapshot.json`) em cada repo.
- **2026-08-02:** explorado o armazenamento do `qa-knowledge-base` central — decidido usar **Git como fonte de verdade** (não banco de dados): regras como arquivos JSON versionados, escrita via commit/PR (auditoria de graça, sem sistema de permissão novo), leitura via `git pull` + índice local (SQLite/BM25) reconstruído a partir dos arquivos, `.qa-context/snapshot.json` como derivado filtrado do central. Trade-off aceito: Git não escala bem para concorrência/queries relacionais complexas, irrelevante no piloto. Também formalizado um **teto rígido de contexto por operação** (top-K fixo na recuperação por similaridade, exclusão de regras revogadas da comparação ativa, snapshot nunca injetado inteiro no prompt) para evitar que o consumo de token cresça conforme a base de regras amadurece — sem esses limites explícitos, a arquitetura de retrieval seletivo já desenhada não é suficiente sozinha.
- **2026-08-02:** incorporado o conceito de **skill técnica por projeto**, complementar à regra de negócio: enquanto a regra responde "o quê" (fato de domínio, volátil, precisa de conflito/versionamento), a skill responde "como" (convenção técnica do projeto — estrutura de teste, mock do SDK da WebView, nomenclatura —, estável, mas precisa de revalidação quando o padrão do projeto muda de verdade). Gerada no bootstrap junto com a baseline de regras, validada na mesma sessão de refinamento com o engenheiro (mesma governança já definida para regra), consumida pelo Módulo 2 e sujeita ao mesmo teto de contexto por operação — não é mecanismo novo, é outro artefato do mesmo processo. Adicionada tool `get_project_skill(area)` ao `mcp-project-context`. Também feita uma revisão geral do documento: dois itens em aberto (hospedagem do central e modelo de sincronização) foram marcados como resolvidos, já cobertos pela decisão de usar Git como central; identificada e registrada uma lacuna nova (resolução de divergência em escrita concorrente no `qa-knowledge-base`).
- **2026-08-02:** resolvidos os quatro gaps abertos sobre skill técnica e concorrência: (1) formato de armazenamento = Markdown com frontmatter (`SKILL.md`), model-agnostic por ser só texto; (2) versionamento = mesmo repositório central `qa-knowledge-base`, pasta separada das regras; (3) revalidação = âncoras verificáveis (ex.: dependência em `package.json`) checadas no início de cada execução do Módulo 2, sem revarredura periódica, marcando a skill como stale e reaproveitando a sessão de refinamento em lote quando a âncora não bate mais; (4) divergência em escrita concorrente = conflito textual resolvido nativamente pelo Git, conflito semântico coberto reforçando o sync-antes-de-decidir para rodar de novo imediatamente antes do commit final, não só no início da sessão.
- **2026-08-02:** revisitado o contexto para validar se o plano ataca o problema certo. Confirmado: a dor mais urgente é regra de negócio perdida/fragmentada especificamente no uso em **testes regressivos**; o valor só é percebido quando o teste regressivo é de fato gerado/mantido — não com repositório de regras passivo nem só alerta de conflito. Decisão revisada: MVP é um corte vertical fino (Módulo 1 mínimo + Módulo 2 mínimo), não o Módulo 1 completo isolado. Piloto definido como frontend (Angular/Next), sem projeto específico ainda — o conceito deve ser genérico para qualquer projeto Angular/Next. Revisado o entendimento sobre WebView nativa própria: não é um shell neutro — existe camada de SDK/bridge própria que intermedia acesso ao backend (vs. chamada direta no web normal) e **pode alterar comportamento** (cache, permissões, rate limit). O conceito de teste regressivo precisa cobrir os dois canais de comunicação, não assumir que testar só via web direto é suficiente. Ponto positivo descoberto: o time já mantém um **mock do SDK da WebView** respeitando o contrato pré-estabelecido frontend↔backend (usado nas fases iniciais de projeto) — provável reaproveitável no Módulo 2 para rodar o mesmo teste regressivo nos dois canais sem precisar de execução real em dispositivo/emulador. Definido também o mecanismo central de governança do Módulo 1: **toda regra nova (bootstrap ou incremental) passa por uma sessão de refinamento com o engenheiro responsável** antes de virar baseline — a IA propõe/extrai, mas nunca confirma regra sozinha. Não é aprovação de tela única; é sessão de esclarecimento ativa, cobrindo o que o projeto já faz e o objetivo da entrega, para eliminar dúvidas que impactariam negativamente o valor entregue. Formato definido: CLI síncrono em lote como padrão (uma sessão cobre todas as regras candidatas da feature) + fallback assíncrono via comentário estruturado no PR/task quando a geração é disparada pela pipeline sem engenheiro ativo no terminal.
