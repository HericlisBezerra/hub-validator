# 🧭 hub-validator

**O que acontece depois que o PRD existe.**

Um PRD não vira produto sozinho. Entre o documento e a entrega ao cliente há oito passos, e a ordem entre eles não é preferência — é o que separa um produto entregável de um que vaza dado de cliente.

Esta skill é o playbook desses oito passos, com três frentes de trabalho e as ferramentas de cada uma.

Feita para **Claude Code**. Companheira do [hebe-prdcreator](https://github.com/HericlisBezerra/hebe-prdcreator), que produz o PRD que entra aqui.

---

## A ordem

```
1 · PRD                                  skill hebe-prd
2 · Design de telas                      Frente 1  →  /design
3 · Requisitos de segurança              Frente 2  →  os 7 critérios, ANTES do build
4 · Configurações de APIs e organização  Frente 3
5 · Criar o projeto na Lovable           o build
6 · Revisar segurança                    Frente 2  →  os 6 prompts, DEPOIS do build
7 · Validar UI e funcionalidades         Frente 1  →  checklist + fundamentais do PRD
8 · Remix                                enviar ao cliente
```

**Segurança aparece duas vezes de propósito.** No passo 3 são requisitos — regras de construção que entram junto do PRD para a Lovable já nascer certa. No passo 6 é auditoria, sobre o que ela de fato construiu. Prevenção não substitui auditoria; auditoria não recupera o que a prevenção teria evitado.

**A revisão nunca vem antes do build.** A Lovable inventa tabela, policy e endpoint que o PRD não pediu. O risco mora na diferença entre o especificado e o construído — auditar antes é auditar o que ainda não existe.

**O remix é sempre o último.** Ele copia o projeto para outra conta. Remixar antes do passo 6 entrega ao cliente um projeto cuja segurança ninguém verificou; antes do 7, um que ninguém abriu.

## Recebeu o PRD pronto?

Sua entrada é o **passo 2**. Antes de seguir, confira que o PRD traz telas e rotas, modelo de dados com a política de RLS de cada tabela, decisões técnicas derivadas, fora do escopo e integrações.

Faltou algum? **Volte para quem escreveu.** Cada lacuna vira decisão que a Lovable toma sozinha — e é isso que o passo 6 vai ter que caçar depois, com mais trabalho.

## As três frentes

| Frente | Entrega | Passos |
|---|---|---|
| **1 · Design e telas** | as telas pelo `/design` com tokens exatos, dois temas, e o checklist de validação | 2 e 7 |
| **2 · Segurança e RLS** | 7 critérios como requisito + 6 prompts para a Lovable auditar o próprio projeto | 3 e 6 |
| **3 · Configuração da organização** | o onboarding: o que o cliente final configura antes de o produto servir para algo | 4 |

## Por que os prompts de segurança são assim

A [hebe-security-scan](https://github.com/HericlisBezerra/hebe-security) audita **lendo código**, num terminal. Aqui o projeto vive dentro da Lovable e você tem uma janela de chat — então a Frente 2 é **outro veículo**: prompts que você cola lá para que ela audite o que ela mesma construiu.

Isso cria um problema que o desenho enfrenta de frente: **pedir a um modelo que audite o próprio trabalho**. As defesas:

- **Inventário separado do julgamento** — levantar primeiro, julgar depois, para não pular a superfície que ele "sabe" que está certa
- **Evidência literal, nunca opinião** — "liste as policies de cada tabela", jamais "está seguro?"
- **Duas listas obrigatórias no fim** — "provado bloqueado" e "não executado", porque item não testado não é item aprovado
- **Um verificador que não é modelo** — Supabase Advisors, lint de máquina, fora da conversa. Alerta em aberto = auditoria não concluída

## O que a auditoria enxerga que quase não enxergou

`pg_tables` não retorna views. Uma view em `public` é servida pelo PostgREST como se fosse tabela, roda com o privilégio do dono e ignora a RLS de baixo, salvo `security_invoker = true`.

Uma primeira versão desta skill mandava criar uma view sobre a tabela de segredos **e** auditava com `pg_tables` — a view teria passado nos sete critérios, nos seis prompts e na checklist inteira, servindo as chaves de API do cliente para qualquer visitante com a anon key. O inventário hoje usa `pg_class` e cobre view e materialized view, e existe um prompt dedicado a provar que nada referencia a coluna do segredo.

## Configuração da organização, "opcional mas obrigatório"

Chave de API, nome da organização e canal de mensagem são configuração do **cliente final**, não sua. Sem elas o produto abre e não serve — um chatbot sem chave de IA é um chat mudo.

O cadastro nunca bloqueia, mas **inconfigurado é estado de produto, não erro**: a feature aparece visível e travada, com o caminho direto para destravá-la. A feature que o cliente vê e não pode usar é o melhor vendedor do próprio setup.

E a regra que não se dobra: **chave de cliente final nunca chega ao browser.** Tabela sem policy de leitura, `REVOKE ALL` de `anon` e `authenticated` (no Supabase, tabela nova já nasce com `SELECT` concedido), uso só por Edge Function, front vendo apenas provedor e últimos quatro dígitos.

## Instalar

```bash
git clone https://github.com/HericlisBezerra/hub-validator.git
cp -R hub-validator/hebe-hub-validator ~/.claude/skills/
```

Skill local carrega na abertura da sessão — abra uma nova depois de instalar.

## Usar

```
/hebe-hub-validator
```

Ou em linguagem natural: "validar o projeto", "o PRD está pronto e agora", "revisar a segurança do que a Lovable fez", "fazer as telas do projeto".

## Estrutura

```
hebe-hub-validator/
├── SKILL.md                                 os 8 passos e o roteamento
└── references/
    ├── 01-design-e-telas.md                 tokens, padrões, condução do /design
    ├── 02-seguranca-e-rls.md                critérios, 6 prompts, gate do Advisors
    └── 03-configuracao-da-organizacao.md    o onboarding e onde vivem as chaves
```

Sem script, sem dependência. Nada executa; é instrução.

## O que ainda não foi validado

Os prompts **não foram testados contra uma sessão real da Lovable**. A eficácia contra a complacência do gerador é desenho informado, não resultado medido.

Sobre o remix, o comportamento já é conhecido: com o **Supabase integrado da Lovable**, ele copia o banco junto, refaz as secrets e reconfigura o SMTP — automaticamente, sem carregar conexão para o Supabase de origem.

Sobram duas bordas: **o banco é copiado, então o dado vai junto** (limpe o projeto-molde antes), e **se o projeto usa o Supabase próprio do cliente** não há banco da Lovable para copiar — aí o remix carrega a conexão para um banco externo, e vale conferir o que atravessa.

## Licença

MIT — ver [LICENSE](LICENSE). Feito pela **HeBeDigital** ([@hericlisbezerra](https://github.com/HericlisBezerra)).
