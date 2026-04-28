# Claude Code Skills

> Pacotes de conhecimento operacional para o [Claude Code](https://claude.com/claude-code) que substituem princípios abstratos por **regras mecânicas com tells verificáveis** — fixando a oscilação de qualidade que ocorre quando o modelo recebe instruções como "siga SOLID" ou "escreva código limpo".

Mantido por **Tarcísio Quaresma** · [@tarcisioq](https://github.com/tarcisioq)

---

## O problema que essas skills resolvem

Ao instruir o Claude com diretrizes abstratas — *"aplique SOLID"*, *"escreva código limpo"*, *"use TDD"* — a qualidade do output **oscila entre sessões**. Em uma execução, o código passaria em revisão; na próxima, padrões que um senior reprovaria sem hesitar. A causa não é capacidade do modelo: é que **princípios abstratos admitem múltiplas interpretações**, e o modelo pondera entre elas a cada contexto.

Estas skills resolvem isso através de **regras mecânicas** — diretrizes verificáveis através de tells concretos: contagem de linhas, profundidade de indent, número de instance variables, padrões sintáticos específicos. O modelo não interpreta — ele aplica a regra.

```diff
- "Single Responsibility Principle: a class should have one reason to change"
+ "Split a class when ANY fires:
+   - > 50 linhas
+   - > 2 instance variables
+   - > 5 public methods
+   - method names span 2+ verb domains (save*, validate*, format*)
+   - description requires 'and'"
```

A primeira instrução produz interpretações divergentes. A segunda produz output reproduzível.

---

## Volume e maturidade

| Métrica | Valor |
|---------|-------|
| Skills publicadas | 2 (`solid` 2.2.0, `vanilla-js-architect` 2.2.0) |
| References operacionais | 28 |
| Linhas de conhecimento estruturado | ~13.900 |
| Anti-patterns documentados (com `❌` + Tell + Fix) | 27+ |
| Decision trees | 50+ |
| Refactor recipes (Fowler-style, steps numerados) | 19 |
| Tells mecânicos com thresholds quantitativos | 100+ |
| Cobertura de runtimes | Browsers, Node 18+, Workers, Deno, Bun, Cloudflare Workers |

Cada skill é versionada (semver), tem changelog detalhado, dev docs separados, processo de feedback loop, e review checklist auto-aplicável.

---

## Skills

### [`solid/`](./solid/) — Engineering Discipline (Cross-Language)

Operacionaliza **SOLID + TDD + clean code + refactoring + design patterns + arquitetura + code review + legacy code + when-not-to-apply** em uma única skill cross-language.

**Quando dispara:** qualquer task de implementação, refactor, design, code review ou debug em linguagem orientada a objetos.

**Diferenciador:** `references/when-not-to-apply.md` define discipline levels (0-3) com triggers de promote/demote. Evita o fundamentalismo que faz "best practices" virarem ruído em scripts throwaway, spikes ou MVPs.

**Conteúdo:**

- **14 references operacionais** (~7.900 linhas)
- **15 anti-patterns** no SKILL.md com tells de detecção mecânica (inclui mutação através de `await` boundary)
- **8 decision trees** principais: TDD ou não, split classe, compose vs inherit, throw vs Result vs null, refactor agora vs depois, usar pattern, mock vs fake vs real, **propagação de cancelamento (`AbortSignal`/`CancellationToken`/`context.Context`) através de cada layer de `await`**
- **19 refactor recipes** Fowler-style com steps numerados (Extract Method, Replace Conditional with Polymorphism, Replace Primitive with Value Object, etc.)
- **TDD avançado:** London/Detroit, double-loop TDD, ATDD, mock vs fake decision tree, mutation testing, property-based testing, transformation priority premise
- **Métricas operacionais:** cyclomatic complexity ≤ 10, cognitive ≤ 15, nesting ≤ 2, instability metric (`I = Ce / (Ca + Ce)`), churn × complexity hotspot detection
- **Legacy code playbook** (Michael Feathers): characterization tests, seams (object/preprocessing/link), sprout method/class, wrap method/class, Strangler Fig pattern
- **Architecture decisions:** modular monolith → microservices decision tree, hexagonal vs clean vs layered, Conway's Law operacional, ADR templates

**Linguagem dos exemplos:** TypeScript. Regras universais aplicam a Java, C#, PHP, Python e qualquer linguagem OO.

---

### [`vanilla-js-architect/`](./vanilla-js-architect/) — Browser SDKs in Vanilla JavaScript

Designs e implementa **SDKs, bibliotecas e frameworks browser-side** em vanilla JS — com foco em arquitetura, encapsulamento, distribuição (ESM/UMD/IIFE) e suporte cross-runtime.

**Quando dispara:** *construção* (não consumo) de SDK / biblioteca / framework JavaScript para browser e ambientes Web Standards.

**Diferenciador:** ensina shippar `.d.ts` types via JSDoc usando `tsc --emitDeclarationOnly --allowJs` (sem migrar source para TypeScript). Cobre primitivos modernos pouco endereçados em material publicado:

- `AbortSignal.any` / `AbortSignal.timeout`
- `Symbol.dispose` / `Symbol.asyncDispose` / `using` / `await using` (ES2024)
- `AsyncDisposableStack`
- `structuredClone` com transferable objects
- ES2025 Set operations (`union`, `intersection`, `difference`, `symmetricDifference`)
- `npm publish --provenance` (supply chain attestation)
- Subresource Integrity (SRI) hashes para CDN distribution
- Trusted Types para CSP-strict environments

**Conteúdo:**

- **14 references operacionais** (~5.000 linhas)
- **13 anti-patterns** no SKILL.md (sometimes-sync/async, default-export-everything, polyfill em top-level, mutating consumer options com scope qualifier, AbortSignal multi-layer trace, inconsistent async surface, etc.)
- **Hard Skip Gate** no topo do SKILL.md com 6 tells programáticos — refusa carregar quando 2+ matchearem (ex: imports `express`/`fastify`, sem bundler config, runtime PM2/node direto)
- **Decision trees:** factory vs class, sync vs async, throw vs Result, singleton vs multi-instance, inheritance vs composition
- **Cobertura cross-runtime:** browsers, Web Workers, Service Workers, Cloudflare Workers, Deno, Bun, Node 18+ via Web Standards primitives
- **Testing rigoroso:** contract tests, plugin integration tests, memory leak detection (WeakRef + GC), bundle size regression, tree-shake validation programática
- **Security baseline:** CSP-safe code, prototype pollution defense, DOM injection (Trusted Types), postMessage origin allowlist, supply chain hardening
- **Release engineering:** changesets workflow, dist-tags (`beta`/`next`/`canary`), 3-step deprecation cadence, Strangler Fig para migrations, rollback strategy

**Linguagem:** JavaScript ES2022+. Source vanilla; types gerados via JSDoc + TypeScript compiler.

---

## Metodologia

O que separa estas skills de coleções genéricas de prompts:

### 1. Tells mecânicos sobre intuição

Princípios viram regras com gatilho verificável. O modelo aplica regras; não pondera interpretações.

| Princípio abstrato | Regra operacional |
|--------------------|-------------------|
| Single Responsibility | Classe > 50 linhas, > 2 instance variables, ou métodos com 2+ verb domains → split |
| Long Method | Body > 10 linhas é suspeito; > 20 deve ser extraído |
| Primitive Obsession | Parâmetro `string` ou `number` para id/email/money/date/status → wrap em value object |
| Cyclomatic Complexity | ≤ 10 OK; > 10 refatorar |
| Cognitive Complexity | ≤ 15 OK; > 15 refatorar |
| Nesting Depth | ≤ 2 OK; > 2 extrair |
| Law of Demeter | Mais de 1 dot por chain (`a.b.c.d`) → Hide Delegate |

### 2. Decision trees com inputs concretos

Decisões ambíguas (factory vs class? mock vs fake? refactor agora vs depois?) viram árvores com inputs verificáveis. O modelo executa a árvore — não pesa opções.

```
Q: How to signal failure?
├── Programmer error (bad args, misuse, contract violation)
│   → throw new SDKError(msg, "INVALID_*")
├── Expected runtime failure (network, validation routinely handled)
│   → return Result type ({ ok, value?, error? })
├── Operation may legitimately have no result
│   → return null (NOT undefined, NOT sentinel)
└── Async operation failure
    → reject Promise with typed error
```

### 3. Anti-pattern galleries com `❌/✅`

Padrões que o modelo tende a gerar a partir do training data são listados explicitamente como erros, com a versão correta lado a lado. Ataque direcionado ao training-data drift.

### 4. Refactor recipes Fowler-style

Cada refactor tem **steps numerados**, não descrição abstrata. *"Extract Method"* não é "extraia uma função" — é uma sequência de 7 passos:

```
1. Identify the cohesive block to extract
2. Create a new method with intention-revealing name
3. Identify variables used by the block
4. Copy the block into the new method
5. Replace the original with a call to the new method
6. Run tests. If green, commit.
7. Repeat for next cohesive block.
```

### 5. When NOT to apply

A reference mais importante de todo o projeto é [`solid/references/when-not-to-apply.md`](./solid/references/when-not-to-apply.md). Define discipline levels:

| Lifetime | Discipline | What you do | What you skip |
|----------|-----------|-------------|---------------|
| Hours (REPL, ad-hoc) | Level 0 | Make it work | Tests, abstractions, polish |
| Weeks (one-off scripts) | Level 1 | Clear names, error handling | Unit tests, design patterns |
| Months (MVP) | Level 2 | Happy path tests, basic structure | Full coverage, premature abstraction |
| Years (production) | Level 3 | Full SOLID + TDD + everything | Nothing |

Disciplina aplicada no contexto errado é tão prejudicial quanto ausência de disciplina. Promote/demote triggers definem quando subir ou descer de nível.

### 6. Self-applicable review checklist

Antes de declarar pronto, o modelo executa um checklist por categoria (testes, naming, structure, design, complexity). Verificação explícita — não "achei que tava bom".

---

## Como instalar

Este repositório é um **plugin marketplace** do Claude Code. Recomendado: instalar via comando `/plugin`. Para casos sem Claude Code (CI, scripts, edição manual), há opções de clone também.

### Opção 1 — Plugin marketplace (recomendado)

Dentro do Claude Code:

```
/plugin marketplace add tarcisioq/claude-skills
/plugin install solid@tarcisio-skills
/plugin install vanilla-js-architect@tarcisio-skills
```

O `@tarcisio-skills` é o nome do marketplace (declarado em `.claude-plugin/marketplace.json`). Você só precisa adicionar o marketplace uma vez — depois pode listar e instalar skills com:

```
/plugin marketplace list
/plugin list
```

Para atualizar quando uma skill ganhar nova versão:

```
/plugin marketplace update tarcisio-skills
/plugin update solid@tarcisio-skills
```

> **Importante:** após uma atualização upstream, `/plugin marketplace update tarcisio-skills` é obrigatório antes de qualquer `/plugin install` ou `/plugin update` — caso contrário, o cliente reusa o clone local antigo e a nova versão nunca chega ao runtime.

### Opção 2 — `degit` (cópia simples sem Claude Code)

A estrutura publicada segue o spec de plugin do Claude Code, então o `SKILL.md` real fica em `<skill>/skills/<skill>/`. Aponte o `degit` para a pasta interna:

```bash
npx degit tarcisioq/claude-skills/solid/skills/solid ~/.claude/skills/solid
npx degit tarcisioq/claude-skills/vanilla-js-architect/skills/vanilla-js-architect ~/.claude/skills/vanilla-js-architect
```

Útil em automação (CI, dotfiles managers) ou quando você prefere copiar sem git history. Re-rode para atualizar.

### Opção 3 — Clone completo (contribuidores)

```bash
git clone https://github.com/tarcisioq/claude-skills.git
cp -r claude-skills/solid/skills/solid ~/.claude/skills/solid
cp -r claude-skills/vanilla-js-architect/skills/vanilla-js-architect ~/.claude/skills/vanilla-js-architect
```

Útil se você quer todas as skills e pretende contribuir de volta via PR.

### Opção 4 — Sparse checkout (mantém `git pull`)

```bash
mkdir ~/.claude/skills/solid && cd ~/.claude/skills/solid
git init
git remote add origin https://github.com/tarcisioq/claude-skills.git
git config core.sparseCheckout true
echo "solid/skills/solid/*" > .git/info/sparse-checkout
git pull origin master
# move o conteúdo da subpasta para a raiz da skill
mv solid/skills/solid/* . && rm -rf solid
```

Útil se você quer atualizar via `git pull` sem clonar o repo inteiro.

### Verificar instalação

Via Claude Code (skill instalada como plugin):
```
/plugin list
/skills
```

Manualmente (skill copiada para `~/.claude/skills/`):
```bash
ls ~/.claude/skills/
# → solid  vanilla-js-architect
```

Cada subpasta em `~/.claude/skills/<name>/` deve conter `SKILL.md` + `references/` na raiz. (Plugins instalados via `/plugin` ficam em `~/.claude/plugins/cache/` com a estrutura `skills/<name>/SKILL.md` interna — não tente copiar de lá.)

### Forçar always-on em um projeto

Adicione ao `CLAUDE.md` do projeto onde você quer rigor permanente:

```markdown
## Project conventions

Always apply the `solid` skill when working in this repository.
```

Útil em projetos de produção onde a disciplina não é opcional.

### Quando NÃO instalar

Cada skill tem seção *"When NOT to Use"* no SKILL.md. Skills disparam por descrição — uma skill mal aplicada é ruído. Para projetos que não casam com o escopo declarado, deixe de fora.

---

## Como usar

### Disparo automático (default)

Skills disparam quando o Claude detecta match com a `description` no frontmatter. Você não precisa invocar explicitamente:

```
> implementa um sistema de descontos com regras complexas
[solid dispara — task de design OO em código de produção]

> desenha um SDK de tracking de eventos pra browser
[vanilla-js-architect dispara — task de SDK browser-side]
```

### Disparo explícito

Para forçar uma skill específica:

```
> @solid revise este código
> @vanilla-js-architect desenhe a API pública
```

### Combinando skills

Skills compõem sem duplicar. Construindo um SDK de analytics em vanilla JS com TDD, ambas disparam:

```
> implementa um SDK de analytics em vanilla JS com TDD

# Skills invocadas:
# - vanilla-js-architect → arquitetura SDK, API design, distribution
# - solid                → TDD discipline, value objects, review checklist
```

A `solid` cobre **paradigm-level rules** (universais OO); `vanilla-js-architect` cobre **domain-specific rules** (SDK browser-side). Sem overlap.

---

## Economia de tokens

Custo real por tipo de tarefa, em tokens:

| Cenário | Sem skill | Com skill | Resultado |
|---------|-----------|-----------|-----------|
| Tarefa não relacionada (skill não dispara) | 0 | ~80 (description) | Empate |
| Tarefa simples (1-2 turnos, sem padrão complexo) | ~3.000 | ~12.000 | Sem skill é mais econômico — skill é overkill |
| Tarefa média (escolher pattern, design API) | ~13.000 (1-2 correções) | ~12.000 (acerto direto) | Empate em tokens; **skill ganha em qualidade** |
| Tarefa complexa (SDK completo, plugin system, refactor de domínio) | ~30.000+ (3-5 correções) | ~18.000 | **Skill economiza ~40%** |

O ganho real **não é token economy** — é **consistência de qualidade**. Skills resolvem oscilação de output, não custo. Em código de produção, a economia em rounds de correção e re-prompt supera o custo upfront.

Skills são **lazy by design**: a `description` ocupa ~80 tokens enquanto não invocada. SKILL.md (~5.500 tokens) só carrega quando a skill dispara. References carregam sob demanda quando o SKILL.md instrui.

---

## Decisões de engenharia

Algumas escolhas deliberadas que moldam estas skills:

**TypeScript como linguagem-exemplo da `solid`** — comunidade dominante hoje; tipos facilitam ilustrar value objects e contratos; sintaxe próxima de Java/C# facilita tradução mental. Regras (size limits, smells, refactor recipes) são universais e portam para qualquer OO language.

**Uma skill `solid` em vez de três separadas** (`clean-code-discipline`, `tdd-workflow`, `solid-principles`) — todas se reforçam (TDD revela violações de SOLID; clean code é o que SOLID produz). Skills separadas duplicariam exemplos e fragmentariam decisões. `references/when-not-to-apply.md` modulariza disciplina dentro da própria skill.

**ESM-only como default em `vanilla-js-architect`** — bundlers modernos (Vite, esbuild, Rollup, webpack 5+) e Node 18+ tratam ESM nativamente. Dual ESM/CJS introduz dual-package hazard. Posição explícita > "suporte universal".

**JSDoc + `tsc --emitDeclarationOnly --allowJs` em vez de migrar source para TypeScript** — incremental, source executa sem build em dev, audiences no-build / `<script>` ainda atendidas, types disponíveis para consumers TS. O melhor de dois mundos sem o custo da migração.

**Decision trees explícitos em vez de "use bom senso"** — bom senso oscila entre sessões. Decision tree com inputs concretos é reproduzível.

**Skills versionadas com semver, changelog discipline, deprecation cadence** — embora não sejam pacotes npm hoje, o processo (changesets, dist-tags `beta`/`next`, 3-step deprecation, rollback strategy via `npm deprecate`) é o mesmo de qualquer artefato shippado.

---

## License & contribuição

Todas as skills sob licença **MIT** (declarada no frontmatter de cada `SKILL.md`).

- **Bug reports / sugestões** → [issue no repositório](https://github.com/tarcisioq/claude-skills/issues), com label da skill afetada (`solid`, `vanilla-js-architect`)
- **PRs** → bem-vindos para fixes em smells, tells imprecisos, exemplos quebrados. Um PR por skill afetada (não misture mudanças em skills diferentes)
- **Novos patterns / references** → discuta em issue antes do PR (escopo importa — cada skill tem `description` calibrada e expansão precisa ser deliberada)
- **Skills novas** → abra issue de proposta primeiro (escopo, problema que resolve, when-not-to-apply); se aprovado, fork + PR com skill nova em subpasta + entrada em `marketplace.json`

Cada skill é versionada independentemente (semver no frontmatter `metadata.version`); um PR pode bumpar uma skill sem tocar nas outras.

---

## Sobre

Mantido por **Tarcísio Quaresma** ([@tarcisioq](https://github.com/tarcisioq)).

Skills são desenvolvidas como ferramenta pessoal para acelerar trabalho com Claude Code em projetos de produção; publicadas como portfolio + contribuição open-source.

Conexões profissionais e oportunidades: GitHub [@tarcisioq](https://github.com/tarcisioq).
