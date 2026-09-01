# Frente 3 — Configuração da organização

Todo mini-SaaS nasce **vazio de contexto**: o produto sabe o que faz, mas não sabe de quem é,
como se chama a operação do cliente, nem com que ferramentas dele precisa falar. Esta frente
define o **onboarding de configuração** — a primeira coisa que o cliente final vê depois do
cadastro — e o que o Héric cola na Lovable para construí-lo.

> **Quem configura é o cliente final** — a pessoa que comprou o mini-SaaS e precisa plugá-lo
> nas ferramentas *dela*. Não é o Héric, não é admin de plataforma. Todo texto de tela, toda
> instrução de "onde pego essa chave", fala com essa pessoa.

---

## 1. Derivar do PRD o que precisa ser configurado

A lista de configurações **não se inventa — se extrai do PRD**. Leia estas seções, nesta ordem:

| Seção do PRD | O que sai dela |
|---|---|
| **Decisões técnicas derivadas** | Cada integração derivada vira um passo de setup. "Endpoint de LLM" → passo de chave de IA. "E-mail transacional" → passo de remetente. |
| **O produto / Regras de negócio** | Menções a canal (WhatsApp, e-mail, voz) viram passo de conexão de canal. Menção a áudio/voz → passo de chave de voz (ex.: ElevenLabs). |
| **Usuário e papéis** | Se há equipe/organização → passo de identidade da organização e convite de membros. Se é uso individual → identidade vira só "seu nome/sua marca". |
| **Modelo de dados** | Entidade `organizacao`/`empresa`/`agente` no modelo → seus campos editáveis são setup (nome da empresa, nome do agente, logo). |
| **Requisitos funcionais** | Todo FR que depende de serviço externo aponta a configuração que o destrava. Anote o par: *FR-X depende do passo Y*. |
| **Fora do escopo** | O que está fora não ganha passo de setup. Passo de coisa cortada é escopo voltando pela janela. |

O produto da leitura é uma tabela — leve-a pronta para o bloco da Lovable (seção 5):

```
| Passo | O que pede | Destrava o quê (FRs) | Obrigatório p/ valor? |
```

**Teste do passo legítimo:** se o passo não destrava nenhum FR, ele não existe. Setup sem
consequência é burocracia.

---

## 2. A regra "opcional mas obrigatório"

Na fala do dono: *"ele é opcional, mas ao mesmo tempo obrigatório. Não tem como a Nina
responder aos clientes se não tiver uma chave de API da OpenAI. O chat vai ficar em aberto."*

Resolvido em produto, isso vira **três regras**:

> **Regra 1 — Nunca bloqueie o cadastro.** O cliente entra, olha, navega. Wizard de setup
> aparece no primeiro login, mas **todo passo tem "pular por enquanto"**. Setup forçado antes
> de ver o produto é onde se perde o cliente que acabou de pagar.

> **Regra 2 — Inconfigurado é um estado de produto, não um erro.** Cada feature declara suas
> dependências de configuração. Dependência faltando → a feature **aparece, mas em estado
> bloqueado**: visível, explicada, com um CTA que leva direto ao passo que a destrava. A Nina
> sem chave de IA mostra o chat com os clientes chegando e um banner: *"A Nina está muda —
> conecte uma chave de IA para ela responder. [Configurar agora]"*. O dado real parado na tela
> é o melhor vendedor do setup. Esconder a feature seria esconder o motivo de configurá-la.

> **Regra 3 — O estado de setup é persistente e visível.** Um card/checklist de "Configuração
> da conta" no dashboard, com progresso (ex.: 2 de 4), fica presente até 100% — e some depois.
> Cada item pendente diz **o que destrava**, não só o que pede: "Conectar WhatsApp — para a
> Nina falar com seus clientes", não "Configurar Evolution API".

Comportamento por estado, resumido:

| Estado | O produto faz |
|---|---|
| Nada configurado | Tudo navegável; features dependentes em estado bloqueado com CTA; checklist visível |
| Parcial | Só as features com dependência faltante ficam bloqueadas; o resto funciona pleno |
| Completo | Checklist some; nenhum banner de setup em lugar nenhum |
| Chave inválida/revogada | A feature **volta** ao estado bloqueado, com o erro dito em português e CTA para trocar a chave — nunca falha silenciosa |

---

## 3. O que sempre aparece no setup

Independente do produto, três blocos são recorrentes. O PRD diz quais existem e com que cara:

1. **Identidade da organização** — nome da empresa, nome do agente/produto interno (a "Nina"
   do cliente pode se chamar outra coisa), logo. É o único bloco quase sempre presente.
2. **Chaves de IA** — o cliente escolhe o provedor (OpenAI · Anthropic · DeepSeek — os que o
   PRD suportar) e cola a chave dele. Um provedor ativo por vez, salvo o PRD pedir mais.
   Se o produto usa voz, a chave de voz (ex.: ElevenLabs) é um item separado.
3. **Canal de mensagem** — e aqui está a ramificação que o wizard precisa suportar:

> **Regra da ramificação:** a escolha do canal **muda os campos seguintes**. O wizard não é
> uma lista fixa de perguntas — é uma árvore.

| Escolheu | O passo pede em seguida |
|---|---|
| WhatsApp via Evolution | URL da instância + chave de API da Evolution, **antes** de qualquer QR code |
| WhatsApp via Zapify (ou similar) | As credenciais do Zapify, no ato |
| Outro canal do PRD (e-mail, SMS…) | As credenciais daquele provedor |
| "Depois" | Passo marcado pendente; features de canal em estado bloqueado (regra 2) |

Cada campo de chave tem uma linha de ajuda com **onde o cliente encontra aquilo** no painel
do provedor. Campo de chave sem ajuda gera ticket de suporte.

---

## 4. Onde a configuração vive

Duas naturezas, dois lugares. **Esta separação é a decisão de segurança da frente.**

**Configuração não sensível** (nome, logo, provedor escolhido, canal escolhido, flags de
passo concluído): tabela `organization_settings` (1:1 com a organização), RLS por `org_id`,
lida e escrita pelo front normalmente.

**Chaves de API do cliente** (OpenAI, ElevenLabs, Evolution…):

> **Regra que não se quebra:** chave de API de cliente final **nunca chega ao browser** —
> nem no bundle, nem numa resposta de SELECT, nem em `localStorage`. No Supabase, qualquer
> tabela que o client consegue ler via RLS está exposta no browser de qualquer membro da
> organização — e chave no browser é chave vazada num print, numa extensão, num XSS.

O desenho:

- Tabela separada `organization_secrets` (`org_id`, `provider`, `secret`, `last4`,
  `created_at`). **RLS sem nenhuma policy de SELECT/UPDATE/DELETE para o client** — a tabela
  é invisível ao front por padrão.
- 🚨 **E RLS não basta.** No Supabase, tabela nova em `public` **já nasce com `SELECT`
  concedido a `anon` e `authenticated`** por default privileges. Rode sempre:
  ```sql
  REVOKE ALL ON public.organization_secrets FROM anon, authenticated;
  ```
  Sem isso, a RLS é a única coisa entre a chave do cliente e a anon key — que é pública.
- **Escrita** via Edge Function `save-secret`: valida o JWT do usuário, confere papel de
  admin da org, grava com service role. A chave viaja uma vez, por HTTPS, e nunca volta.
- **Uso** via Edge Functions: toda chamada de IA/voz/canal sai de uma Edge Function que lê a
  chave com service role e chama o provedor. O front chama a função, nunca o provedor.
- **Exibição** no front: só `provider`, `last4` e `created_at` — "OpenAI · ••••4f2a ·
  conectada em 12/03". Trocar = sobrescrever; nunca existe botão "ver chave".
  🚨 **Nunca por view.** View em `public` é servida pelo PostgREST como se fosse tabela, roda
  com o privilégio do dono e **ignora a RLS de baixo** salvo `security_invoker = true`. Uma view
  sobre a tabela de segredos é o caminho mais curto para vazar a chave do cliente — e a
  auditoria da Frente 2 só a enxerga porque foi corrigida para isso.
  **O jeito certo:** grave `provider`, `last4` e `created_at` em `organization_settings`
  (tabela normal, com RLS por org, sem a coluna do segredo), ou devolva-os pela mesma Edge
  Function que salva. A coluna `secret` **nunca** é referenciada por view, função ou trigger.
- **Validação no ato:** ao salvar, a Edge Function faz uma chamada barata de teste ao provedor
  e devolve verde/vermelho na hora. Chave errada descoberta no primeiro uso real é a pior
  experiência possível de onboarding.

**Suposição declarada:** criptografia da coluna `secret` em repouso (ex.: Supabase Vault ou
`pgsodium`) é desejável, mas depende do que a Lovable consegue gerar de forma confiável — trate
como melhoria pós-geração, a validar na Frente 2 (segurança/RLS). O inegociável desta frente é
o desenho acima: **client nunca lê a tabela de segredos; provedor só é chamado do servidor.**

---

> **Antes de seguir:** a Frente 2 tem um prompt dedicado (Prompt 6) que verifica exatamente
> isto — que nenhum objeto do banco referencia a coluna de segredo, e que `anon`/`authenticated`
> não têm privilégio na tabela. Rode-o depois do build.

## 5. O que passar para a Lovable — pronto para colar

Preencha `{{...}}` com a tabela derivada na seção 1 e cole junto do PRD:

```
## Onboarding de configuração da organização

Após o primeiro login, mostre um wizard de configuração. Regras:

1. NENHUM passo bloqueia o uso: todo passo tem "Pular por enquanto".
   O cadastro nunca depende do setup.
2. Passos do wizard (derivados do PRD):
{{tabela: passo · o que pede · quais FRs destrava}}
3. O passo de canal é RAMIFICADO: a escolha do canal define os campos
   seguintes ({{ex.: Evolution → URL da instância + chave de API antes
   do QR code; Zapify → credenciais do Zapify}}).
4. Cada campo de chave tem texto de ajuda dizendo onde o cliente
   encontra aquele valor no painel do provedor.
5. Estado inconfigurado é estado de produto: feature com dependência
   faltante aparece BLOQUEADA (visível, com explicação em uma frase e
   botão que abre o passo que a destrava) — nunca escondida, nunca
   quebrada. Ex.: {{ex. do produto — "chat visível, agente muda, banner
   'Conecte uma chave de IA' "}}.
6. Dashboard tem um card "Configuração da conta" com progresso (X de N)
   e link por item pendente. Some quando completo.
7. Chave inválida ou revogada devolve a feature ao estado bloqueado,
   com mensagem em português e CTA para trocar a chave.

## Segurança das chaves (inegociável)
- Config não sensível: tabela organization_settings, RLS por org_id.
- Chaves de API: tabela organization_secrets SEM policy de SELECT para
  o client. Escrita só via Edge Function (JWT validado + papel admin,
  grava com service role). Todas as chamadas a provedores externos
  (IA, voz, WhatsApp) saem de Edge Functions que leem a chave no
  servidor — o front NUNCA recebe a chave nem chama provedor direto.
- Front exibe apenas provedor + últimos 4 dígitos + data. Sem "ver
  chave". Trocar = sobrescrever.
- Ao salvar uma chave, a Edge Function testa com uma chamada barata ao
  provedor e retorna sucesso/erro imediato.
- Só o papel admin da organização vê e mexe no setup de chaves.
```

**Depois que a Lovable gerar:** confira que nenhuma chamada a provedor externo partiu do
front e que `organization_secrets` não tem policy de leitura para o client — e mande a
Frente 2 (`02-seguranca-e-rls.md`) validar o resto.
