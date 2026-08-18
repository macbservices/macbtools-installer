# Teste local com VirtualBox + Cloudflare Tunnel

Este guia cobre como testar o MacbTools numa máquina virtual local, usando seu próprio
domínio via Cloudflare Tunnel — sem precisar de IP público, port forward no roteador,
nem certificado Let's Encrypt (o Cloudflare cuida do HTTPS na borda).

> Este guia já incorpora as correções de dois problemas reais encontrados durante os
> primeiros testes (subdomínio de dois níveis sem SSL grátis, e conflito de arquivo de
> configuração do `cloudflared`). Seguindo na ordem abaixo, você não deve encontrá-los.

## 1. Criando a VM no VirtualBox

| Item | Valor |
|---|---|
| Sistema | Ubuntu Server 22.04 LTS (ISO oficial "Server") |
| vCPUs | 2 (mínimo), 4 se possível — com 1 vCPU o build do frontend funciona, só demora bem mais (20-30 min) |
| RAM | 4 GB mínimo (o build do frontend React é pesado) |
| Disco | 40 GB dinâmico — confirme depois da instalação com `df -h /`; se aparecer bem menor que 40GB, rode `lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv && resize2fs /dev/ubuntu-vg/ubuntu-lv` |
| Rede | **NAT** é suficiente — o Cloudflare Tunnel só faz conexões de *saída*, não precisa de porta aberta nem IP público. Se quiser acessar a VM por SSH direto do seu PC pela rede local, use Bridged Adapter; se ficar em NAT, configure uma regra de Port Forwarding (host `2222` → guest `22`) para o SSH. |

No instalador do Ubuntu, marque para instalar o **OpenSSH server**, e faça login como
usuário com acesso root habilitado (ou já direto como root).

## 2. Pré-requisitos no Cloudflare

1. Seu domínio precisa estar **adicionado e ativo no Cloudflare** (nameservers
   apontando pro Cloudflare — se você comprou o domínio em outro lugar, é só trocar os
   nameservers, o Cloudflare te guia nisso ao adicionar o site).
2. Não crie nenhum registro DNS manualmente para os subdomínios de teste — o
   `cloudflared` faz isso sozinho no passo 5.

### ⚠️ Importante: escolha dos nomes de subdomínio

O certificado SSL grátis do Cloudflare (Universal SSL) só cobre o domínio raiz e
subdomínios de **um nível** (ex: `app.seudominio.com`). Subdomínios de **dois níveis**
(ex: `api.app.seudominio.com`) não são cobertos e dão erro de certificado
(`ERR_SSL_VERSION_OR_CIPHER_MISMATCH`), a menos que você pague pelo Advanced
Certificate Manager.

**Solução:** use sempre subdomínios de um nível só, separados por hífen — nunca por
ponto. Neste guia usamos como exemplo:
- Painel: `macbtools.seudominio.com`
- API: `api-macbtools.seudominio.com` (hífen, não `api.macbtools.seudominio.com`)

## 3. Instalando o cloudflared na VM

Conecte na VM por SSH e rode:

```bash
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
cloudflared --version
```

## 4. Autenticando e criando o túnel

```bash
cloudflared tunnel login
```
Isso imprime uma URL no terminal. Copie e cole no navegador do Windows, faça login no
Cloudflare e selecione o seu domínio. Depois disso, crie o túnel:

```bash
cloudflared tunnel create macbtools
```
Anote o **UUID** do túnel que aparece na saída (algo como
`Created tunnel macbtools with id 6ff42ae2-765d-...`).

## 5. Configurando as rotas (ingress)

### ⚠️ Importante: use direto o arquivo em `/etc/cloudflared/`

O comando `cloudflared service install` (passo 6) lê a configuração de
`/etc/cloudflared/config.yml`, **não** de `~/.cloudflared/config.yml` (que é só onde
ficam as credenciais de login). Editar o arquivo errado faz o serviço rodar com uma
configuração desatualizada mesmo depois de reiniciado — foi exatamente isso que nos
custou boas horas de diagnóstico na primeira instalação. Para evitar esse problema,
crie o arquivo direto no lugar certo desde o início:

```bash
mkdir -p /etc/cloudflared
nano /etc/cloudflared/config.yml
```

Cole (troque `SEU_UUID`, `seudominio.com`, e as portas pelas que você vai usar na
instalação — as mesmas que você vai informar no instalador, ex: `3000` e `4000`):

```yaml
tunnel: SEU_UUID
credentials-file: /root/.cloudflared/SEU_UUID.json

ingress:
  - hostname: macbtools.seudominio.com
    service: http://localhost:3000
  - hostname: api-macbtools.seudominio.com
    service: http://localhost:4000
  - service: http_status:404
```

Crie os registros DNS automaticamente:
```bash
cloudflared tunnel route dns macbtools macbtools.seudominio.com
cloudflared tunnel route dns macbtools api-macbtools.seudominio.com
```

## 6. Rodando o túnel como serviço

```bash
cloudflared --config /etc/cloudflared/config.yml service install
systemctl start cloudflared
systemctl enable cloudflared
systemctl status cloudflared
```
Confirme que aparece `active (running)` em verde.

**Teste antes de continuar** (evita perder tempo depurando a instalação do zero se o
túnel não estiver 100%):
```bash
curl -i -X OPTIONS https://api-macbtools.seudominio.com/ \
  -H "Origin: https://macbtools.seudominio.com" \
  -H "Access-Control-Request-Method: GET"
```
Se vier `HTTP/2 404` com o header `x-powered-by: Express` presente, o túnel está
repassando corretamente (o 404 em si é normal — só existe backend nas rotas `/auth`,
`/tickets` etc., não na raiz `/`). Se não aparecer `x-powered-by: Express` nenhuma,
pare aqui e revise o `config.yml` e os registros DNS antes de seguir.

## 7. Instalando o MacbTools (variante local, sem Certbot)

```bash
cd /root
git clone https://github.com/macbservices/macbtools-installer.git
cd macbtools-installer
chmod +x install_local install_instancia
./install_local
```

Responda às perguntas normalmente (veja a tabela completa no `README.md` deste
repositório), com estas diferenças:

- **Domínio do FRONTEND**: `macbtools.seudominio.com` (o mesmo do `config.yml`)
- **Domínio do BACKEND**: `api-macbtools.seudominio.com` (com hífen, não ponto)
- **Porta do FRONTEND** e **Porta do BACKEND**: exatamente as mesmas do `config.yml`

O `install_local` pula as etapas de Certbot — tudo o mais é idêntico ao instalador de
produção (`install_primaria`).

## 8. Testando

Acesse `https://macbtools.seudominio.com` no navegador do Windows (`Ctrl+Shift+R` para
ignorar cache no primeiro acesso). O Cloudflare vai servir com HTTPS válido
automaticamente, mesmo a VM estando na sua rede local.

Login padrão criado pelo seed do banco: `admin@admin.com` / `123456` — troque a senha
assim que entrar.

## Se decidir manter essa VM como ferramenta definitiva (não só teste)

Funciona perfeitamente como instalação "de verdade" — é exatamente esse o ponto forte
do Cloudflare Tunnel. Só lembre de:
- Configurar a VM para iniciar automaticamente com o host, ou migrar para um servidor
  sempre ligado
- Fazer backup periódico do banco de dados Postgres (`pg_dump`), já que tudo fica
  local na sua máquina
- Se quiser SSL "de verdade" também nos subdomínios de dois níveis futuramente
  (ex: para múltiplos clientes SaaS), considere o Total TLS do Cloudflare (grátis em
  algumas contas, ou o Advanced Certificate Manager pago em outras)
