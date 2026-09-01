# Frente 1 — Design e telas

Esta frente conduz o **`/design` do Claude** (comando, não skill) para construir as telas do
projeto seguindo o design system do OmniHub **à risca** — e depois verifica que o resultado
não divergiu. O input vem do PRD (seção **Telas e rotas**); o design system vem daqui.

> **Regra que não se quebra:** o `/design` nunca recebe descrição de cor ("azul da marca") —
> recebe **valor exato**. Descrição vira interpretação; valor vira tela. Todo prompt para o
> `/design` carrega o bloco de tokens abaixo, literal.

---

## O design system — fonte da verdade

### Tokens (copie este bloco, nunca reescreva de memória)

```css
/* OmniHub — tokens de tema */
:root, [data-theme="light"] {
  --acc: #3859FF;        /* cor de marca / links */
  --accHi: #3859FF;      /* CTA primário */
  --accSoft: #EDF1FF;    /* fundo suave de marca */
  --bg: #FFFFFF;
  --bg2: #F8F9FC;
  --bg3: #F1F5F9;
  --tx: #030616;         /* texto primário */
  --tx2: #677489;        /* texto secundário */
  --ln: #E6EAF2;         /* bordas */
  --card: #FFFFFF;
  --sh: 0 1px 2px rgba(3,6,22,.06);
  --font: 'Inter', system-ui, sans-serif;
}
[data-theme="dark"] {
  --acc: #5C77FF;
  --accHi: #3859FF;
  --accSoft: rgba(92,119,255,.13);
  --bg: #0A0F1C;
  --bg2: #0D1424;
  --bg3: #141D31;
  --tx: #F2F5FA;
  --tx2: #8B99B4;
  --ln: rgba(255,255,255,.09);
  --card: #0F1626;
  --sh: 0 1px 3px rgba(0,0,0,.45);
  --font: 'Outfit', system-ui, sans-serif;
}
/* Semânticas */
:root {
  --ok: #22C55E; --ok-soft: rgba(34,197,94,.14);
  --warn: #F59E0B; --warn-soft: rgba(245,158,11,.14);
  --err: #E5484D;
}
```

### Fundamentos

| Item | Valor |
|---|---|
| Fontes | **Inter** (light) / **Outfit** (dark) — Google Fonts, pesos 400–700 |
| Marca | `#3859FF` (light) / `#5C77FF` (dark) |
| Raios | cards **16px** · inputs/botões **9px** · pills **99px** |
| Sombra de cartão | `0 1px 2px rgba(3,6,22,.06)` |
| Sombra flutuante (modal, dropdown, toast) | `0 12px 32px rgba(3,6,22,.14)` |
| Semânticas | sucesso `#22C55E` · alerta `#F59E0B` · erro `#E5484D` |

### Padrões obrigatórios

- **Auth** (login, 2FA, recuperar senha): card de **400px**, centrado na viewport,
  **um único CTA primário** por tela. Ações secundárias são links, não botões.
- **Erro de sistema** (404, 500, sem permissão): página cheia, código gigante em indigo
  translúcido (`--acc` com baixa opacidade), **sempre com rota de saída** — botão ou link
  que leva de volta a um lugar útil. Beco sem saída é bug de design.
- **Ação destrutiva**: botão em `--err` (#E5484D) **+ modal de confirmação** antes de executar.
  Nunca destrutiva em um clique só.

Referência visual completa: `OmniHub Subtelas - Standalone.html` (352 KB — **não carregue
inteiro**; consulte pontualmente para confirmar um padrão específico). Logomarca:
`icon-consultor.svg`.

---

## Lacunas do design system — declare, não invente

O DS do OmniHub **não define**: escala de espaçamento, escala tipográfica, alturas de
componente, breakpoints, estados de hover/focus, conjunto de ícones.

> **A regra:** tudo que o DS não define segue o **padrão do shadcn/ui**, que a stack
> (React + Tailwind + shadcn/ui) já traz. Isso vale para você e vale para o `/design` —
> diga a ele explicitamente. Inventar uma escala "que combina" cria um segundo design system
> que ninguém mantém.

Na prática: espaçamento na escala do Tailwind, tipografia/alturas/estados dos componentes
shadcn/ui como vêm, ícones **lucide-react** (o conjunto que o shadcn/ui usa por padrão).
A única sobreposição permitida é a tabela de fundamentos acima: raio, sombra, fonte e cor
do OmniHub vencem o default do shadcn onde conflitarem.

---

## Como conduzir o `/design`

### Ordem de trabalho

1. **Uma leva por fluxo, não o app inteiro de uma vez.** Agrupe as telas do PRD por fluxo
   (auth → fluxo principal → configuração → erros) e rode o `/design` uma leva por vez.
   Leva grande dilui o DS: as primeiras telas saem fiéis, as últimas derivam.
2. **Auth e erros primeiro.** São os fluxos com padrão fechado no DS — servem de calibração.
   Se o `/design` acerta o card de 400px e a página de erro, o resto tende a seguir.
3. **Cada leva recebe o prompt completo** (abaixo), com os tokens de novo. Não confie que
   o contexto da leva anterior sobreviveu.
4. **Verifique cada leva antes da próxima** (checklist no fim). Divergência detectada na
   leva 1 custa um ajuste; detectada na leva 4 custa retrabalho em cadeia.

### O prompt — pronto para colar

Substitua `{{...}}` e cole no `/design`:

```
Construa as telas abaixo seguindo o design system do OmniHub À RISCA.

## Tokens (use estas variáveis CSS, valores exatos — não substitua nem aproxime)
[COLE AQUI O BLOCO tokens.css INTEIRO, da seção "Tokens" acima]

## Regras do design system
- Fontes: Inter no tema light, Outfit no tema dark (Google Fonts, 400–700).
- Raios: cards 16px; inputs e botões 9px; pills 99px.
- Sombras: cartão 0 1px 2px rgba(3,6,22,.06); elementos flutuantes
  (modal, dropdown, toast) 0 12px 32px rgba(3,6,22,.14).
- Semânticas: sucesso #22C55E, alerta #F59E0B, erro #E5484D. Nunca use
  outra cor para estado de sucesso/alerta/erro.
- TODA tela existe nos DOIS temas, light e dark, via [data-theme] e as
  variáveis acima. Nenhuma cor hardcoded fora dos tokens.
- Telas de auth: card de 400px centrado, UM único CTA primário por tela;
  ações secundárias como link.
- Telas de erro de sistema: página cheia, código do erro gigante em indigo
  translúcido, sempre com botão/link de saída.
- Ações destrutivas: botão vermelho #E5484D + modal de confirmação.

## O que o design system NÃO define — siga o shadcn/ui
Espaçamento, escala tipográfica, alturas de componente, breakpoints,
estados de hover/focus e ícones seguem o padrão do shadcn/ui + Tailwind
(ícones: lucide-react). Não invente escalas próprias. Onde o shadcn
conflitar com as regras acima (raio, sombra, fonte, cor), as regras
acima vencem.

## Acessibilidade (requisito, não enfeite)
Contraste AA sobre --bg e --card nos dois temas; foco visível em todo
elemento interativo; alvos de toque mínimos de 44px em mobile.

## Telas desta leva (do PRD, seção "Telas e rotas")
{{cole aqui as linhas das telas desta leva: rota, o que mostra, o que dá pra fazer}}
```

---

## Verificação — a tela saiu ou divergiu?

Rode este checklist em **cada leva** antes de aceitar. Reprovou um item → devolve ao
`/design` apontando o item, não refaz na mão.

- [ ] **Tokens literais.** Grep visual/textual por cor hardcoded: todo `#3859FF`-like deve
  vir de `--acc`/`--accHi`; fundos de `--bg*`/`--card`; textos de `--tx`/`--tx2`; bordas
  de `--ln`. Cor fora da paleta e fora das semânticas é divergência.
- [ ] **Dois temas.** Alterne `data-theme` light↔dark: nada ilegível, nenhum fundo claro
  vazando no dark, fonte troca Inter↔Outfit.
- [ ] **Raios.** Cards 16px, inputs/botões 9px, pills 99px — o erro comum é o default do
  Tailwind (`rounded-md`/`rounded-lg`) passar por cima.
- [ ] **Sombras.** Cartão usa a sombra de cartão; modal/dropdown/toast usam a flutuante.
- [ ] **Padrão de auth.** Card 400px centrado, um CTA primário, secundárias como link.
- [ ] **Padrão de erro.** Página cheia, código gigante translúcido, rota de saída presente.
- [ ] **Destrutivas.** Vermelho `--err` + confirmação. Botão "Excluir" que executa direto
  reprova.
- [ ] **Semânticas corretas.** Sucesso/alerta/erro só nas cores semânticas — toast de
  sucesso em azul de marca é divergência.
- [ ] **Lacunas respeitadas.** Espaçamento/tipografia/estados parecem shadcn default, sem
  escala inventada.
- [ ] **Acessibilidade.** Foco visível ao tabular; contraste do `--tx2` sobre `--bg2`/`--bg3`
  conferido nos dois temas; alvos de toque ≥ 44px.

**Divergência sistemática** (mesmo erro em várias telas) significa que o prompt da leva
não carregou o bloco de regras — refaça a leva com o prompt completo, não corrija tela a tela.
