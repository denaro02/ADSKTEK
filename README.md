# ADSTEK — central de anúncios do Meta

Servidor em **Node.js + TypeScript**. Na **Vercel** ele roda como função (pasta `api/`) e guarda tudo num **banco MySQL na nuvem** (TiDB Cloud, Aiven, Railway, Hostinger…). No computador ou num servidor próprio ele roda com `npm start` e pode usar SQLite.

O que tem dentro: login da equipe com telas e contas liberadas por pessoa, números do Meta (pelo Adspirer ou direto pela API do Meta), pausar/escalar/duplicar com conferência antes, Estratégia de anúncio com IA, Pixel e eventos, cadastros e depósitos do sistema da casa (API, webhook ou puxada), afiliados, log do sistema, webhooks de saída e o **Piloto automático**.

> **Repositório privado.** Nada de senha, chave ou token aqui dentro: tudo vai nas variáveis de ambiente da Vercel (ou num arquivo `.env` que nunca sobe para o GitHub).

---

## 1. Banco de dados na nuvem (5 minutos)

Qualquer MySQL 8 serve. Sugestão grátis: **TiDB Cloud Starter** (compatível com MySQL).

1. Crie a conta em tidbcloud.com e um cluster **Starter**.
2. Em **Connect**, escolha “General” e anote: host, porta (4000), usuário, senha.
3. Crie o banco: no SQL Editor rode `CREATE DATABASE adstek;`
4. Guarde esses dados para o passo 3. Com TiDB, use `DB_SSL=1`.

As tabelas são criadas sozinhas no primeiro acesso. Se você já tinha a versão PHP rodando num MySQL, pode apontar para o **mesmo banco**: os dados, as chaves cadastradas e as senhas continuam valendo (cada pessoa só precisa entrar de novo).

## 2. GitHub

1. Crie um repositório **privado** (ex.: `adstek`).
2. Envie todos os arquivos desta pasta (no site do GitHub: *Add file › Upload files*, arraste tudo; ou `git push`).
3. Confira que **não** subiu nenhum `.env` nem `*.sqlite` (o `.gitignore` já bloqueia).

## 3. Vercel

1. *Add New › Project* e importe o repositório.
2. **Framework Preset: Other.** Deixe Build, Output e Install como estão (o `vercel.json` já cuida).
3. Em **Environment Variables**, cadastre:

| Variável | O que colocar |
|---|---|
| `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS` | dados do banco do passo 1 |
| `DB_SSL` | `1` (TiDB e a maioria dos bancos na nuvem) |
| `APP_SEGREDO` | um texto longo e aleatório (40+ caracteres). Protege as chaves guardadas no banco. **Não troque depois** (as chaves cadastradas deixariam de abrir) |
| `CRON_SECRET` | outro texto aleatório (protege o agendador) |
| `ADMIN_NOME`, `ADMIN_EMAIL`, `ADMIN_SENHA` | o seu acesso de administrador, criado no primeiro acesso |
| `IMPORTAR_CONTAS` | `1` para já trazer as contas An 01 e An 02 (só ID, nome e configurações) |
| `APP_NOME` | opcional, padrão `ADSTEK` |

As chaves dos serviços (Adspirer, Meta, Claude) você cadastra depois pela própria tela, em **Integrações › Serviços** (ficam criptografadas no banco). Se preferir, também dá para usar `ADSPIRER_KEY`, `META_TOKEN` e `ANTHROPIC_KEY` como variáveis.

4. **Deploy.** Abra o endereço que a Vercel deu: a central cria as tabelas e o seu administrador sozinha e mostra a tela de login.
5. Depois de entrar: **troque a sua senha** (Minha conta) e **apague a variável `ADMIN_SENHA`** na Vercel (ela só serve para a primeira vez).

## 4. Primeiros passos dentro da central

1. **Integrações › Serviços**: cole a chave do Adspirer (`sk_live_…`), o token do Meta (usuário do sistema com `ads_read`) e a chave da API do Claude (`sk-ant-…`). Cada uma pode valer para todas as contas ou só para algumas. O botão **Testar** confere sem gastar nada. Com o **token do Meta** cadastrado, os números vêm direto do Meta (grátis, sem gastar chamadas do Adspirer) e atualizam sozinhos ao trocar o período ou escolher as datas no calendário. Como a central casa os números com os do Gerenciador: `docs/DADOS-META.md`.
2. **Início › Adicionar conta**: o botão **Buscar contas no provedor** lista as contas que já estão ligadas ao Adspirer ou ao token do Meta; é só escolher.
3. **Equipe e acessos**: crie o login de cada pessoa, escolha as telas e as contas (vê / pode alterar).
4. **Integrações › Sistema da casa**: gere a chave da API para o dev (a documentação completa está em **Documentação para o dev** e em `docs/API-INTEGRACAO.md`).

## 5. Agendador (puxada do sistema da casa + piloto automático)

O piloto automático e a puxada automática precisam que alguém “bata na porta” da central de tempos em tempos.

- **Recomendado:** em cron-job.org (grátis), crie um job a cada **15 minutos** chamando o endereço que aparece em **Integrações › Sistema da casa › Agendador** (`https://SEU-DOMINIO/integracao.php?r=cron&t=…`). Também funciona `https://SEU-DOMINIO/cron?t=…`.
- O `vercel.json` já traz um **Vercel Cron** diário (`/api/cron`, protegido pela `CRON_SECRET`). No plano grátis da Vercel ele só pode rodar uma vez por dia; por isso o cron-job.org.
- Num servidor próprio (`npm start`) o agendador já roda sozinho a cada 5 minutos (desligue com `PILOTO_INTERNO=0`).

## 6. Piloto automático

Em **Ferramentas › Piloto automático**, escolha a conta e:

1. Defina o **CPA alvo** (custo por compra/cadastro). Sem alvo, ele usa o CPA médio da conta na semana.
2. Clique em **Simular agora**: mostra o que ele faria, com o motivo e os números. Não mexe em nada.
3. Comece em **Só sugerir**: ele propõe, alguém com permissão de alterar a conta aprova ou recusa.
4. Quando as sugestões estiverem boas, passe para **Automático**.

As regras, os limites e as travas de segurança estão em `docs/PILOTO.md`.

## 7. Rodar no computador

Precisa do **Node.js 22.5 ou mais novo**.

```bash
npm install
cp .env.example .env        # e edite: para testar sem MySQL, use DB_DRIVER=sqlite
npm run dev                 # compila e abre em http://localhost:8080
```

Com `DB_DRIVER=sqlite` e `ADMIN_EMAIL`/`ADMIN_SENHA` no `.env`, a central se instala sozinha no primeiro acesso (o arquivo do banco fica em `DB_PATH`, padrão `adstek.sqlite`). Sem as variáveis `ADMIN_*`, ela abre a tela de instalação (`/instalar.php`), que pede o `INSTALAR_CODIGO`.

Servidor próprio (VPS, Railway, Render…): `npm install && npm run build && npm start`, com as mesmas variáveis. A porta vem de `PORT` (padrão 8080).

## 8. Estrutura

```
api/app.ts          função da Vercel: todas as páginas e APIs
api/cron.ts         função da Vercel: agendador
src/                servidor (TypeScript)
  app.ts            rotas (mesmas URLs da versão PHP: /api.php, /integracao.php, /instalar.php)
  piloto.ts         piloto automático
  integracao.ts     cadastros, depósitos, afiliados, resumo
  segredos.ts       chaves criptografadas, webhooks, puxada
  adspirer.ts meta.ts claude.ts store.ts nucleo.ts schema.ts db.ts http.ts config.ts tempo.ts bcrypt.ts
  servidor.ts       servidor próprio (npm start)
app/                telas (painel e login)
public/assets/      arquivos estáticos
docs/               documentação da API para o dev, OpenAPI e piloto
```

## Segurança

- Senhas guardadas com scrypt (as antigas, da versão PHP, continuam valendo e são trocadas para scrypt no próximo login).
- Chaves dos serviços e endereços de webhook criptografados (AES-256-GCM) com a `APP_SEGREDO`; a tela só mostra o final de cada chave.
- Chaves da API de integração mostradas uma vez só; no banco fica só o hash.
- Sessão no banco, cookie `HttpOnly` + `Secure` + `SameSite=Lax`, proteção CSRF em toda ação, limite de tentativas de login.
- Toda alteração no Meta confere permissão da pessoa na conta, fica no Histórico e no log do sistema (180 dias).
- O piloto nunca cria, apaga ou reativa nada sozinho, e confere o valor atual no Meta antes de mudar um orçamento.
