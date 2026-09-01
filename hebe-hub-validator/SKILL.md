---
name: hebe-hub-validator
description: "Valida e completa um mini-SaaS gerado na Lovable a partir de um PRD: constrói as telas com o design system do OmniHub, especifica o onboarding de configuração da organização (chaves de API, nome, canais) e audita RLS e segurança do que a Lovable inventou — antes de compartilhar por remix. Use quando o usuário disser 'validar o projeto', 'rodar o validator', 'o PRD está pronto e agora', 'revisar a segurança do que a Lovable fez', 'fazer as telas do projeto', ou ao terminar um PRD e partir para a construção. NÃO use para escrever o PRD — isso é a skill hebe-prd."
user-invocable: true
---

# Hub Validator

O que acontece **depois** que o PRD existe. Três frentes que transformam um documento num
produto entregável — e uma ordem de execução que não é negociável.

---

## A ordem — os 8 passos, sem pular nenhum

Todo projeto segue esta sequência. Não é sugestão: cada passo depende do anterior, e dois
deles existem justamente porque o passo seguinte destrói a chance de fazê-los depois.

```
1 · PRD                                    skill hebe-prd
2 · Design de telas                        Frente 1  →  /design
3 · Requisitos de segurança                Frente 2  →  os 7 critérios, ANTES do build
4 · Configurações de APIs e organização    Frente 3
5 · Criar o projeto na Lovable             o build
6 · Revisar segurança                      Frente 2  →  os 6 prompts, DEPOIS do build
7 · Validar UI e funcionalidades           Frente 1  →  checklist + os fundamentais do PRD
8 · Remix                                  enviar ao cliente
```

**Segurança aparece duas vezes, e são coisas diferentes.** No passo 3 são **requisitos**: os 7
critérios entram como regra de construção junto do PRD, para a Lovable já nascer certa. No
passo 6 é **auditoria**: os prompts rodam sobre o que ela de fato construiu. Prevenção não
substitui auditoria, e auditoria não recupera o que a prevenção teria evitado.

**Por que a revisão nunca vem antes do build.** A Lovable inventa tabela, policy e endpoint que
o PRD não pediu. Auditar antes é auditar o que ainda não existe — o risco mora na **diferença**
entre o especificado e o construído.

**Por que o remix é sempre o último.** Ele copia o projeto para outra conta. Remixar antes do
passo 6 é entregar ao cliente um projeto cuja segurança ninguém verificou; antes do 7, é
entregar um que ninguém abriu.

**O que o remix faz sozinho** — confirmado na prática, com o **Supabase integrado da Lovable**:
copia o banco junto, **refaz as secrets** e reconfigura o SMTP. Não carrega conexão para o
Supabase de origem. Duas bordas continuam sendo responsabilidade de quem entrega:

- **O banco é copiado, então o dado vai junto.** Projeto-molde com dado real, de teste ou de
  outro cliente entrega esse dado na conta do novo cliente. **Limpe antes.**
- **Se o projeto usa o Supabase próprio do cliente**, não há banco da Lovable para copiar — o
  remix carrega a **conexão** para um banco externo. Confira o que atravessa antes de compartilhar.

---

## Entrando no meio: quem recebe o PRD pronto

Parte do time não escreve PRD — **recebe**. Nesse caso a entrada é o **passo 2**, e o passo 1
já veio feito.

Antes de seguir, confira que o PRD que chegou tem o que os próximos passos consomem:

- [ ] **Telas e rotas** — o passo 2 constrói a partir daqui
- [ ] **Modelo de dados com a política de RLS de cada tabela** — o passo 3 vira requisito disso
- [ ] **Decisões técnicas derivadas** — diz o que foi decidido e por quê
- [ ] **Fora do escopo** — o que segura a Lovable de inventar
- [ ] **Integrações** — o passo 4 deriva as configurações daqui

**Faltou algum? Volte para quem escreveu o PRD.** Não preencha o buraco no chute: cada lacuna
aqui vira decisão que a Lovable toma sozinha, e é exatamente isso que o passo 6 vai ter que
caçar depois — com mais trabalho.

---

## As três frentes

| Frente | O que entrega | Quando roda | Referência |
|---|---|---|---|
| **1 · Design e telas** | as telas pelo `/design`, tokens exatos, dois temas — e o checklist de validação | passos **2** e **7** | `references/01-design-e-telas.md` |
| **2 · Segurança e RLS** | os 7 critérios como requisito, e 6 prompts para a Lovable auditar o próprio projeto | passos **3** e **6** | `references/02-seguranca-e-rls.md` |
| **3 · Configuração da organização** | a spec do onboarding: o que o cliente final configura antes de o produto servir para algo | passo **4** | `references/03-configuracao-da-organizacao.md` |

Abra a referência da frente que você vai executar. Não carregue as três de uma vez.

---

## O que cada frente resolve, em uma linha

**Frente 1** — o `/design` do Claude não é skill, é comando. Ele recebe **valor exato de token**,
nunca descrição de cor. Trabalhe por leva de fluxo, com auth e erros primeiro como calibração.

**Frente 2** — a `hebe-security-scan` audita código lendo arquivo, num terminal. Esta frente é
**outro veículo**: prompts que você cola na janela da Lovable, porque é lá que o projeto vive.
Peça **evidência, nunca opinião** — "liste as policies de cada tabela", jamais "está seguro?".

**Frente 3** — chave de API, nome da organização e canal de mensagem são **configuração do
cliente final**, não sua. Sem elas o produto abre e não serve: a Nina sem chave de IA é um chat
mudo. É **opcional, mas ao mesmo tempo obrigatório** — o cadastro nunca bloqueia, mas a feature
sem dependência fica visível e travada, com o caminho para destravá-la.

---

## Regras que valem nas três

**Nunca antes do PRD.** Esta skill consome um PRD pronto. Sem ele, as três frentes viram chute:
não há de onde derivar as configurações, nem lista de telas, nem o que a Lovable deveria ter
construído. Se não existe PRD, a skill certa é a `hebe-prd`.

**Chave de cliente final nunca chega ao browser.** Tabela sem policy de leitura, escrita e uso
só por Edge Function. No Supabase, tabela legível por RLS chega ao navegador de qualquer membro
da organização — e chave no browser vaza por print, extensão ou XSS.

**Segurança não fecha num modelo só.** A Frente 2 acha; o veredito de alto risco é adjudicado
pelo `@advisor` ou pela `hebe-security-scan`. Item "não executado" que envolva dinheiro,
permissão ou dado de cliente **não é risco assumível** — é trabalho pendente.

**Mudou o schema, roda de novo.** A tabela criada na terça não é coberta pela auditoria de
segunda. A Frente 2 é cíclica, não um carimbo de uma vez.

---

## Use tudo que a Lovable já oferece

Antes de construir qualquer coisa à mão, verifique se a plataforma já entrega. Recurso nativo
não usado é trabalho refeito pior — e que some na próxima atualização dela.

| Nativo da Lovable | Use para | Não faça |
|---|---|---|
| **Supabase integrado** (ou o Supabase próprio do cliente, se ele já tiver) | banco, auth, storage, RLS | subir banco à parte ou API intermediária sem necessidade |
| **Auth do Supabase** | sessão, e-mail, OAuth, recuperação de senha, MFA | autenticação na mão |
| **Edge Functions** | chamar provedor de IA, guardar segredo, webhook | chave de API no front |
| **Storage + policy** | upload de arquivo do cliente | bucket público "pra facilitar" |
| **Revisor / preview** | ver antes de publicar, a cada leva | publicar e descobrir na produção |
| **SEO nativo** — meta, título, Open Graph, sitemap | página encontrável | produto que nasce invisível |
| **Infraestrutura e deploy da própria Lovable** | publicar e versionar | pipeline paralelo que ninguém mantém |

**Cliente com Supabase próprio:** conecte o existente, nunca crie um segundo. Dois bancos para
o mesmo produto é dado dividido — e some da auditoria da Frente 2, porque ela olha um só.

**No fechamento**, confirme item a item: o que a Lovable oferecia foi usado, ou há um motivo
escrito para não ter sido.

---

## Erros que custam caro

- **Confundir os dois momentos de segurança.** Passo 3 é requisito, passo 6 é auditoria. Fazer
  só o 3 é confiar que a Lovable obedeceu; fazer só o 6 é caçar o que dava para ter evitado.
- **Pular o passo 7.** Segurança aprovada não quer dizer que o produto funciona. Ninguém abriu
  a tela, ninguém testou o fundamental do PRD.
- **Fazer remix antes da Frente 2.** Compartilha para outra conta um projeto não verificado.
- **Deixar a configuração da organização para depois.** Vira tela improvisada e chave no `.env`
  do front — que é o mesmo que publicar a chave.
- **Passar cor por descrição para o `/design`.** "Azul da marca" não é `#3859FF`. Valor exato
  constrói tela; adjetivo produz aproximação.
- **Aceitar "está tudo seguro" como resposta.** Modelo gerador é complacente com o próprio
  trabalho. Sem evidência colada na conversa, não houve auditoria.
- **Reconstruir o que a Lovable já dá.** Auth na mão, banco à parte, deploy paralelo — trabalho
  a mais que envelhece pior que o nativo.
