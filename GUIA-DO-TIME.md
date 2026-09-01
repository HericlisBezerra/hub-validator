# Guia do Time — do PRD ao cliente

> Anexe este arquivo ao seu Claude. Ele responde as dúvidas do fluxo sem você precisar
> abrir os repositórios.
>
> **HeBeDigital** · dois repositórios, oito passos, uma ordem.

---

## Os dois repositórios

| Repositório | Para quê | Quando |
|---|---|---|
| [hebe-prdcreator](https://github.com/HericlisBezerra/hebe-prdcreator) | entrevista o dono do produto e gera o PRD | passo 1 |
| [hub-validator](https://github.com/HericlisBezerra/hub-validator) | os 8 passos do PRD até a entrega | passos 2 a 8 |

### Instalar os dois

```bash
git clone https://github.com/HericlisBezerra/hebe-prdcreator.git
cp -R hebe-prdcreator/hebe-prd ~/.claude/skills/

git clone https://github.com/HericlisBezerra/hub-validator.git
cp -R hub-validator/hebe-hub-validator ~/.claude/skills/
```

**Abra uma sessão nova depois de instalar.** Skill local só carrega na abertura.

### Usar

```
/hebe-prd              escreve o PRD
/hebe-hub-validator    conduz os 8 passos
```

Ou fale normal: *"vamos criar um PRD"*, *"o PRD está pronto e agora"*, *"revisar a segurança do que a Lovable fez"*.

---

## A ordem — os 8 passos

```
1 · PRD                                  /hebe-prd
2 · Design de telas                      Frente 1  →  /design
3 · Requisitos de segurança              Frente 2  →  7 critérios, ANTES do build
4 · Configurações de APIs e organização  Frente 3
5 · Criar o projeto na Lovable           o build
6 · Revisar segurança                    Frente 2  →  6 prompts, DEPOIS do build
7 · Validar UI e funcionalidades         Frente 1  →  checklist + fundamentais do PRD
8 · Remix                                enviar ao cliente
```

**Não pule passo, e não troque a ordem.** Cada um existe porque o seguinte destrói a chance
de fazê-lo depois.

---

## Passo a passo — o que fazer e o que verificar

### 1 · PRD

**Faz:** roda `/hebe-prd`. São 14 perguntas de negócio (nenhuma de tecnologia) mais o nome
do produto. A skill não entrega o PRD antes de todas respondidas.

**Verifica antes de seguir:** o `.md` saiu com telas e rotas, modelo de dados com a política
de RLS de cada tabela, decisões técnicas derivadas, fora do escopo e integrações.

> **Se você recebeu o PRD pronto,** sua entrada é o passo 2 — mas confira essa lista mesmo
> assim. Faltou algo? **Volte para quem escreveu.** Não preencha no chute: cada lacuna vira
> decisão que a Lovable toma sozinha, e o passo 6 vai ter que caçar depois.

### 2 · Design de telas

**Faz:** abre a Frente 1 e roda o `/design` do Claude, por **leva de fluxo** — nunca o app
inteiro de uma vez. Auth e telas de erro primeiro, que servem de calibração.

**Verifica:** os tokens foram respeitados à risca, os dois temas funcionam, e os padrões de
auth, erro e ação destrutiva foram aplicados.

> **Passe valor exato, nunca descrição.** "Azul da marca" não é `#3859FF`. Adjetivo produz
> aproximação; valor constrói tela.

### 3 · Requisitos de segurança

**Faz:** pega os 7 critérios da Frente 2 e **cola junto do PRD** ao criar o projeto. São
regras de construção, para a Lovable já nascer certa.

**Verifica:** os 7 estão no que você vai colar na Lovable, não só na sua cabeça.

### 4 · Configurações de APIs e organização

**Faz:** deriva do PRD o que o **cliente final** vai precisar configurar — chaves de IA, canal
de mensagem, nome e marca da organização. Especifica o onboarding.

**Verifica:** a regra "opcional mas obrigatório" está clara: o cadastro nunca bloqueia, mas a
feature sem a chave fica **visível e travada**, com o caminho para destravá-la.

> **Chave de cliente nunca chega ao browser.** Tabela sem policy de leitura, `REVOKE ALL` de
> `anon` e `authenticated`, uso só por Edge Function, front vendo provedor e últimos 4 dígitos.

### 5 · Criar o projeto na Lovable

**Faz:** cola o PRD (com os requisitos de segurança do passo 3) e deixa construir.

**Verifica:** usou o que a plataforma já dá — Supabase integrado (ou o do cliente, se ele já
tiver), Auth do Supabase, Storage com policy, Edge Functions para qualquer coisa que segure
chave, envio de e-mail nativo, SEO nativo, revisor antes de publicar.

> **Não reconstrua o que a Lovable já entrega.** Auth na mão, segundo banco, Resend por fora,
> deploy paralelo — trabalho a mais que envelhece pior.

### 6 · Revisar segurança

**Faz:** cola os 6 prompts da Frente 2 na Lovable, **um por vez, na ordem**. Inventário →
julgamento das policies → storage/edge/client → correção → prova adversarial → tabela de segredos.

**Verifica:**
- [ ] **Supabase Advisors (Database → Advisors → Security) sem alerta em aberto.** É o único
      verificador que não é modelo. Alerta aberto = auditoria não concluída.
- [ ] Toda tabela **e view** do `public` foi julgada — view sem `security_invoker = true` é
      reprovação automática
- [ ] Nenhum objeto referencia a coluna de segredo além da Edge Function
- [ ] A lista "não executado" foi lida, não só a "provado bloqueado"

> **"Está tudo seguro" não é resposta.** Modelo gerador é complacente com o próprio trabalho.
> Sem evidência colada na conversa, não houve auditoria.

> **Mudou o schema? Roda de novo.** A tabela criada na terça não é coberta pela auditoria de
> segunda.

### 7 · Validar UI e funcionalidades

**Faz:** abre o produto e usa. Confere as telas contra o design system e as funcionalidades
contra o que o PRD chamou de fundamental.

**Verifica:** o que o PRD colocou em "o que não pode faltar" funciona de ponta a ponta. Estados
vazios, erro e carregamento existem — não só o caminho feliz.

> Segurança aprovada não quer dizer que o produto funciona.

### 8 · Remix

**Faz:** compartilha o projeto com a conta do cliente. **Ative Database e Email Send no ato do
remix.**

**Verifica antes do primeiro remix da vida** — uma vez, e nunca mais precisa: a cópia carrega
conexão ao **nosso** Supabase? Carrega secrets? Carrega dados semeados de teste? Se qualquer
resposta for sim, **pare** — o último passo do fluxo vira vazamento.

---

## Perguntas que vão aparecer

**Recebi o PRD e não tem seção de RLS. Sigo?**
Não. Volte para quem escreveu. Sem a política por tabela, o passo 3 não tem o que virar
requisito e o passo 6 audita no escuro.

**Posso rodar a revisão de segurança antes de criar o projeto?**
Não adianta. A Lovable inventa tabela, policy e endpoint que o PRD não pediu — o risco mora na
diferença entre o especificado e o construído. Antes do build você audita o que não existe.
O que **existe** antes é o passo 3, que é prevenção, não auditoria.

**A Lovable disse que está tudo seguro. Posso seguir?**
Não sem evidência. Peça a lista literal de policies, o resultado da query, o print do Advisors.
Ela é otimista sobre o próprio trabalho — é o modo de falha esperado, não exceção.

**O cliente já tem Supabase. Crio outro?**
Nunca. Conecte o existente. Dois bancos para o mesmo produto é dado dividido — e some da
auditoria, que olha um só.

**Preciso de Resend para enviar e-mail?**
Não. A Lovable envia e-mail nativamente. Serviço externo é mais uma conta, mais uma chave,
mais um domínio para verificar e mais uma fatura — para fazer o que já vinha junto.

**Onde guardo a chave de API que o cliente cadastrar?**
Tabela própria, sem policy de leitura para o client, com `REVOKE ALL ON <tabela> FROM anon,
authenticated`. Usada só por Edge Function. **Nunca** exponha por view: view em `public` é
servida como tabela e ignora a RLS de baixo.

**Achei um problema de segurança que a skill não cobre.**
Se envolver dinheiro, permissão ou dado de cliente, **não é risco assumível** — é trabalho
pendente. Fale com o Héric antes de entregar.

**Posso pular a validação de UI se a segurança passou?**
Não. São coisas diferentes. Segurança aprovada só diz que ninguém acessa o que não devia —
não diz que o produto faz o que prometeu.

---

## O que ainda não foi validado

Honestidade sobre os limites, para ninguém assumir garantia que não existe:

- Os prompts de segurança **nunca rodaram contra uma sessão real da Lovable**. A eficácia deles
  contra a complacência do gerador é desenho informado, não resultado medido.
- **O que o remix carrega** para a conta do cliente não foi testado. É o passo 8, e é o teste
  mais importante que falta.

Encontrou o limite de alguma coisa? Reporte — a skill melhora com caso real, não com teoria.

---

*HeBeDigital · MIT*
