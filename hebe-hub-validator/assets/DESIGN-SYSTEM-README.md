# OmniHub — Solutions Design System (v1)

Design system para as SOLUÇÕES que rodam dentro do Hub (ONovoMercado).
Regra central de marca: dentro de uma solução usa-se APENAS o ícone — a logo completa "omnihub" fica restrita ao shell do Hub (login, sidebar, splash).

## Arquivos
- **OmniHub Solutions DS — Standalone.html** — abra em qualquer navegador, funciona offline. Contém: regra do ícone (certo/errado), cores, tipografia, espaçamento, raios & sombras, botões, badges, inputs, iconografia e motion. Toggle light/dark no topo.
- **tokens.css** — tokens de tema (light + dark) prontos para uso.
- **icon-consultor.svg** — o ícone da marca (aplicar via CSS mask para herdar a cor do contexto).

## Fundamentos
- **Fontes:** Inter (light) / Outfit (dark) — Google Fonts, pesos 400–700.
- **Acento:** #3859FF (light) / #5C77FF (dark) · soft #EDF1FF.
- **Semânticas:** sucesso #22C55E · aviso #F59E0B · erro #E5484D.
- **Raios:** 7–8px chips/inputs · 9–12px CTAs · 14px cards · 16px modais · 99px pills.
- **Motion:** cubic-bezier(.25,1,.5,1); splash da solução usa só o ícone (zoom+blur, ~1s), sem letras em cascata.

## Regra do ícone (resumo)
- Ícone isolado em 48 / 32 / 20px via CSS mask.
- Em chip: container 32px raio 9px, ícone 18px.
- Sobre gradiente/fundo escuro: ícone branco puro.
- Assinatura permitida: badge "Uma solução OmniHub" (ícone 16px + texto 11.5px) — única menção nominal ao Hub dentro da solução.
