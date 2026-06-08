# Auditoria — Fase 2 (aplicação das decisões aprovadas)

**Arquivo:** `index.html` · **Continuação de:** `AUDITORIA.md` (fase 1)
**Escopo:** aplicar **apenas** as 6 decisões aprovadas pelo Everton. Nenhuma re-auditoria; nada fora da lista.
**Resultado:** as 6 aplicadas e validadas. **Zero erro de JS. Nenhuma regressão.**

**md5 final do `index.html`:** `436561ace754d6b46d4424c9b8994b2b`
**Diff fase 2:** `1 file changed, 26 insertions(+), 17 deletions(-)`

---

## 1. O que mudou em cada item

### Item 1 — Pés em "X" (corrige X vs V)
- **Local:** seção evo, `.evo-cap` (linha ~925).
- **Antes:** "…acabamento escovado uniforme, **pernas em V** — feito para o litoral."
- **Depois:** "…acabamento escovado uniforme, **pernas em X** — feito para o litoral."
- **Verificação:** o hero (linha ~652) já estava com "X". Busca por `pernas em V`/`pés em V` → **0 ocorrências**. Restam só as duas menções, ambas "X".

### Item 2 — Foco visível on-brand no CTA da chama
- **Local:** CSS do `.flame-btn` (linha ~464).
- **Antes:** `.flame-btn:focus,.flame-btn:focus-visible,.flame-btn:active{outline:0!important;box-shadow:none!important;border:none!important}` (removia o indicador de foco — regressão de a11y).
- **Depois:**
  ```css
  .flame-btn:focus-visible{outline:none}
  .flame-btn:focus-visible .flame-label{text-decoration:underline;text-underline-offset:4px;color:var(--e3)}
  .flame-btn:focus-visible .flame-outer{filter:drop-shadow(0 0 10px var(--eg))}
  ```
- Seletores dos filhos confirmados nos nomes reais (`.flame-label`, `.flame-outer`). Mantém o foco invisível ao mouse (só `:focus-visible`) e claramente visível ao teclado.

### Item 3 — Corrida no swap do vídeo mobile (P1-4)
- **Local:** `initHeroVideo()` + IIFE "VIDEO MOBILE SWAP".
- **O que era:** a IIFE rodava na análise do script e chamava `v.load()` **antes** de `initHeroVideo` (no `DOMContentLoaded`) anexar os listeners → em cache/conexão rápida, `loadedmetadata`/`loadeddata` podiam disparar antes dos listeners → `vdur=0`.
- **O que ficou:** o swap mobile + `heroVid.load()` foi movido para **dentro de `initHeroVideo`, logo após anexar os dois listeners**. A IIFE foi removida (substituída por comentário-referência). Ordem garantida: *listeners → swap src → load()*.
- **Nada mais da lógica de vídeo/scrubbing foi tocado** — só a ordem de anexar listeners vs. `load()`.

### Item 4 — Unidades iOS (`vh` → `svh`)
- **Local/Antes → Depois:**
  - `.pt{… bottom:12vh …}` → `bottom:12svh`
  - `.pt.center{… bottom:14vh …}` → `bottom:14svh`
  - `#scue{ bottom:5.5vh …}` → `bottom:5.5svh`
- **Apenas essas três regras.** Nenhuma outra unidade do arquivo foi alterada (a override mobile `@media(max-width:760px){.pt{bottom:13vh}…}` foi deixada intacta, conforme instrução).

### Item 5 — Fallback sem JavaScript (`<noscript>`)
- **Local:** novo bloco `<noscript><style>` no `<head>`, logo antes de `</head>`.
- **Conteúdo:**
  ```html
  <noscript><style>
    #intro,#darkrise{display:none!important}
    #track{display:none!important}
    html,body{overflow:auto!important}
    .rv{opacity:1!important;transform:none!important;filter:none!important}
    .n2 h2.rv .n2hl{transform:none!important}
  </style></noscript>
  ```
- **Efeito:** sem JS, o `#intro` (e o `#darkrise`) somem em vez de prender a página; o scroll é liberado; todo o conteúdo `.rv` da proposta fica visível. O hero scrollytelling (`#track`, que depende de JS) é ocultado para não virar um vão vazio de 3000vh. Corporativo com JS travado lê a proposta inteira.

### Item 6 — Contraste do token `--faint` (#65686f → #7d808a)
- **Local:** as **3 definições** do token (`:root`, `.evo-scope`, `.cw-scope`) **e** 1 cópia inline idêntica (`<p style="color:#65686f">` na nota do comp-map, linha ~703) — todas para `#7d808a`, para não deixar um texto solto mais escuro que o resto.
- **Contraste:** `#7d808a` sobre `#090909` ≈ **5,0:1** (acima do AA 4,5:1 para texto normal; antes ~3,6:1). Continua na família cinza/aço e **mais escuro** que `--dim`(#9a9da5)/`--s3`(#8a8d95) → hierarquia preservada (confirmado visualmente no screenshot do rodapé/pricing — sem achatamento).
- **Busca:** `#65686f` restante → **0**; `#7d808a` → 4 ocorrências.

---

## 2. Diff resumido (linhas alteradas)

```diff
- --faint:#65686f;   (×3 definições + 1 inline)
+ --faint:#7d808a;
- .pt{… bottom:12vh …}            → + bottom:12svh
- .pt.center{… bottom:14vh …}     → + bottom:14svh
- #scue{ bottom:5.5vh …}          → + bottom:5.5svh
- .flame-btn:focus,…:active{outline:0!important;…}
+ .flame-btn:focus-visible{outline:none}
+ .flame-btn:focus-visible .flame-label{text-decoration:underline;text-underline-offset:4px;color:var(--e3)}
+ .flame-btn:focus-visible .flame-outer{filter:drop-shadow(0 0 10px var(--eg))}
+ <noscript><style> … #intro,#darkrise,#track display:none; .rv opacity:1 … </style></noscript>
- pernas em V  → + pernas em X
+ (swap mobile movido para dentro de initHeroVideo, após os listeners)
- (IIFE "VIDEO MOBILE SWAP" removida)
```

---

## 3. Validações

### Integridade (estática)
| Checagem | Resultado |
|---|---|
| `<div>` / `</div>` | 344 / 344 ✓ |
| `<section>` / `</section>` | 14 / 14 ✓ |
| `<script>` / `</script>` | 3 / 3 ✓ |
| `<style>` / `</style>` | 4 pares reais ✓ (a contagem bruta dá 5/4: o 5º `<style>` é **texto literal dentro de um comentário HTML**, e há 1 novo par real no `<noscript>`) |
| `{` / `}` (CSS+JS) | 833 / 833 ✓ |
| `(` / `)` (CSS+JS) | 1632 / 1632 ✓ |
| `alt` em imagens | 100% ✓ |
| Variáveis CSS usadas-sem-definição | **0** ✓ |
| `pernas em V` / `pés em V` | **0** ✓ |
| `#65686f` restante | **0** ✓ |

### Playwright/Chromium headless — 1440×900, 768×1024, 390×844 + passe `prefers-reduced-motion`
- **Erros de JS / `pageerror`: 0** em todos os viewports (único log é a CDN de fontes bloqueada pelo sandbox — artefato de rede, não do código).
- **Reveals da fase 1 continuam corretos (sem regressão):** `.big` (manifesto), `.pos-shot` e `.evo-compare` = **opacity 1** em todos os viewports e em reduced-motion.
- **Varredura completa de `.rv` (após settle de 3,8 s p/ transições + delays terminarem):**
  - desktop 1440: apenas **2** abaixo de 1 → as duas cartas laterais do showroom em `0.82` (**efeito de profundidade intencional**, já documentado na fase 1).
  - mobile 390: **0** abaixo de 1.
  - reduced-motion: **0** abaixo de 1.
  - *(Obs.: uma medição feita cedo demais após scroll rápido mostra muitos valores intermediários — são transições em andamento, não bugs; comprovado pelo passe reduced-motion com 0 abaixo de 1.)*
- **Item 2 — foco via teclado (real):** com foco de teclado no `#flameBtn`, `:focus-visible` casa (`matchesFocusVisible: true`), `activeElement` é o botão, e o `.flame-label` recebe `text-decoration: underline` + cor/brilho ember. Indicador de foco **visível e on-brand** confirmado.
- **Item 5 — passe com JavaScript desativado (`javaScriptEnabled:false`):**
  `#intro` → `display:none`, `#track` → `display:none`, `.big.rv` → `opacity:1`, `html` overflow → `auto`. **Conteúdo liberado, sem prender na intro.** ✓
- **Item 3:** sem erro de JS após a reordenação; o swap mobile permanece (define `hero-wan-mobile.mp4` e chama `load()` após os listeners). *Obs.: o Chromium headless não tem codec H.264, então o valor de `vdur` em si não é verificável aqui — mas a correção é de ordem de execução, estruturalmente confirmada no diff.*

---

## 4. Confirmação de integridade pós-edições

- Markup, CSS e JS balanceados (tabela acima).
- 0 erro de console/JS introduzido.
- 0 variável CSS indefinida.
- Backup/rollback: histórico Git (commit anterior à fase 2 contém o `index.html` da fase 1).

---

## 5. Git — comandos prontos para o Everton publicar

A fase 2 foi **commitada na branch `claude/gracious-hypatia-99li51`** e a **`main` foi atualizada por fast-forward** (sem divergência), pronta para publicar. **Nenhum `push` foi feito** — você revisa e publica.

Para publicar (deploy via GitHub Pages na `main`):

```bash
# já está tudo na main local por fast-forward; basta publicar:
git checkout main
git push origin main
```

Se preferir revisar antes na branch e mergear você mesmo:

```bash
git checkout main
git merge --ff-only claude/gracious-hypatia-99li51   # fast-forward, sem merge commit
git push origin main
```

Conferência rápida antes de publicar:
```bash
git log --oneline -3
md5sum index.html        # esperado: 436561ace754d6b46d4424c9b8994b2b
```

> Se quiser manter o rastro da branch de trabalho no remoto também: `git push origin claude/gracious-hypatia-99li51`.

---

*Fase 2 concluída de forma autônoma, restrita às 6 decisões aprovadas. Não foram tocados: dados de mercado, asset do blueprint (D-1), focus-trap do modal (P2-3), código morto (P2-7), preload do vídeo (P1-3), layout thrashing (P2-8), nem paleta/fontes/marca/comercial/contatos/specs.*
