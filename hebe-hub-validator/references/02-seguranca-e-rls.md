# Segurança e RLS — auditoria do que a Lovable construiu

Esta frente valida a segurança de um projeto **depois** que a Lovable o gerou a partir do PRD.
O veículo é diferente de tudo que já existe na casa: aqui não há agente lendo repositório.
Há **prompts prontos para colar na janela da Lovable**, para que ela mesma audite, corrija
e prove o que construiu.

> **Princípio: audita-se o que nasceu, não o que foi pedido.**

---

## Por que depois, nunca antes

A Lovable inventa. Ela cria tabela de apoio que o PRD não mencionou, policy que "faz funcionar",
edge function auxiliar, bucket para um upload que surgiu no meio. Auditar o PRD não encontra
nada disso — **o PRD não sabe o que a Lovable inventou.** O risco mora exatamente na diferença
entre o que foi pedido e o que foi construído.

Por isso a ordem é fixa: PRD → build na Lovable → **esta auditoria** → publicar.
E ela **se repete depois de cada rodada grande de mudanças** — tabela nova criada na terça
não é coberta pela auditoria de segunda.

---

## Divisão de trabalho com a `hebe-security-scan`

A skill `hebe-security-scan` (17 categorias, ~104 itens) é a auditoria profunda, feita por
agente com o repositório aberto. **Esta frente não a repete** — ela cobre o que dá para
verificar e corrigir de dentro da janela da Lovable, que é onde o Héric está.

| Fica aqui (janela da Lovable) | Fica lá (repo aberto, `hebe-security-scan`) |
|---|---|
| RLS, policies, WITH CHECK, campos sensíveis | Race conditions, lógica de negócio, injeção |
| Buckets de storage e edge functions | Rate limiting, pagamentos, dependências |
| Secrets no bundle do client (`VITE_`) | Histórico do git, headers/CORS, mass assignment fora do banco |

Quando o projeto sair da Lovable para um repo (export GitHub), rode a `hebe-security-scan`
completa. Esta frente é o portão mínimo antes de publicar — não substitui a auditoria total.

---

## Os critérios fundamentais

O mínimo inegociável de um Supabase gerado por IA. Não é checklist de 104 itens — é o que
esse tipo de gerador erra com frequência e que causa dano real, em ordem de estrago:

1. **Tabela sem RLS.** Tabela nova nasce com RLS desligada por padrão no Postgres. Sem RLS,
   qualquer pessoa com a `anon` key — que é pública por design — lê e escreve tudo.
2. **Policy que só checa "está logado".** `USING (true)` ou `USING (auth.uid() IS NOT NULL)`
   deixa qualquer conta recém-criada ler os dados de todo mundo. Policy tem que comparar
   `auth.uid()` com uma **coluna da linha**.
3. **`INSERT`/`UPDATE` sem `WITH CHECK`.** `USING` controla o que se lê; sem `WITH CHECK`,
   o usuário grava linha apontando o `user_id` de outra pessoa — forja ou rouba propriedade.
4. **Campo sensível gravável pelo dono da linha.** Policy de update "correta" (checa dono)
   ainda deixa o usuário fazer `SET role = 'admin'`, `credits = 99999` na própria linha.
5. **`service_role` ou secret no client.** Variável `VITE_*` é inlined no bundle — qualquer
   visitante lê. A `service_role` key ignora toda RLS; no client, o jogo acabou.
6. **Bucket de storage público ou sem policy.** Storage é superfície separada das tabelas.
   Bucket esquecido deixa um usuário ler/apagar arquivo dos outros.
7. **Edge function sem verificação de auth.** É um endpoint POST público. Se não valida o
   JWT e a propriedade do recurso, qualquer um chama.

Tudo isso deriva da regra-mãe: **nunca confie no client.** O que só existe no browser,
o atacante controla.

---

## Os prompts — colar na Lovable, nesta ordem

Regras de uso, antes de colar o primeiro:

- **Um prompt por vez.** Espere a resposta, confira a evidência, só então avance.
- **Resposta sem evidência não vale.** Se a Lovable disser "está tudo seguro" sem mostrar
  tabela, policy literal ou resultado de query, cole de novo: *"não pedi opinião, pedi a
  evidência — mostre o resultado da query e o texto de cada policy."* Modelo gerador tende
  a aprovar o próprio trabalho; a defesa é exigir o dado bruto.
- **Suposição declarada:** os prompts assumem que a Lovable consegue rodar SQL no Supabase
  conectado e ler as próprias edge functions e variáveis. Se ela não conseguir rodar alguma
  query, rode-a você no **SQL Editor** do painel do Supabase e cole o resultado de volta
  no chat — o prompt seguinte continua valendo.

### Prompt 1 — Inventário (mapear antes de julgar)

```
Aja como um auditor de segurança externo vendo este projeto pela primeira vez. Nesta etapa
você NÃO corrige nada e NÃO opina se algo está seguro — só levanta evidência.

Me entregue, em tabelas:

1. Todas as tabelas do schema public com o status de RLS. Rode e mostre o resultado de:
   -- tabelas E views E materialized views: pg_tables NÃO retorna view,
   -- e view em public é servida pelo PostgREST como se fosse tabela
   SELECT c.relname,
          c.relkind,          -- r=tabela  v=view  m=matview  p=particionada
          c.relrowsecurity,
          c.reloptions        -- security_invoker aparece aqui
     FROM pg_class c
     JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE n.nspname = 'public' AND c.relkind IN ('r','v','m','p')
    ORDER BY c.relkind, c.relname;
2. Todas as policies existentes. Rode e mostre o resultado de:
   SELECT tablename, policyname, cmd, roles, qual, with_check FROM pg_policies
   WHERE schemaname = 'public';
3. Todos os buckets de storage: nome, se é público, e as policies de storage.objects
   de cada um (texto literal).
4. Todas as edge functions do projeto: nome, o que faz em uma linha, e se o código
   verifica o JWT do chamador (cite a linha que verifica — ou escreva "NÃO VERIFICA").
5. Todas as variáveis de ambiente usadas no frontend (prefixo VITE_): nome e onde cada
   uma é usada. Não mostre os valores.
6. Todas as funções SECURITY DEFINER, se existirem, e em que schema estão.

Regra: item que você não conseguir verificar recebe "NÃO VERIFICADO" — nunca presuma
que está certo. Não resuma: quero as tabelas completas.
```

### Prompt 2 — Julgamento das policies (o veredito, tabela por tabela)

```
Com base no inventário que você acabou de levantar, julgue cada tabela do schema public.
Você é o auditor, não o autor: trate o projeto como se outro time o tivesse escrito, e
procure o erro, não a confirmação de que está bom.

Para cada tabela, responda com a policy literal como evidência:

a) Um usuário autenticado comum consegue LER linhas que pertencem a outro usuário?
b) Consegue INSERIR ou ATUALIZAR uma linha apontando o user_id (ou org_id) de outra
   pessoa? (procure policy de INSERT/UPDATE sem WITH CHECK)
c) Consegue alterar campos sensíveis da PRÓPRIA linha — role, is_admin, credits, balance,
   plan, subscription, verified ou equivalentes?

Regras de reprovação automática:
- Policy de SELECT/UPDATE/DELETE cujo USING é apenas "true" ou "auth.uid() IS NOT NULL",
  sem comparar auth.uid() com uma coluna da linha → REPROVADA, a menos que a tabela seja
  genuinamente pública (justifique por escrito por que ela pode ser lida por todos).
- Policy de INSERT/UPDATE sem WITH CHECK → REPROVADA.
- Tabela com rowsecurity = false → REPROVADA, sem exceção.

Formato: tabela | veredito (OK / REPROVADA / BLOQUEADA) | evidência (a policy) | por quê.
"BLOQUEADA" = RLS ativa e nenhuma policy — nada passa; marque para eu decidir se é
intencional. NÃO corrija nada ainda: primeiro eu quero ver a lista inteira.
```

> 🚨 **Reprovação automática, sem exceção:**
> - **View em `public` sem `security_invoker = true`** → REPROVADA. Ela roda com o privilégio
>   do dono e **ignora a RLS das tabelas de baixo**.
> - **Materialized view exposta em `public`** → REPROVADA. Matview não tem RLS, ponto.
> - Julgue a **UNIÃO das policies por comando**, nunca cada policy isolada: policies
>   permissivas se somam por OR, e cada uma pode parecer correta enquanto o conjunto abre a tabela.

### Prompt 3 — Storage, edge functions e client (as superfícies fora das tabelas)

```
Agora julgue o que está fora das tabelas, com a mesma postura de auditor externo:

1. STORAGE: para cada bucket, um usuário autenticado comum consegue ler, sobrescrever ou
   apagar arquivo de OUTRO usuário? A policy restringe o caminho à pasta do próprio uid
   (storage.foldername(name))? Bucket marcado como público: justifique por que cada um
   pode ser público, ou marque REPROVADO.
2. EDGE FUNCTIONS: para cada função, mostre o trecho que valida o JWT e o trecho que
   confere se o recurso pertence ao chamador. Função que executa sem usuário válido, ou
   que aceita um id no body sem checar o dono → REPROVADA. Função que usa a service_role
   key: liste o que ela faz com esse poder e por que precisa dele.
3. CLIENT: alguma variável VITE_ contém service_role key, secret de API de terceiro,
   connection string ou secret de JWT? A anon key e IDs públicos são esperados no client;
   qualquer outra credencial é REPROVADA. Se o código do frontend importa a service_role
   key de qualquer forma, é CRÍTICO — destaque no topo da resposta.
4. SECURITY DEFINER: para cada função dessas no schema public, ela é chamável via API por
   qualquer pessoa com a anon key. Diga o que ela permite fazer sem RLS e se valida os
   inputs. Sem validação → REPROVADA.

Formato: item | veredito | evidência | por quê. Só a lista — nada de corrigir ainda.
```

### Prompt 4 — Corrigir (uma reprovação por vez, sem afrouxar nada)

```
Corrija agora TODAS as reprovações das duas etapas anteriores, nesta ordem de prioridade:
1) credencial exposta no client, 2) tabela sem RLS, 3) policy permissiva demais,
4) falta de WITH CHECK, 5) campo sensível gravável, 6) storage, 7) edge functions,
8) SECURITY DEFINER.

Para CADA correção: mostre o SQL (ou diff de código) aplicado, e diga qual feature do app
depende daquela tabela/função e por que ela continua funcionando depois da correção.

Proibições:
- Proibido "corrigir" afrouxando uma checagem ou removendo uma feature para o problema sumir.
- Proibido usar a service_role key no client como atalho para "resolver" um bloqueio de RLS.
- Campo sensível (role, credits, plan...): a correção é impedir o update pelo usuário
  (REVOKE UPDATE na coluna, ou mover a mudança para função controlada) — não é esconder
  o campo na interface.
- Se uma correção quebrar uma feature de verdade, PARE nessa correção e me explique o
  conflito em vez de escolher sozinha entre segurança e feature.

Ao final, liste o que foi corrigido e o que ficou pendente com o motivo.
```

### Prompt 5 — Prova adversarial (mostrar que fechou, não dizer que fechou)

```
Agora prove que as correções fecharam os buracos. Assuma o papel de um atacante que tem
apenas: a anon key (que é pública) e uma conta comum recém-criada (usuário B). No banco
existe um usuário A com dados.

1. Re-rode e mostre o inventário COMPLETO do Prompt 1 (a query de `pg_class`, com
   `relkind` e `reloptions`) **e** `SELECT * FROM pg_policies WHERE schemaname='public';`.
   Reconcilie em números: quantos objetos inventariados, quantos julgados, quantos provados.
   Se não baterem, diga qual sobrou — não siga.
   Toda tabela precisa estar com rowsecurity = true.
2. Para cada tabela, descreva a requisição concreta que o usuário B faria para (a) ler os
   dados do usuário A, (b) gravar uma linha em nome do usuário A, (c) elevar os próprios
   privilégios — e mostre qual policy bloqueia cada tentativa, citando o texto da policy.
3. Faça o mesmo para cada bucket (ler/apagar arquivo do usuário A) e cada edge function
   (chamar sem login, e chamar logado como B passando um recurso do A).
4. Onde você conseguir executar o teste de verdade (query com o papel authenticated
   simulando B), execute e mostre o resultado. Onde não conseguir executar, escreva
   "NÃO EXECUTADO — verificado só por análise" ao lado do item.

Termine com duas listas separadas: "provado bloqueado" e "não executado". Uma resposta
sem a segunda lista está incompleta — eu sei que nem tudo dá para testar daí de dentro.
```

---

### Prompt 6 — Nada enxerga a tabela de segredos

```
Liste TODO objeto do banco que referencia a coluna de segredo da tabela de chaves
da organização — view, materialized view, função, trigger, ou qualquer policy que
a exponha. Mostre a definição de cada um.

A resposta correta é: NENHUM, além da Edge Function que a usa.

Mostre também o resultado de:
  SELECT grantee, privilege_type FROM information_schema.role_table_grants
   WHERE table_name = 'organization_secrets';

Se 'anon' ou 'authenticated' aparecerem com qualquer privilégio, rode:
  REVOKE ALL ON public.organization_secrets FROM anon, authenticated;
e mostre o resultado da query de novo.
```

**Por que este prompt existe:** no Supabase, tabela nova em `public` **já nasce com `SELECT`
concedido a `anon` e `authenticated`** por default privileges — a RLS é a única coisa segurando.
E uma view sobre essa tabela contorna a RLS inteira. Este é o caminho mais curto para vazar
a chave de API de um cliente pagante.

---

## Como saber que a auditoria valeu

> 🔒 **Gate obrigatório, e não é modelo nenhum:** abra o painel do Supabase em
> **Database → Advisors → Security** e cole o print do resultado. É lint de máquina, roda fora
> da conversa, e sinaliza exatamente `rls_disabled_in_public`, `security_definer_view`,
> `function_search_path_mutable` e `auth_users_exposed`. Um clique.
> **Advisor com alerta em aberto = auditoria não concluída**, independentemente do que a
> Lovable tenha respondido nos prompts.


O projeto só publica quando **tudo isso estiver provado com evidência na conversa** — não
com a frase "está seguro":

- [ ] Toda tabela do `public` com `rowsecurity = true` (resultado da query, do Prompt 5)
- [ ] Nenhuma policy `USING (true)` / `auth.uid() IS NOT NULL` sem justificativa escrita de dado público
- [ ] Toda policy de `INSERT`/`UPDATE` com `WITH CHECK`
- [ ] Campos de privilégio/valor (`role`, `credits`, `plan`...) não graváveis pelo próprio usuário
- [ ] Nenhuma credencial além da anon key no client; bucket público só com justificativa
- [ ] Toda edge function com validação de JWT e de propriedade citada linha a linha
- [ ] A lista "não executado" do Prompt 5 **lida e aceita** — cada item dela é risco assumido
  conscientemente, ou motivo para testar por fora (SQL Editor, ou a `hebe-security-scan`
  no repo exportado)

Item da lista "não executado" que envolva dinheiro, permissão ou dado de cliente **não é
aceitável como risco assumido** — vai para teste real antes de publicar.

E a regra que fecha o ciclo: **mudou o schema, rodou de novo.** No mínimo os Prompts 1 e 5
depois de cada leva de features novas — o custo é colar dois textos; o custo de não colar
é a tabela de terça aberta ao mundo.
