# Auditoria Final — Proposta Vision Inox × Everton Stedile

**Arquivo auditado:** `index.html` (single-file, ~135 KB) · **Assets:** `assets/`
**Data:** 2026-06-08 · **Backup criado antes de editar:** `index.html` original preservado no histórico Git (commit anterior a esta auditoria).
**Validação:** Playwright/Chromium headless em 1440×900, 768×1024 e 390×844 + passe com `prefers-reduced-motion`.

---

## 1. Resumo executivo

A peça está **muito bem construída** — direção de arte coerente, copy sênior, arquitetura de scrollytelling sofisticada e um sistema de reveal cuidadoso. Mas a auditoria encontrou **um defeito crítico de renderização cross-browser que estava mascarado no Chrome** e teria deixado blocos-chave **invisíveis em Safari/iOS/Firefox** — exatamente o cenário que o briefing temia.

A causa raiz era dupla e se reforçava:

1. O guard `@supports(animation-timeline:--disabled-xyz())` — escrito para **desligar** a camada de reveal scroll-driven — é avaliado como **VERDADEIRO** no Chrome moderno (recurso de *dashed-function*). Isso reativava a animação CSS e mascarava o item nº 2.
2. A regra `.rv[data-rv="mask"].in` **não reassentava `opacity:1`** e perdia em ordem de cascata para `.rv[data-rv="mask"]{opacity:0}`. Resultado: a headline do **manifesto** ("Pegar uma indústria de inox… margem de fabricante"), a **foto lifestyle do posicionamento** (`.pos-shot`) e o comparador antes/depois ficavam em `opacity:0` em qualquer navegador sem suporte a `animation-timeline` — ou seja, **Safari desktop, iOS e Firefox antigos não viam esses blocos**.

Ambos foram **corrigidos e validados**: unifiquei o reveal num único mecanismo (IntersectionObserver), determinístico e idêntico em todos os navegadores, e adicionei uma rede de segurança. Após as correções, **todos os 97 elementos `.rv` revelam em opacity 1** nos três viewports e com `prefers-reduced-motion`.

Também corrigi uma **imagem quebrada** no hero (asset desktop inexistente), adicionei **meta tags sociais/SEO**, **acessibilidade de movimento reduzido**, **lazy-loading** e fechamento do modal por **Esc**.

### Nota de prontidão para envio: **8,5 / 10**

**Pode ser enviada.** O que separa do 10 são itens de decisão sua (não bloqueantes): regenerar o asset desktop do blueprint à mão (hoje cai num substituto em retrato que recorta no desktop), confirmar **"pernas em X" vs "pernas em V"**, confirmar a URL de deploy para as tags OG, e otimizar ~11 MB de imagens de referência. Nenhum bloqueia o envio; todos deixam a peça impecável.

---

## 2. Achados por severidade

### 🔴 P0 — Bloqueia/compromete o envio

#### P0-1 — Reveal scroll-driven duplicado e inconsistente entre navegadores *(CORRIGIDO)*
- **Local:** `@supports(animation-timeline:--disabled-xyz())` (bloco "MOTION SCROLL-DRIVEN", ~linha 311 do CSS).
- **Impacto:** No Chrome moderno o `@supports` é verdadeiro → a camada CSS `animation-timeline:view()` ficava **ativa simultaneamente** ao reveal JS (`.in`), gerando reveal duplo e dependente de feature recente/parcial. Em Safari/iOS o `@supports` é falso → comportamento divergente.
- **Recomendação aplicada:** Desliguei a camada CSS de forma determinística (`A and (not A)` = sempre falso, em qualquer navegador). O reveal passou a ser **100% via IntersectionObserver** — mesma língua que o Safari já usava. Mecanismo único, previsível, sem ambiguidade.

#### P0-2 — Blocos `[data-rv="mask"]` invisíveis fora do Chrome *(CORRIGIDO)*
- **Local:** `.rv[data-rv="mask"].in` (CSS, ~linha 305). Afeta `.big` (manifesto), `.pos-shot` (posicionamento) e `.evo-compare`.
- **Diagnóstico:** A regra de "revelado" (`.in`) zerava `transform/filter/clip-path` mas **esquecia `opacity`**. Como `.rv[data-rv="mask"]{opacity:0}` tem a **mesma especificidade** de `.rv.in{opacity:1}` e vem **depois** na cascata, vencia — e o elemento ficava `opacity:0` mesmo "revelado". No Chrome a animação do P0-1 mascarava; em Safari/iOS/Firefox **não havia nada revelando** esses blocos.
- **Correção aplicada:**
  ```css
  .rv[data-rv="mask"].in{transform:none;filter:none;clip-path:none;opacity:1}
  ```
- **Prova:** screenshots `proof_manifesto-1440` e `proof_posshot-1440` mostram os blocos visíveis; check programático: 0 elementos em opacity<1 após a correção.

#### P0-3 — Imagem quebrada no hero (asset desktop ausente) *(CORRIGIDO)*
- **Local:** cena `#kf-draw` (linha ~618). `<img src="assets/blueprint_desenho_final.png">` apontava para um arquivo **que não existe** — só existe `blueprint_desenho_final-mobile.png`.
- **Impacto:** No **desktop**, a cena "blueprint desenhada à mão" do scrollytelling carregava uma **imagem quebrada** (404). No mobile funcionava (o `<source>` usa o `-mobile`).
- **Correção aplicada (paliativa):** apontei o `<img src>` para `blueprint_desenho_final-mobile.png` (que existe) → nada mais 404. **Atenção:** esse arquivo é **retrato 1024×1536**; no palco landscape do hero ele **recorta** (mostra a faixa central). Funciona, mas o ideal é regenerar a versão desktop landscape — ver item de decisão D-1.

---

### 🟡 P1 — Importante (peso, robustez, conversão)

#### P1-1 — Reveal principal sem failsafe nem fallback sem-IO *(CORRIGIDO)*
- **Local:** observer principal `const io = new IntersectionObserver(...)` no JS final.
- **Diagnóstico:** as seções `.evo` e `.cw` têm fallback (`if(!('IntersectionObserver' in window))`) **e** um `setTimeout(4500)` de segurança — "padrão já validado no projeto". O observer **principal não tinha nenhum dos dois**. Ele só não falhava no Chrome porque a camada P0-1 o socorria.
- **Correção aplicada:** adicionei uma rede de segurança aditiva (só revela, nunca esconde), no mesmo padrão das outras seções:
  - fallback sem IntersectionObserver → revela tudo;
  - sweep aos 4500 ms para o que estiver perto do viewport;
  - sweep ao parar o scroll para elementos já ultrapassados (flick/âncora/scroll muito rápido).
  - O fallback também finaliza `data-count` e `data-w` (contadores e barras) para o conteúdo nunca ficar "zerado".

#### P1-2 — Peso de imagens de referência: ~11 MB em 6 PNGs *(REPORTAR — sem ferramenta de imagem no ambiente)*
- **Local:** `assets/ref_1.png … ref_6.png` (1,3–2,2 MB cada, **PNG RGB**), usadas na seção "Evolução do produto".
- **Impacto:** são renders fotográficos salvos como PNG sem transparência — formato errado. Em JPEG/WebP encolhem **80–90%** sem perda perceptível. Mitiguei o impacto no carregamento inicial com `loading="lazy"` (ver Corrigido), mas o peso permanece quando o usuário chega na seção.
- **Recomendação:** transcodificar para WebP (ou JPEG q82). Comandos prontos:
  ```bash
  # WebP (melhor): precisa de cwebp
  for f in assets/ref_*.png; do cwebp -q 82 "$f" -o "${f%.png}.webp"; done
  # ou JPEG com ImageMagick:
  for f in assets/ref_*.png; do magick "$f" -quality 82 "${f%.png}.jpg"; done
  # no seu Mac, com sips (mantém qualidade):
  for f in assets/ref_*.png; do sips -s format jpeg -s formatOptions 82 "$f" --out "${f%.png}.jpg"; done
  ```
  Depois atualizar os `src` correspondentes. *(Não executei: este ambiente Linux não tem `sips`/ImageMagick/cwebp/ffmpeg instalados.)*

#### P1-3 — Vídeo do hero: 8,5 MB com `preload="auto"` *(REPORTAR — não toquei, é lógica do hero)*
- **Local:** `#heroVid` (linha ~627), `preload="auto"`, `hero-wan-web.mp4` (8,5 MB) / `hero-wan-mobile.mp4` (8,0 MB).
- **Impacto:** `preload="auto"` pode puxar os ~8 MB **no load**, competindo com fontes/poster e atrasando o primeiro frame. O vídeo só é usado em ~50% do scroll do hero.
- **Recomendação (sua decisão — mexe no carregamento do hero):** considerar `preload="metadata"` e disparar o carregamento no clique do intro (`heroVid.load()` dentro de `enterExperience`). Trade-off: precisa garantir que esteja pronto quando o scrubbing chegar na cena praia, senão trava. Como o scrubbing por `currentTime` depende do vídeo bufferizado, **mantive como está** — é uma troca consciente.

#### P1-4 — Corrida no swap do vídeo mobile *(REPORTAR — lógica do hero)*
- **Local:** IIFE "VIDEO MOBILE SWAP" (linha ~1820) roda na análise do script e chama `v.load()`; os listeners `loadedmetadata`/`loadeddata` só são anexados no `DOMContentLoaded` (depois).
- **Impacto (raro):** em conexão rápida/cache, o `loadedmetadata` pode disparar **antes** dos listeners → `vdur` fica 0 → o scrubbing do vídeo na cena praia não roda (mostra poster/primeiro frame). Edge case, mas real.
- **Recomendação:** mover o swap para **dentro** de `initHeroVideo`, **antes** de anexar os listeners; ou anexar listeners antes do `load()`. Não alterei por tocar a inicialização do vídeo do hero.

---

### 🟢 P2 — Polish (acessibilidade fina, consistência, perf marginal)

#### P2-1 — Contraste de `--faint` (#65686f) abaixo de AA *(REPORTAR)*
- **Cálculo:** `#65686f` sobre `#090909` ≈ **3,6:1**. Passa em texto grande (≥24px) e AA Large, **falha** em texto normal (precisa 4,5:1). Usado **19×**, inclusive no rodapé (12px) e em legendas pequenas.
- **Recomendação:** subir para ~`#7d808a` (≈4,6:1) — continua dentro da família cinza/aço da paleta — **ou** restringir `--faint` a textos decorativos/grandes e usar `--s3`/`--dim` nos textos pequenos. Não alterei por ser token global (afeta a hierarquia visual em muitos pontos) — sua decisão estética.

#### P2-2 — Outline de foco removido no botão "Revelar a proposta" *(REPORTAR)*
- **Local:** `.flame-btn:focus,.flame-btn:focus-visible,.flame-btn:active{outline:0!important;…}` (linha ~464).
- **Impacto:** usuário de teclado **não vê o foco** no CTA que abre o modal de investimento. Regressão de acessibilidade feita por estética.
- **Recomendação:** trocar a remoção total por um foco visível discreto e on-brand, sem a "caixa feia":
  ```css
  .flame-btn:focus-visible{outline:none}
  .flame-btn:focus-visible .flame-label{text-decoration:underline;text-underline-offset:4px;color:var(--e3)}
  .flame-btn:focus-visible .flame-outer{filter:drop-shadow(0 0 10px var(--eg))}
  ```

#### P2-3 — Modal sem *focus trap* / sem foco inicial *(parcialmente mitigado)*
- **Adicionei** fechar por **Esc** (ver Corrigido). Falta, para acessibilidade plena: mover o foco para dentro ao abrir, prender o Tab no modal e devolver o foco ao `#flameBtn` ao fechar. É mais invasivo — deixei para sua decisão. Esqueleto:
  ```js
  // ao abrir: const f = modal.querySelectorAll('button,a,[tabindex]'); f[0]?.focus();
  // keydown Tab: ciclar entre f[0] e f[last]; ao fechar: flameBtn.focus();
  ```

#### P2-4 — Sem `<noscript>` *(REPORTAR)*
- Como o `#intro` trava o scroll por JS e só libera no clique, **sem JavaScript o usuário vê apenas a tela de intro** e não passa. Recomendo um fallback CSS/`<noscript>` que esconda o `#intro` e libere o conteúdo quando JS estiver desativado (público B2B com navegadores corporativos restritivos pode cair nisso).

#### P2-5 — Inconsistência de copy: "pernas em X" vs "pernas em V" *(DECISÃO — ver D-2)*
- Linha 640 (hero): "pernas em **X**". Linha 913 (evo): "pernas em **V**". É detalhe de produto — você decide qual é o correto e unifico.

#### P2-6 — `vh` vs `svh/dvh` em sobreposições do hero *(REPORTAR, menor)*
- `#stage` já usa `100svh` (correto p/ iOS). Mas `.pt{bottom:12vh}`, `.pt.center{bottom:14vh}` e `#scue{bottom:5.5vh}` usam `vh` puro → podem "pular" levemente com a barra de endereço do iOS. Trocar por `svh`/`dvh` deixa cravado. Cosmético.

#### P2-7 — Código morto *(REPORTAR, opcional)*
- O primeiro `.flame-btn{…}`, `.flame-icon` e `@keyframes flkr` (linhas ~423–430) foram substituídos pela redefinição "Onda 2" (`.flame-svg`). São regras sobrescritas/órfãs — removíveis. Deixei intactas para não arriscar a integridade do arquivo validado; limpeza opcional.

#### P2-8 — Read/write de layout intercalado em loops de scroll *(REPORTAR, perf marginal)*
- Os loops das barras scrubadas (`.stat-bar i,.crow-track i`) e do parallax `[data-px]` leem `getBoundingClientRect()` e escrevem `transform` **intercalados** no mesmo frame (potencial *layout thrashing*). Tudo já está em `rAF` + `{passive:true}`, então o custo é baixo e único por frame. Otimização possível: ler tudo primeiro, depois escrever. Não reescrevi (toca a lógica de scroll).

#### P2-9 — `praia_final.png` é PNG de 799 KB *(REPORTAR — o medo do "4K" era falso)*
- **Importante:** ao contrário do que o briefing supôs, `praia_final.png` **não é 4K** — é **1536×1024, 799 KB**. Não precisa downsample. Mas é foto salva em PNG; em JPEG cairia para ~150 KB. Existe `praia_final-mobile.png` (retrato, servido via `<picture>`). Otimização menor (já está com `loading="lazy"`).

---

## 3. Corrigido automaticamente

Todas as alterações são cirúrgicas, comentadas com `AUDITORIA:` no código e reversíveis. Integridade revalidada após cada edição.

| # | O que mudou | Onde | Por quê | Antes → Depois |
|---|---|---|---|---|
| 1 | **Neutralizei o `@supports`** scroll-driven (`A and (not A)`) | bloco MOTION SCROLL-DRIVEN | Eliminar reveal duplo/ambíguo; um mecanismo único | Camada CSS ativa só no Chrome → **desligada em todos** |
| 2 | **`opacity:1`** em `.rv[data-rv="mask"].in` | CSS reveal | Manifesto/pos-shot/evo-compare ficavam invisíveis fora do Chrome | `opacity:0` revelado → **visível em todos** |
| 3 | **Rede de segurança do reveal** (fallback sem-IO + 2 sweeps) | JS, após `io.observe` | Garantir reveal em qualquer navegador/velocidade | Sem failsafe → **failsafe igual ao de .evo/.cw** |
| 4 | **Corrigi imagem quebrada** `#kf-draw` (src → `-mobile`) | hero, linha ~618 | Asset desktop não existia (404) | 404 no desktop → **imagem carrega** |
| 5 | **Defini `--text-dim` e `--line-2`** no `:root` | `:root` | 11 usos de variáveis **inexistentes** | `var()` inválido (herdava cor errada/borda some) → **resolve** |
| 6 | **Meta tags** description, theme-color, color-scheme, **OG**, **Twitter**, robots, **favicon SVG inline** | `<head>` | SEO + preview de link (WhatsApp/redes) + sem 404 de favicon | Nenhuma → **conjunto completo** |
| 7 | **`prefers-reduced-motion`** (bloco CSS) | fim do `<style>` principal | Acessibilidade: neutraliza loops/transições/parallax; garante `.rv` visível | Só o JS de smooth-scroll respeitava → **CSS também respeita** |
| 8 | **`loading="lazy" decoding="async"`** em 10 imagens fora da 1ª tela | evo (ref_*, proto, blueprint_2), pos-shot, retrato | Não baixar ~11 MB no load (todas vivem **depois** de 3000vh de hero) | Eager → **lazy** |
| 9 | **`-webkit-backdrop-filter`** no `.p-modal` | CSS modal | Safari precisa do prefixo para o blur do fundo | Blur só no Chrome → **blur no Safari também** |
| 10 | **Fechar modal com `Esc`** | JS modal | Acessibilidade de teclado | Só ✕/clique no overlay → **+ Esc** |

> Variáveis indefinidas (#5) confirmadas zeradas após a correção: scan automático de `var(--x)` vs `--x:` → **0 variáveis usadas sem definição**.

---

## 4. Requer decisão do Everton

| Id | Item | Recomendação |
|---|---|---|
| **D-1** | **Asset desktop do blueprint à mão.** Hoje o `#kf-draw` no desktop usa o arquivo `-mobile` (retrato 1024×1536), que **recorta** no palco landscape. | Regenerar/exportar `assets/blueprint_desenho_final.png` em landscape (~1536×1024, alinhado aos outros frames). Eu já garanti que **nada quebra** enquanto isso. |
| **D-2** | **"Pernas em X" (hero, l.640) vs "pernas em V" (evo, l.913).** | Me diga a geometria correta e eu unifico nos dois pontos. |
| **D-3** | **URL de deploy para OG/Twitter.** Usei o padrão `https://evertonstedile.github.io/visioninox-proposta/` nas tags `og:url`/`og:image`. | Se o deploy for outro domínio (custom), me passe a URL e eu ajusto (preview de link só funciona com URL **absoluta** correta). |
| **D-4** | **Otimizar ~11 MB de `ref_*.png`** (P1-2). | Rodar os comandos do P1-2 no seu Mac (sips) e atualizar os `src`. Posso preparar o patch dos `src` assim que os arquivos existirem. |
| **D-5** | **Contraste de `--faint`** (P2-1, 3,6:1). | Subir para `#7d808a` (global) ou trocar usos pequenos por `--s3`. Decisão estética. |
| **D-6** | **Foco visível no CTA da chama** (P2-2) e **focus-trap do modal** (P2-3). | Aplicar o snippet de `:focus-visible` que deixei; trap é opcional. |
| **D-7** | **Specs técnicas** ("80mm de curso" l.947, inox 304, "304 vs 430"). | **Não alterei** (fora de escopo). Apenas sinalizo: validar com o diretor técnico antes do envio final. |
| **D-8** | **Dados de mercado** ("USD 6,36 bi" l.685, "62%" l.686, "7.400 km" l.694, "crescendo 5%/ano"). | Conferir fontes/ano — são números fortes e checáveis; se um cliente questionar, convém ter a fonte à mão. |
| **D-9** | **`preload` do vídeo** (P1-3) e **corrida do swap mobile** (P1-4). | Avaliar as recomendações; tocam a inicialização do hero (validada), por isso não mexi. |

> **Fora de escopo, intocado conforme instruções:** paleta, par de fontes, marca "Vision Inox"/logo, valores e estrutura comercial, telefone do WhatsApp (`5548991578012`) e e-mail (`evertonstedile@gmail.com`), specs do produto, e toda a lógica de scroll/áudio/zoom/timing do hero.

---

## 5. Resultado das validações (Playwright/Chromium headless)

Rodado em **1440×900**, **768×1024**, **390×844** e um passe extra com **`prefers-reduced-motion: reduce`**. Em cada um: dispensa do intro ("continuar sem som"), varredura por todo o documento, screenshots em 3 alturas + modal.

| Viewport | Erros console/pageerror | `.rv` total | `.rv` não revelados | Modal abre | Esc fecha | `#kf-draw` |
|---|---|---|---|---|---|---|
| desktop 1440 | 0¹ | 97 | **0**² | ✅ | ✅ | ✅ 1024×1536 |
| tablet 768 | 0¹ | 97 | **0** | ✅ | ✅ | ✅ |
| mobile 390 | 0¹ | 97 | **0** | ✅ | ✅ | ✅ |
| 1440 + reduced-motion | 0¹ | 97 | **0** | ✅ | ✅ | ✅ |

¹ O único log capturado foi `net::ERR_CERT_AUTHORITY_INVALID` ao buscar **Google Fonts** — artefato do sandbox **sem rede/cert**, não do código. **Zero `pageerror`/erro de JS** introduzido pelas edições.
² Antes da correção P0-2 havia 2 elementos em `opacity:0` (`.big` manifesto e `.pos-shot`). Após a correção: 0. Os únicos `.rv` abaixo de opacity 1 que restam são as **2 cartas laterais do showroom no desktop em `opacity:.82`** — isso é o **efeito de profundidade intencional** (`.line-grid.showroom …:.82`), confirmado: em tablet/mobile ficam em opacity 1.

**Imagens-chave decodificadas** (naturalWidth>0) em todos os viewports: `#kf-draw`, `.pos-shot`, `.pc-photo` (retrato), `.evo-side.after` (ref_5). **Reveal das imagens das seções (incl. a foto da praia): confirmado em opacity 1.**

**Evidências visuais geradas:** manifesto e pos-shot agora visíveis (antes invisíveis fora do Chrome); modal de investimento renderiza com a borda correta (efeito colateral positivo do `--line-2`); layout mobile do "dor × Vision" empilha certo com o sinal verde "Resolvido".

---

## 6. Confirmação de integridade

Revalidado **após** todas as edições:

- **Markup balanceado:** `<div>` 344/344 · `<section>` 14/14 · `<picture>` 15/15 · `<style>` 3 pares · `<script>` 3 pares. (`<source>` é elemento vazio — 18 aberturas, 0 fechamentos, correto.)
- **Chaves/parênteses (CSS+JS):** `{` 827 / `}` 827 · `(` 1628 / `)` 1628 — **balanceados**.
- **Variáveis CSS:** scan `var(--x)` × `--x:` → **0 indefinidas**.
- **Console:** **0 erros de JavaScript / 0 `pageerror`** introduzidos (único log é a CDN de fontes bloqueada pelo sandbox).
- **`alt`:** 100% das imagens têm `alt` (verificado antes e depois).
- **Diff:** 83 inserções / 17 deleções, todas rastreáveis e comentadas no código.

**O original é recuperável** a qualquer momento via `git diff`/histórico (o commit anterior a esta auditoria contém o `index.html` intacto).

---

*Auditoria conduzida de forma autônoma. As correções aplicadas são de baixo risco e inequívocas; tudo que envolvia julgamento de marca, specs, comercial ou a mecânica validada do hero foi apenas reportado para sua decisão.*
