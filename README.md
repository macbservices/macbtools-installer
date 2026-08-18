# MacbTools — Instalador

Script de instalação automatizada do MacbTools em uma VPS **Ubuntu Server 22.04**, com
usuário **root** habilitado. Instala e configura tudo do zero: Node.js 20, PostgreSQL,
Redis (via Docker), Nginx, SSL (Certbot/Let's Encrypt), backend e frontend.

Este repositório instala o código-fonte do repositório
[`macbtools`](https://github.com/macbservices/macbtools) — você vai informar a URL desse
repositório durante a instalação.

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
em um repositório privado. Se for privado, gere um Personal Access Token
(GitHub → Settings → Developer settings → Personal access tokens) para poder cloná-lo
durante a instalação.

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
| Link do GitHub do MacbTools que deseja instalar     | URL do repositório `macbtools` (com token, se privado — veja abaixo)   |
| Nome para a Instância/Empresa                       | `macbtools` (minúsculo, sem espaços/acentos)                          |
| Qtde de Conexões/Whats                              | Ex: `10`                                                                |
| Qtde de Usuários/Atendentes                         | Ex: `10`                                                                |
| Domínio do FRONTEND/PAINEL                          | `app.seudominio.com` (sem `https://`)                                  |
| Domínio do BACKEND/API                              | `api.seudominio.com` (sem `https://`)                                  |
| Porta do FRONTEND                                    | Um valor entre `3000` e `3999`, ex: `3000`                            |
| Porta do BACKEND                                     | Um valor entre `4000` e `4999`, ex: `4000`                            |
| Porta do REDIS                                       | Um valor entre `5000` e `5999`, ex: `5000`                            |

**Formato da URL do repositório**, se ele for privado:
```
https://macbservices:SEU_TOKEN@github.com/macbservices/macbtools.git
```

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
