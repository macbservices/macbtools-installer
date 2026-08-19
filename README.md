# MacbTools — Instalador

Script de instalação automatizada do MacbTools em uma VPS **Ubuntu Server 22.04**, com
usuário **root** habilitado. Instala e configura tudo do zero: Node.js 20, PostgreSQL,
Redis (via Docker), Nginx, SSL (Certbot/Let's Encrypt), backend e frontend.

Este repositório instala o código-fonte do repositório
[`macbtools`](https://github.com/macbservices/macbtools) — você vai informar a URL desse
repositório durante a instalação.

> **Dois repositórios, dois níveis de acesso:** este aqui
> (`macbtools-installer`) é **público** — só scripts de infraestrutura, sem
> segredo nenhum. Já o [`macbtools`](https://github.com/macbservices/macbtools)
> (o código-fonte de verdade, com a IA e o sistema de licença) continua
> **privado** — é pra esse que você usa o token do GitHub, na Parte 3 deste
> guia.

> **Quer testar antes numa VM local (VirtualBox) usando seu próprio domínio via
> Cloudflare Tunnel, sem precisar de VPS ainda?** Veja o guia
> [`TESTE_LOCAL.md`](./TESTE_LOCAL.md) e use o script `install_local` em vez do
> `install_primaria`.

## Antes de começar

### 1. Servidor
- VPS com **Ubuntu Server 22.04**, acesso root via SSH
- Mínimo recomendado: 2 vCPU / 4GB RAM / 40GB SSD (para uma instância; some mais RAM se
  for hospedar várias empresas na mesma VPS)

### 2. Domínios (DNS)
Você precisa de **dois subdomínios** apontando para o IP da VPS **antes** de rodar o
instalador (o Certbot valida isso na hora de gerar o SSL):

| Subdomínio             | Aponta para (tipo A) | Exemplo                  |
|-------------------------|-----------------------|---------------------------|
| Painel (frontend)       | IP da VPS             | `app.seudominio.com`      |
| API (backend)           | IP da VPS             | `api.seudominio.com`      |

Confirme que a propagação já aconteceu antes de instalar:
```bash
ping app.seudominio.com
ping api.seudominio.com
```

### 3. Repositório do código-fonte no GitHub
Suba o conteúdo do repositório `macbtools` (backend + frontend) para o **seu** GitHub,
em um repositório privado.

⚠️ **Não use um Personal Access Token clássico da sua conta** pra clonar isso
na VPS do cliente — um token clássico dá acesso a **todos** os seus
repositórios privados, incluindo outros projetos seus que nada têm a ver com
esse cliente.

**Forma recomendada: token fine-grained restrito a este repositório**
(mais simples — é só uma URL pra colar, sem passos extras no servidor do cliente):

1. GitHub → foto de perfil → **Settings** → **Developer settings** →
   **Personal access tokens** → **Fine-grained tokens** → **Generate new token**
2. **Repository access**: marque **"Only select repositories"** → escolha
   só o `macbtools`
3. **Permissions** → **Repository permissions** → **Contents**: `Read-only`
4. Defina a expiração (até 366 dias, ou "No expiration" se sua conta permitir)
5. Gere e copie o token

Esse token **não abre nenhum outro repositório seu**, mesmo que vaze. Se você
usa o [Painel de Licenças](https://github.com/macbservices/macbtools/blob/main/COMO_CONECTAR_PAINEL_DE_LICENCAS.md),
salve esse token lá uma vez — ele já monta a URL de clone pronta toda vez que
você gerar uma licença nova.

Formato da URL a usar no instalador:
```
https://SEU_TOKEN@github.com/macbservices/macbtools.git
```

**Alternativa: Deploy Key** (mais segura ainda, mas com um passo a mais —
a chave é gerada no próprio servidor do cliente, então nem o token viaja):
<details>
<summary>Ver passo a passo da Deploy Key</summary>

1. Na VPS do cliente, gere um par de chaves dedicado:
   ```bash
   ssh-keygen -t ed25519 -C "macbtools-cliente" -f ~/.ssh/macbtools_deploy -N ""
   cat ~/.ssh/macbtools_deploy.pub
   ```
2. Copie a saída desse `cat` (a chave **pública**)
3. GitHub → repositório `macbtools` → **Settings** → **Deploy keys** →
   **Add deploy key** → cole, deixe **somente leitura** marcado
4. Configure o SSH na VPS:
   ```bash
   mkdir -p ~/.ssh
   cat >> ~/.ssh/config << 'EOF'
   Host github.com
     IdentityFile ~/.ssh/macbtools_deploy
     StrictHostKeyChecking no
   EOF
   ```
5. Use a URL SSH no instalador: `git@github.com:macbservices/macbtools.git`
</details>

## Passo a passo da instalação

### 1. Conectar na VPS como root
```bash
ssh root@SEU_IP_DA_VPS
```

### 2. Atualizar o sistema e instalar o Git
```bash
apt update && apt upgrade -y
apt install -y git
```

### 3. Clonar este repositório (o instalador) na VPS
Este repositório (`macbtools-installer`) é **público** — só tem scripts de
infraestrutura, sem código proprietário — então não precisa de token nem
login nenhum pra clonar:
```bash
cd /root
git clone https://github.com/macbservices/macbtools-installer.git
cd macbtools-installer
chmod +x install_primaria install_instancia
```

### 4. Rodar o instalador
```bash
./install_primaria
```

### 5. Responder às perguntas do instalador, nesta ordem

| Pergunta                                          | O que responder                                                        |
|------------------------------------------------------|--------------------------------------------------------------------------|
| Senha para o usuário Deploy e Banco de Dados        | Uma senha forte, **sem caracteres especiais** (ex: `Macb2026Tools`)    |
| Link do GitHub do MacbTools que deseja instalar     | URL do repositório com o token — veja "Repositório" acima, ou cole a URL pronta do Painel de Licenças |
| Nome para a Instância/Empresa                       | `macbtools` (minúsculo, sem espaços/acentos)                          |
| Qtde de Conexões/Whats                              | Ex: `10`                                                                |
| Qtde de Usuários/Atendentes                         | Ex: `10`                                                                |
| Domínio do FRONTEND/PAINEL                          | `app.seudominio.com` (sem `https://`)                                  |
| Domínio do BACKEND/API                              | `api.seudominio.com` (sem `https://`)                                  |
| Porta do FRONTEND                                    | Um valor entre `3000` e `3999`, ex: `3000`                            |
| Porta do BACKEND                                     | Um valor entre `4000` e `4999`, ex: `4000`                            |
| Porta do REDIS                                       | Um valor entre `5000` e `5999`, ex: `5000`                            |
| LICENSE_SECRET (opcional)                            | Cole aqui, ou deixe em branco se não usar licença                      |
| LICENSE_KEY (opcional)                               | Cole a chave gerada no Painel de Licenças pra esse cliente             |

> Envie pro cliente (ou preencha você mesmo, se administrar o servidor) as
> duas últimas linhas já preenchidas — assim ele não precisa editar nenhum
> `.env` manualmente depois da instalação. Veja
> [`COMO_CONECTAR_PAINEL_DE_LICENCAS.md`](https://github.com/macbservices/macbtools/blob/main/COMO_CONECTAR_PAINEL_DE_LICENCAS.md)
> pra gerar essas duas chaves antes de instalar.

A partir daqui o script roda sozinho — leva de 10 a 20 minutos. Ele instala
dependências do sistema, cria o usuário `deploy`, clona o código, sobe o Postgres e o
Redis, builda o backend e o frontend, configura o Nginx e gera o certificado SSL.

### 6. Acessar o painel
Ao terminar, acesse `https://app.seudominio.com` no navegador. O login padrão do
primeiro acesso segue a documentação original do curso (usuário/empresa "Empresa 1"
criados pelo seed do banco) — troque a senha assim que entrar.

## Depois de instalar

- **Configurar a IA (Groq gratuito) e a Base de Conhecimento**: veja o `INSTALL.md`
  do repositório `macbtools`.
- **Trocar logo/favicon/cores**: `frontend/src/assets/logo.png`,
  `frontend/public/favicon.ico`, `frontend/src/App.js` — depois rode `npm run build`
  no frontend (dentro do servidor, em `/home/deploy/macbtools/frontend`).

## Gerenciando a instalação depois

Rode `./install_primaria` novamente (ou `./install_instancia` para adicionar uma nova
empresa/instância no mesmo servidor) e escolha uma das opções do menu:

```
[0] Instalar MacbTools
[1] Atualizar MacbTools
[2] Deletar MacbTools
[3] Bloquear MacbTools
[4] Desbloquear MacbTools
[5] Alterar domínio MacbTools
```

## Segurança

- O arquivo `config` (criado automaticamente na primeira execução) guarda as senhas
  geradas/informadas durante a instalação. Ele **não é** versionado no Git
  (está no `.gitignore`) — mantenha-o apenas no servidor, com permissão restrita
  (o próprio instalador já ajusta isso para `chmod 700`).
- Nunca reutilize, em produção, qualquer usuário/senha/token que apareça em
  documentação de terceiros ou materiais de curso — gere credenciais novas e
  exclusivas para o seu servidor.

## Estrutura deste repositório

```
install_primaria    Instalação completa do zero (primeira instância no servidor)
install_instancia   Adiciona uma nova instância a um servidor já preparado
lib/                Funções de sistema, backend, frontend e menu interativo
variables/          Variáveis padrão (timezone, cores do terminal etc.)
utils/               Utilitários (banner)
```
