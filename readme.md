#TESTE1
####################
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
##
#
##
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
##
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
##
#
##
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
##
#
#
#
#
#
##
#

# Gestão de Disparos de Tagueamento com Contentstack

Sep 30, 2026 · atualizado em Oct 17, 2026 · @Fábio José Araújo

## Visão geral

O Contentstack passa a ser a fonte única de verdade do plano de tagueamento. A aplicação apenas captura sinais (este elemento foi visto, este elemento foi clicado) e um centralizador consulta a configuração do CMS para decidir se cada sinal vira um disparo e com qual payload. Este documento registra o modelo em refinamento, para que o time continue evoluindo e estressando a ideia.

**Problema.** Hoje os disparos de tagueamento ficam espalhados no código de cada projeto e canal (web e mobile), sem padrão comum. Mudar um parâmetro, desligar um evento ou incluir um novo exige deploy, e não há uma visão central do que é disparado em cada elemento.

**Objetivos.**

- Centralizar no Contentstack a definição de todos os disparos, organizados por projeto, tela e elemento.
- Permitir ligar, desligar, ajustar e criar disparos sem deploy das aplicações.
- Manter o mesmo contrato de payload (`event_name` + `parameters`) em web e mobile.
- Adotar o modelo de forma gradual em aplicações que já estão em produção, sem perder nem duplicar dados.
- Servir como documentação viva do plano para o time de dados.

**Princípios.**

- **Instrumentação universal, ativação por configuração:** o código captura os sinais de todo elemento identificado; o CMS decide quais viram disparo. O código não decide o que é enviado.
- **O `trackId` identifica o molde, não a instância:** itens repetidos (listas, carrosséis) compartilham o mesmo `trackId`; o que os diferencia é o contexto enviado junto com o sinal.
- **Vínculo não mandatório:** nenhum elemento é obrigado a ter tagueamento; sem configuração, nada quebra.
- **Migração sem ruptura:** em aplicações legadas, o disparo existente continua valendo até o CMS assumir o controle daquele elemento.
- **Componentes não conhecem ferramentas:** a aplicação fala só com o centralizador; GA4, Firebase e outras ficam atrás de adaptadores.
- **Desacoplamento:** textos da aplicação e tagueamento vivem em content types separados, com donos e ritmos de publicação próprios.

## Glossário

| Termo | Significado | Exemplo |
| --- | --- | --- |
| Tela | Uma tela ou rota da aplicação, identificada pela rota normalizada. | `home`, `produto/:id` |
| Elemento | Parte da tela que pode gerar disparo: banner, botão, card, lista. | Banner "Mais informações" da home |
| `trackId` / `elemento_id` | Identificador estável do elemento, usado no código e no CMS. Identifica o molde, não a instância. | `home.banner.mais-info` |
| Gatilho | Acontecimento técnico que a aplicação sabe capturar. Lista fechada, ligada ao código. | `screen_view`, `view`, `click` |
| Evento (`event_name`) | Nome enviado à ferramenta de analytics. Definido no catálogo do CMS, sem deploy. | `select_item`, `visualization` |
| Bloco de disparo | Configuração de um disparo dentro do Elemento: um gatilho + um evento + parâmetros. | Bloco Clique do banner |
| Contexto | Dados de execução que acompanham o sinal: tela, item da lista, lista. | `{ item: { id, nome, posicao } }` |
| Variável | Referência a um dado do contexto, usada nos parâmetros configurados no CMS. | `{{item.nome}}` |
| Modo de controle | Define quem manda no disparo de um elemento: código, CMS ou ninguém. | `codigo`, `cms`, `desligado` |
| Chave legado | Identificação de um disparo hardcoded existente, usada durante a migração. Aceita curinga. | `click \| card:*` |
| Plano compilado | JSON indexado e versionado, gerado a partir do CMS, que a aplicação consome. | `plano-loja-app-web.json` |
| Centralizador | Serviço da aplicação que recebe os sinais, aplica o plano e despacha os eventos. | `tracker.track(...)` |
| Adaptador | Camada que traduz o payload para cada ferramenta de analytics. | Adaptador GA4, adaptador Firebase |

## Arquitetura

A solução tem um lado de configuração (CMS e compilador) e quatro camadas na aplicação. Só a camada de captura exige código; todo o resto é dirigido pelo plano.

```
 CMS (Contentstack)                          Aplicação (web / mobile)
┌──────────────────────┐                ┌───────────────────────────────────────┐
│ Catálogo de Eventos  │                │ 1. CAPTURA (código)                   │
│ Telas                │                │  trackId + gatilhos genéricos         │
│ Elementos + blocos   │                │  → sinal {trackId, gatilho, contexto} │
└─────────┬────────────┘                │  (legado: interceptação do disparo    │
          │ webhook ao publicar         │   hardcoded → chave legado)           │
          ▼                             │               │                       │
┌──────────────────────┐   JSON/CDN     │ 2. MOTOR DE REGRAS                    │
│ COMPILADOR DO PLANO  │ ─────────────► │  localiza o elemento, aplica o modo,  │
│ valida, indexa e     │  (cache +      │  escolhe o bloco, resolve {{vars}}    │
│ versiona             │   snapshot)    │               │                       │
└──────────────────────┘                │ 3. POLÍTICAS: consentimento, dedup,   │
                                        │    fila offline, debug                │
                                        │               │                       │
                                        │ 4. ADAPTADORES: GA4, Firebase, ...    │
                                        └───────────────────────────────────────┘
```

1. **Captura:** cada elemento com `trackId` emite sinais para os gatilhos suportados, tenha configuração ou não. Em aplicações legadas, os disparos existentes também são encaminhados ao centralizador (ver *Adoção em aplicações em produção*).
2. **Motor de regras:** procura o elemento no plano, aplica o modo de controle, seleciona o bloco do gatilho e monta o payload.
3. **Políticas:** consentimento, deduplicação, fila offline e modo debug.
4. **Adaptadores:** traduzem o payload para cada ferramenta.

## Identidade dos elementos

O `trackId` segue a convenção `tela.tipo-elemento.nome`, em minúsculas e com hífen. A origem depende do tipo de elemento:

| Tipo de elemento | Origem do `trackId` | Exemplo |
| --- | --- | --- |
| Componente de UI (botão, card, banner fixo) | Atribuído pelo dev via prop ou atributo do design system | `home.botao.comprar` |
| Componente renderizado a partir do CMS | Derivado da entry (`content_type.entry_uid`) ou do `tag_id` dela | `banner.blt123abc` |
| Tela | Rota normalizada, sem valores dinâmicos | `produto/:id` |
| Disparo legado ainda sem `trackId` | Chave legado `event_name \| detail` | `click \| botao:mais-informações` |

Não usar texto do elemento, seletor CSS ou posição no DOM como identidade: quebram no primeiro ajuste de layout ou de texto.

## Elementos de lista

Itens repetidos compartilham o mesmo `trackId` e uma única configuração no CMS. A diferença entre eles vem do contexto, que entra no payload por meio de variáveis.

**No código**, o `trackId` é declarado uma vez na lista e cada item informa seus dados:

```tsx
// web
<ul data-tag-id="home.carrossel.ofertas">
  <li data-tag-item='{"id":"sku-123","nome":"Tênis X","posicao":1}'>...</li>
  <li data-tag-item='{"id":"sku-456","nome":"Bota Y","posicao":2}'>...</li>
</ul>

// mobile
<TrackedList trackId="home.carrossel.ofertas" items={ofertas} itemKey="sku" />
```

Ao clicar num card, o centralizador sobe até o `data-tag-id` mais próximo e junta o contexto do item:

```json
{
  "trackId": "home.carrossel.ofertas",
  "gatilho": "click",
  "contexto": {
    "item": { "id": "sku-456", "nome": "Bota Y", "posicao": 2 },
    "lista": { "id": "ofertas", "total": 10 }
  }
}
```

**No CMS**, o bloco Clique usa variáveis: `detail: card:{{item.nome}}`, atributo `item_id: {{item.id}}`, atributo `posicao: {{item.posicao}}`. O payload final sai com `detail: "card:Bota Y"`, `item_id: "sku-456"` e `posicao: 2`.

**Regras para listas.**

- **Posição não é identidade:** o índice não entra no `trackId` (os itens mudam de ordem, entram e saem). A posição vai nos parâmetros, como o GA4 modela listas (`item_list_id` e `index`).
- **Exceções por item:** um item específico pode ter configuração própria (desligado ou com `detail` diferente), identificado pelo valor da `chave_item`. A exceção vence a regra geral da lista. Se as exceções crescerem, é sinal de que esses itens são conteúdo do CMS e merecem identidade própria.
- **Visualização em dois níveis:** a visualização da lista dispara uma vez por exibição da tela; a impressão de cada item é deduplicada por `trackId + chave_item` dentro da mesma exibição. Opcionalmente, as impressões podem ser agrupadas em um único evento com array de itens (formato `view_item_list`).
- **Listas aninhadas:** o contexto é hierárquico (prateleira → card), e as variáveis seguem o mesmo caminho: `{{lista.nome}}`, `{{item.nome}}`.
- **Legado:** quando o `detail` hardcoded já traz o nome do item (`card:tenis-x`), a chave legado usa curinga (`click | card:*`) para que todos os itens casem com o mesmo elemento.

## Modelo de conteúdo no Contentstack

O CMS espelha a aplicação: **Projeto → Tela → Elemento → blocos de disparo**. Quem configura abre o elemento e vê todos os disparos dele, cada um com seu liga/desliga. O modelo usa três content types novos (Evento, Tela e Elemento), um global field (Parâmetros) e uma Taxonomy.

### Evento (catálogo)

Uma entry por `event_name`, compartilhada entre todos os projetos. Garante a governança dos nomes sem exigir deploy para criar um evento novo.

| Campo | Tipo no Contentstack | Obrigatório | Observação |
| --- | --- | --- | --- |
| `title` | Single Line Text (único) | Sim | Igual ao `event_name` |
| `event_name` | Single Line Text | Sim | Valor enviado à ferramenta. Ex.: `select_item` |
| `descricao` | Multi Line Text | Não | Quando usar este evento |
| `parametros_esperados` | Select múltiplo | Não | Ex.: `custom_path`, `detail`, `implementation_teams` |

### Tela

| Campo | Tipo no Contentstack | Obrigatório | Observação |
| --- | --- | --- | --- |
| `title` | Single Line Text (único) | Sim | `projeto \| tela_id`. Ex.: `loja-app \| home` |
| `projeto` | Reference → Projeto | Sim | Content type já existente |
| `tela_id` | Single Line Text | Sim | Rota normalizada: `home`, `produto/:id` |
| `canais` | Select múltiplo | Sim | `web`, `ios`, `android` |
| `disparos` | Modular Blocks | Não | Apenas o bloco **screen_view** |

### Elemento

Uma entry por elemento. Referencia a sua Tela e é classificada na Taxonomy `Projeto > Tela`, para filtrar a lista de entries por tela.

| Campo | Tipo no Contentstack | Obrigatório | Observação |
| --- | --- | --- | --- |
| `title` | Single Line Text (único) | Sim | `projeto \| elemento_id` (ou `projeto \| chave_legado` enquanto não houver `trackId`) |
| `tela` | Reference → Tela (single) | Sim | Liga o elemento à tela |
| `elemento_id` | Single Line Text | Não | O `trackId` usado no código. Preenchido quando o elemento for instrumentado |
| `chave_legado` | Single Line Text | Não | `event_name \| detail` do disparo hardcoded; aceita curinga `*` |
| `modo_controle` | Select | Sim | `codigo` (padrão na migração), `cms`, `desligado` |
| `tipo` | Select | Sim | `unico` ou `lista` |
| `chave_item` | Single Line Text | Se `lista` | Dado do contexto que identifica o item: `sku`, `entry_uid`... |
| `variaveis_disponiveis` | Multi Line Text | Não | Documenta as variáveis que o elemento expõe: `item.nome`, `item.posicao`... |
| `descricao` / `print` | Multi Line Text / File | Não | Ajuda o time de dados a reconhecer o elemento |
| `status` | Select | Sim | `legado`, `planejado`, `implementado`, `validado` |
| `disparos` | Modular Blocks | Não | Um tipo de bloco para cada gatilho (abaixo) |
| `excecoes` | Group (multiple) | Não | Só para `lista`: `valor_chave`, `gatilho`, `ativo`, `detail` |

Pelo menos um entre `elemento_id` e `chave_legado` deve estar preenchido; o compilador do plano valida essa regra.

### Blocos de disparo

Os tipos de bloco correspondem exatamente aos gatilhos que a aplicação sabe capturar. Cada bloco tem os campos de que o seu gatilho precisa, o que permite validação nativa no Contentstack (por exemplo, `detail` obrigatório no Clique).

| Bloco | Onde | Campos |
| --- | --- | --- |
| **screen_view** | Tela | `ativo`, `evento` (Reference → Evento), `canais`, `versao_minima_app`, Parâmetros |
| **Visualização** | Elemento | `ativo`, `evento`, `detail` (obrig.), `agrupar_itens` (listas), `canais`, `versao_minima_app`, Parâmetros |
| **Clique** | Elemento | `ativo`, `evento`, `detail` (obrig.), `canais`, `versao_minima_app`, Parâmetros |

Um gatilho novo (ex.: `submit`, `scroll_depth`) exige um novo tipo de bloco e implementação na camada de captura; as duas mudanças andam juntas.

### Global field Parâmetros

Reaproveitado em todos os blocos.

| Campo | Tipo | Observação |
| --- | --- | --- |
| `custom_path` | Single Line Text | Rota de disparo; aceita variáveis |
| `implementation_teams` | Single Line Text | Sobrescreve o valor padrão do projeto, se houver |
| `atributos_customizados` | Group (multiple) | Subcampos `chave` e `valor`; o valor pode ser fixo ou uma variável (`{{item.id}}`) |

### Projeto (já existente)

Pode guardar um `implementation_teams` padrão, usado quando o bloco não informar um. A entry de textos não recebe campos de tagueamento: componentes vindos do CMS usam o próprio `entry_uid` como identidade.

### Exemplo

```
Projeto: loja-app
└── Tela: home                              tela_id: home
    │   [screen_view]  ativo ✓  evento: screen_view
    │
    ├── Elemento: home.banner.mais-info       tipo: unico   modo: cms
    │     [Visualização]  ativo ✓  evento: visualization  detail: banner:mais-informações
    │     [Clique]        ativo ✓  evento: click          detail: banner:mais-informações
    │
    ├── Elemento: home.carrossel.ofertas      tipo: lista   chave_item: id   modo: cms
    │     [Visualização]  ativo ✓  evento: view_item_list  agrupar_itens ✓
    │     [Clique]        ativo ✓  evento: select_item     detail: card:{{item.nome}}
    │     exceções: sku-999 → Clique desligado
    │
    └── Elemento: click | botao:comprar       chave_legado  modo: codigo   status: legado
          (documenta o disparo hardcoded; o código continua mandando)
```

## Centralizador de disparos

Cada aplicação tem um único centralizador; os componentes nunca chamam GA4 ou Firebase diretamente.

**Contrato.**

```ts
// elementos instrumentados
tracker.track(trackId: string, gatilho: string, contexto?: object)

// disparos legados, encaminhados pela interceptação ou pela fachada
tracker.legacy(eventName: string, parameters: object)

// exemplos
tracker.track("home.banner.mais-info", "view");
tracker.track("home.carrossel.ofertas", "click", { item: { id: "sku-456", nome: "Bota Y", posicao: 2 } });
tracker.legacy("click", { detail: "botao:comprar", custom_path: "/home" });
```

**Resolução de um sinal.**

1. **Localizar o elemento:** pelo `trackId`; se não houver, pela chave legado (primeiro exata, depois com curinga).
2. **Aplicar o modo de controle:**
   - elemento não encontrado: sinal legado segue com o payload do código; sinal instrumentado é descartado;
   - `desligado`: nada é enviado;
   - `codigo`: segue o payload do código (o CMS apenas documenta);
   - `cms`: segue para o passo 3, e o payload do código é ignorado.
3. **Selecionar os blocos:** do gatilho, ativos, do canal atual e com `versao_minima_app` atendida. Um gatilho pode ter mais de um bloco (mais de um evento).
4. **Aplicar exceções:** em listas, a exceção do item (pela `chave_item`) vence a regra geral.
5. **Montar o payload:** junta parâmetros, atributos customizados e variáveis resolvidas do contexto no formato `{ event_name, parameters }`. Variável sem valor no contexto: o parâmetro é omitido e um aviso é registrado.
6. **Aplicar políticas e despachar** para as ferramentas por meio dos adaptadores.

**Payload final.**

```json
{
  "event_name": "select_item",
  "parameters": {
    "custom_path": "/home",
    "detail": "card:Bota Y",
    "implementation_teams": "nome-time",
    "item_id": "sku-456",
    "posicao": 2
  }
}
```

**Captura declarativa.**

- **Web:** elementos recebem `data-tag-id` (e `data-tag-item` em listas); um listener global de clique e um `IntersectionObserver` tratam `click` e `view`.
- **Mobile:** componentes wrapper (`TrackedButton`, `TrackedBanner`, `TrackedList`) recebem `trackId` como prop opcional.
- **screen_view:** disparado por um listener de navegação ou roteador, usando a rota normalizada.

**Políticas.**

- **Deduplicação:** visualização dispara uma vez por exibição da tela; impressões de itens, uma vez por `trackId + chave_item`.
- **Consentimento (LGPD):** nenhum envio antes do consentimento do usuário.
- **Fila offline (mobile):** guarda eventos sem conexão e envia depois.
- **Modo debug:** loga cada sinal, a regra aplicada (ou o motivo do descarte) e o payload; também lista os `trackId` vistos na tela, para alimentar o catálogo de Elementos.

## Plano compilado e entrega

As aplicações não consultam a Delivery API diretamente. Ao publicar Telas, Elementos ou Eventos, um webhook do Contentstack aciona o compilador, que:

1. Lê as entries publicadas do projeto.
2. **Valida** o que o Contentstack não consegue impor sozinho: elemento sem `elemento_id` e sem `chave_legado`, variável fora de `variaveis_disponiveis`, `chave_legado` duplicada, `lista` sem `chave_item`.
3. Gera um JSON indexado e versionado por projeto e canal, e publica em CDN.

```json
{
  "projeto": "loja-app",
  "canal": "web",
  "versao": "2026-10-17T12:00:00Z",
  "elementos": {
    "home.carrossel.ofertas": {
      "modo": "cms",
      "tipo": "lista",
      "chave_item": "id",
      "disparos": {
        "click": [{ "event_name": "select_item", "parameters": { "detail": "card:{{item.nome}}", "item_id": "{{item.id}}" } }]
      },
      "excecoes": { "sku-999": { "click": { "ativo": false } } }
    }
  },
  "legado": {
    "exato": { "click|botao:mais-informações": "home.banner.mais-info" },
    "curinga": [["click|card:*", "home.carrossel.ofertas"]]
  }
}
```

A aplicação baixa o plano ao iniciar, com cache e ETag, e embarca um snapshot no build como fallback. Assim o delivery token não fica exposto e o volume de Tags não pesa na aplicação.

## Adoção em aplicações em produção

As aplicações atuais têm disparos hardcoded, sem padrão e alimentando relatórios em uso. A adoção é gradual: o CMS assume o controle elemento por elemento, por meio do `modo_controle`.

| Modo | Comportamento |
| --- | --- |
| `codigo` (padrão na migração) | O disparo hardcoded segue como hoje. O CMS só documenta. |
| `cms` | O CMS manda: altera parâmetros, desliga ou cria disparos. O payload do código é ignorado. |
| `desligado` | Kill switch: nada é enviado, independentemente do código. |

**Fase 0: inventário (sem mexer na aplicação).** Levantar os disparos atuais por varredura no código (`gtag`, `dataLayer.push`, `logEvent`) e pelos eventos que chegam hoje no GA4 e no Firebase. O resultado é importado no Contentstack pela Management API, criando Telas e Elementos com `chave_legado`, `modo_controle: codigo` e `status: legado`. O CMS passa a documentar o que existe.

**Fase 1: ponto único de passagem.** Todo disparo passa a atravessar o centralizador, por um destes caminhos:

- **Interceptação (web):** envolver `window.gtag` ou `dataLayer.push` uma única vez; os disparos existentes viram `tracker.legacy(...)` sem refatorar as chamadas.
- **Fachada com codemod (web e mobile):** substituir as chamadas diretas por `tracker.legacy(eventName, parameters)`, uma troca mecânica e automatizável.

**Fase 2: modo sombra.** Para os elementos candidatos ao modo `cms`, o motor calcula o payload do CMS, apenas loga e compara com o que o código enviou. Quando baterem, o elemento muda de modo sem risco de diferença nos relatórios.

**Fase 3: controle pelo CMS e evolução.**

- Elementos validados passam para `cms`.
- **Quem mexe, migra:** telas novas ou refatoradas nascem com `trackId` e captura declarativa, sem disparo hardcoded; o Elemento ganha `elemento_id`.
- Disparos hardcoded são removidos à medida que os elementos estabilizam em `cms`.

**Mobile e versões antigas.** Versões sem centralizador continuam em uso por meses e seguem com o comportamento hardcoded. O campo `versao_minima_app` dos blocos garante que uma regra só valha onde o centralizador existe; o time de dados deve considerar essa mistura nos relatórios durante a transição.

## Fluxo de ponta a ponta

1. **Implementação (dev, uma vez por elemento):** o elemento recebe `trackId` e, se for lista, os dados de cada item; ou, no legado, seu disparo passa pelo centralizador.
2. **Cadastro (time de dados, a qualquer momento):** cria ou ajusta o Elemento e seus blocos no Contentstack, revisa e publica.
3. **Compilação:** o webhook gera e valida o plano e publica na CDN.
4. **Carga:** a aplicação baixa o plano (ou usa cache e snapshot).
5. **Disparo:** o usuário interage, a captura emite o sinal, o motor aplica o plano e os adaptadores enviam o evento.

A aplicação só envia quando o plano indica; caso contrário, segue sem erro e sem envio.

## O que exige deploy

| Ação | Deploy? |
| --- | --- |
| Ligar ou desligar um disparo, mudar parâmetros, trocar o evento | Não |
| Criar um `event_name` novo no catálogo | Não |
| Disparar num elemento que já tem `trackId` (ou chave legado) | Não |
| Usar uma variável de contexto que o elemento já expõe | Não |
| Passar um elemento de `codigo` para `cms` | Não |
| Instrumentar um elemento novo (dar `trackId`) | Sim, uma vez |
| Expor uma variável de contexto nova | Sim |
| Criar um gatilho técnico novo | Sim (e novo tipo de bloco no CMS) |

## Governança

**Convenções de nomes.**

- `trackId` / `elemento_id`: `tela.tipo-elemento.nome`, em minúsculas e com hífen. Ex.: `home.banner.mais-info`.
- `tela_id`: rota normalizada, com parâmetros no formato `:param`. Ex.: `produto/:id`.
- `event_name`: snake\_case, definido apenas no catálogo de Eventos.
- `detail`: `tipo:descricao`. Ex.: `banner:mais-informações`, `card:{{item.nome}}`.
- `chave_legado`: `event_name | detail`, com `*` para partes dinâmicas.

**Papéis.**

| Papel | Responsabilidade |
| --- | --- |
| Time de dados / analytics | Mantém o catálogo de Eventos; configura e valida Elementos e blocos; decide a troca de modo |
| Desenvolvimento | Atribui `trackId`, expõe o contexto, mantém centralizador, compilador e adaptadores |
| Conteúdo / produto | Mantém a entry de textos; não edita tagueamento |

**Workflow e ambientes.** Telas e Elementos passam por um workflow próprio (rascunho, revisão, aprovado) e são publicados primeiro em staging, para validação no modo debug, e depois em produção. O campo `status` acompanha o ciclo de vida do elemento e independe da publicação.

## Decisões e alternativas avaliadas

A decisão atual é uma entry por Elemento, agrupada por Tela, com um bloco por gatilho. Ela substitui a proposta anterior de uma entry por disparo.

| Alternativa | Vantagem | Por que não é a escolha atual |
| --- | --- | --- |
| Uma entry por disparo (Tag), proposta anterior | Workflow e versionamento por disparo | Fragmenta a visão do elemento; não permite validação por gatilho; o toggle `ativo` por bloco já dá controle individual |
| Todos os eventos em Modular Blocks numa única entry por projeto | Tudo visível num só lugar | Limite de itens por lista (o stack atual mostra 30), conflitos de edição e workflow valendo para o plano inteiro |
| Tagueamento dentro da entry de textos do projeto | Uma consulta só | Mistura donos, permissões e ritmos de publicação diferentes |
| Campo JSON (app JSON Editor do Marketplace) | Liberdade total de estrutura | Perde validação campo a campo e usabilidade para quem não é técnico |
| Índice do item no `trackId` (`home.carrossel.ofertas.2`) | Configuração por posição | Itens mudam de ordem; multiplica entries; posição é dado de análise, não identidade |
| Instrumentar tudo do zero antes de adotar | Modelo limpo desde o início | Inviável em aplicações em produção; substituído pela migração por modo de controle |

## Questões em aberto

Estes pontos ainda precisam de decisão ou de teste antes de o modelo ir para produção.

- [ ] **Google Tag Manager:** a web já usa GTM? Se sim, avaliar o CMS como fonte de verdade que alimenta o GTM, em vez de competir com ele.
- [ ] **Stacks em produção:** quais frameworks web e mobile (nativo, React Native, Flutter)? Define quantos centralizadores e se a interceptação é viável.
- [ ] **Disparos atuais:** passam por algum helper comum ou são chamadas diretas espalhadas? Define o esforço da Fase 1.
- [ ] **Piloto:** qual aplicação e tela usar para validar as Fases 0 a 2.
- [ ] **Compilador:** onde roda (BFF, função serverless, pipeline de CI) e quem é dono.
- [ ] **Catálogo de Elementos:** gerado automaticamente (varredura no CI ou modo debug) ou cadastrado pelos devs?
- [ ] **Volume:** tamanho esperado do plano por aplicação e impacto no cache do mobile.
- [ ] **Limite de itens:** confirmar se o limite de 30 vem da configuração do campo ou do plano do stack, considerando `excecoes` e `atributos_customizados`.
- [ ] **Atualização do plano:** frequência de recarga na aplicação (a cada abertura, por TTL) e versionamento do snapshot embarcado.
- [ ] **Parâmetros por ferramenta:** se GA4 e Firebase exigirem nomes diferentes, o mapeamento fica no adaptador ou no bloco?
- [ ] **Paridade web × mobile:** testes de contrato compartilhados (fixtures `plano + sinal → payload esperado`) que todas as implementações precisam passar.
- [ ] **Consentimento:** integração com a plataforma de consentimento já usada em cada aplicação.
- [ ] **Fase 2 do CMS:** app de sidebar do Marketplace para listar os elementos de uma tela, e uso do Live Preview / Visual Builder para abrir o elemento clicando na pré-visualização.




#TESTE2
####################
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
##
#
##
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
#
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
##
#
#
#
#
#
#
#
#
#
##
#
#
#
#
#
##
#

Pedro Smith PM

Quero aproveitar este momento para expressar minha gratidão pela sua atuação em nossa equipe. Na minha visão sua liderança tem sido fundamental para o sucesso da nossa squad e para o desenvolvimento de cada um de nós.

A forma como você se comunica, orienta e motiva a equipe faz toda a diferença. Sua capacidade de resolver problemas e manter todos alinhados é inspiradora. Agradeço por sempre estar disponível para ouvir nossas ideias e preocupações, criando um ambiente colaborativo e positivo.

Estou muito feliz em trabalhar sob sua liderança!

Muito obrigado por tudo!



Gui TCL

Quero aproveitar este momento para expressar minha gratidão pela sua atuação como nosso tech lead. Na minha visão a sua liderança é exemplar e seu vasto conhecimento técnico têm sido fundamentais para o meu crescimento e sucesso do nosso time.

Na minha opinião, a forma como você aborda cada desafio com criatividade e determinação é verdadeiramente inspiradora. Sua capacidade de transformar problemas complexos em soluções simples e eficazes demonstra não apenas sua expertise, mas também seu comprometimento com a excelência. 

Além disso, tudo que tenho presenciado sobre as suas habilidades em orientar e motivar a equipe é admirável. Você sempre encontra tempo para compartilhar seu conhecimento, responder às nossas dúvidas e oferecer feedback construtivo, criando um ambiente onde todos se sentem encorajados a aprender e crescer. Sua dedicação em cultivar um espírito de colaboração e camaradagem entre nós é algo que acredito que todos valorizam imensamente.

Agradeço sinceramente por você ser um líder tão acessível e por sempre estar disposto a ouvir nossas ideias. A sua visão clara e paixão pelo que faz nos inspira a dar o nosso melhor todos os dias.

Estou muito feliz em continuar aprendendo e tendo uma evolução contínua sob sua liderança!

Muito obrigado por tudo!
 


Pet SME


Quero expressar a minha gratidão pela sua atuação como nosso SME. Na minha visão, desde que você se juntou à nossa squad, tem sido evidente o impacto positivo que sua experiência e conhecimento estão trazendo para nossa equipe.

Sua capacidade de compartilhar insights valiosos e esclarecer pontos complexos tem feito toda a diferença no nosso trabalho diário. A maneira como você se envolve com todos os membros da equipe, sempre disposto a ajudar e esclarecer dúvidas, cria um ambiente colaborativo e acolhedor que todos nós apreciamos.

Além disso, suas sugestões e orientações têm nos guiado em direção a soluções mais eficientes e inovadoras. É inspirador ver como você se dedica a garantir que todos nós estejamos alinhados e bem informados, elevando o nível do nosso trabalho.

Agradeço por sua disposição em compartilhar seu conhecimento e por ser uma fonte de motivação para todos nós. Estou muito feliz em continuar aprendendo com você e sempre com foco em alcançar grandes resultados juntos!

Muito obrigado por tudo!

Raul CX

Quero expressar a minha gratidão pela sua atuação como nosso CX. Na minha visão as suas contribuições têm sido fundamentais para o sucesso da nossa squad e para a melhoria contínua dos nossos produtos e serviços.

A forma como você se empenha em entender as necessidades dos nossos clientes e traz essas informações para a equipe é admirável. Sua capacidade de traduzir feedbacks em ações concretas nos ajuda a criar experiências mais significativas e valiosas para nossos usuários.

Além disso, a maneira como você incentiva todos a pensar no cliente em cada etapa do processo é inspiradora e nos motiva a sempre buscar a excelência.

Agradeço por você estar sempre disposto a ouvir nossas ideias e por compartilhar seu conhecimento com todos nós. Estou feliz em trabalharmos juntos e sempre com foco em aprimorar cada vez mais a experiência do cliente!


Pedro Simas Eng. Backend

Quero expressar a minha gratidão pela sua atuação junto ao nosso time. Seu vasto conhecimento em infraestrutura tem sido um verdadeiro diferencial para o nosso time e tem contribuído significativamente para o sucesso dos nossos projetos.

Sua capacidade e entendimento para resolver problemas complexos dos nossos ambientes melhora muito a performance de atuações de todos do time.

Além disso, sua disposição em compartilhar suas experiências e ensinar os demais membros da equipe é algo que na minha visão, todos do time aprecia imensamente. Sua paciência em esclarecer dúvidas e seu olhar atento aos detalhes ajudam a criar um ambiente colaborativo onde todos se sentem valorizados e motivados a aprender.

Agradeço por sua dedicação e por sempre estar pronto para ajudar. Estou muito feliz em estarmos trabalhando juntos e sempre buscando alcançar resultados melhores a cada nova atuação!

Muito obrigado por tudo!

Bruno Eng. Backend

Quero agradecer de coração pelas suas atuações ao nosso time. Sua dedicação em aprender e ajudar os colegas faz toda a diferença.

É incrível ver como você sempre está disposto a compartilhar seu conhecimento e esclarecer dúvidas. Isso na minha visão cria um ambiente de trabalho muito mais colaborativo e produtivo.

Obrigado por estar sempre ao nosso lado e por contribuir com o nosso time. Estou muito feliz em estarmos trabalhando juntos e sempre com foco na evolução contínua.

Muito obrigado por tudo!

Nath Eng. Frontend 

Quero aproveitar este momento para expressar minha gratidão pela sua contribuição ao nosso time. Já temos trabalhado juntos há algum tempo, e na minha visão a sua presença é verdadeiramente valiosa não só para mim, como para todo o time.

Sua disposição em ajudar e colaborar é admirável. Você sempre traz um olhar novo e criativo para os desafios que enfrentamos, e suas ideias têm sido fundamentais para o sucesso dos nossos projetos. É inspirador ver como você se dedica a aprender e crescer na sua área de atuação, e isso impacta positivamente todo o time.

Além disso, suas atitudes positivas e seu espírito de equipe tornam o ambiente de trabalho muito mais agradável. Você tem o talento de motivar os colegas ao seu redor, e na minha visão, isso é algo precioso.

Agradeço sinceramente por estar sempre ao nosso lado, apoiando e contribuindo com seu conhecimento. Estou muito feliz em continuarmos evoluindo juntos!

Muito obrigado por tudo!

Kai  Estag. PM

Quero dedicar um momento para agradecer pela sua contribuição ao nosso time. Sua disposição em ajudar e colaborar tem sido fundamental para o nosso sucesso.

É admirável ver como você demonstra tanta proatividade e comprometimento em entender as necessidades do time e dos nossos projetos. Seu olhar atento e suas ideias sempre acrescentam valor às nossas discussões.

Agradeço por estar sempre pronta para apoiar os colegas, esclarecer dúvidas e buscar soluções. Sua atitude positiva e sua vontade de aprender são inspiradoras, e tenho certeza de que você está construindo uma base sólida para uma carreira brilhante.

Estou muito feliz de trabalharmos no mesmo time! 

Muito obrigado por tudo!


Lucas Eng. Backend 

Quero aproveitar este momento para expressar meu agradecimento pela sua contribuição ao nosso time. Apesar de ainda termos tido pouco contato, sempre que preciso de sua ajuda você está sempre disposto a ajudar. Na minha visão o seu comprometimento e a vontade de colaborar são notáveis e contribuem para um ambiente de trabalho mais produtivo e positivo. Estou feliz em como as suas atuações estão enriquecendo nossos projetos.

Muito obrigado por tudo!


Elen Eng. Frontend

Quero tirar um momento para agradecer pela sua contribuição ao nosso time, mesmo com o pouco tempo que estamos trabalhando juntos, é sempre bom ter novas pessoas na equipe, e sua presença faz diferença.

Seu talento e habilidades estão agregando muito ao nosso trabalho.

Agradeço por estar conosco e espero que possamos continuar construindo um ótimo trabalho em equipe!

Muito obrigado por tudo!

Jon Eng. Backend 

Quero agradecer pelas suas contribuições para mim e ao nosso time. Quero expressar minha gratidão pela sua disposição em ajudar o time. Sua experiência e apoio têm sido fundamentais para o nosso sucesso, e é ótimo saber que podemos contar com você em todos os momentos. 

Agradeço por compartilhar seu conhecimento e por estar sempre pronto para colaborar. Sua contribuição faz uma grande diferença na nossa squad!
Muito obrigado por tudo!






Allan Eng. Frontend 

Quero aproveitar este momento para expressar minha gratidão pelas suas contribuições, tanto para mim quanto para o nosso time. Com todo o tempo que trabalhamos juntos, é sempre inspirador ver sua proatividade e disposição em ajudar o time.

Na minha visão, sua habilidade em encontrar soluções e oferecer suporte sempre que necessário faz uma grande diferença no nosso trabalho. Agradeço por compartilhar seu conhecimento e por estar sempre pronto para colaborar, tornando nosso ambiente mais produtivo e positivo.

Estou muito feliz em estar nessa jornada ao seu lado e enfrentar novos desafios juntos!

Muito obrigado por tudo!

prompt PDI
preciso elaborar um PDI para area de desenvolvimento de software, esse deve incluir processos de automações no próprio desenvolvimento de software com uso de io, como implementações de feature com uso de agentes, incluindo os fluxo de teste unitários e e2e, consegue segerir um roadmap com base nesses reguisitos do PDI e também um pdi com base no SWOT?

"Atue como um Mentor Sênior de Engenharia de Software e Especialista em Carreira Tech. Preciso que você elabore um Plano de Desenvolvimento Individual (PDI) detalhado para os próximos 12 meses, focado na transição para um perfil de AI Software Engineer / Automation Expert.

Meu objetivo principal: Dominar a automação do ciclo de vida de desenvolvimento (SDLC) utilizando IA Generativa e Agentes Autônomos.

O que o PDI deve contemplar obrigatoriamente:

Agentes na Implementação de Features: Uso de frameworks (ex: LangChain, CrewAI ou AutoGPT) para criar agentes que escrevem código baseado em requisitos.

Automação de Testes com IA: Fluxos de geração automática de testes unitários e testes End-to-End (E2E) que se auto-ajustam a mudanças na UI.

Roadmap Técnico: Dividido por trimestres (Q1 a Q4), com marcos de aprendizado e projetos práticos.

Análise SWOT Pessoal: Crie uma matriz SWOT hipotética para um desenvolvedor de software que busca essa especialização e, com base nela, gere ações práticas para o PDI (ex: como usar as Forças para mitigar as Ameaças do mercado).

Formate a saída com tabelas, cronogramas claros e sugestões de ferramentas específicas."
