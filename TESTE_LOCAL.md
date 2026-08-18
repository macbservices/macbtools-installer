# Teste local com VirtualBox + Cloudflare Tunnel

Este guia cobre como testar o MacbTools numa máquina virtual local, usando seu próprio
domínio via Cloudflare Tunnel — sem precisar de IP público, port forward no roteador,
nem certificado Let's Encrypt (o Cloudflare cuida do HTTPS na borda).

## 1. Criando a VM no VirtualBox

| Item | Valor |
|---|---|
| Sistema | Ubuntu Server 22.04 LTS (ISO oficial "Server") |
| vCPUs | 2 (mínimo), 4 se possível |
| RAM | 4 GB mínimo (o build do frontend React é pesado) |
| Disco | 40 GB dinâmico |
| Rede | **NAT** é suficiente — o Cloudflare Tunnel só faz conexões de *saída*, não precisa de porta aberta nem IP público. Se quiser acessar a VM por SSH direto do seu PC pela rede local, use Bridged Adapter; se ficar em NAT, configure uma regra de Port Forwarding (host `2222` → guest `22`) para o SSH. |

No instalador do Ubuntu, marque para instalar o **OpenSSH server**.

## 2. Pré-requisitos no Cloudflare

1. Seu domínio precisa estar **adicionado e ativo no Cloudflare** (nameservers
   apontando pro Cloudflare — se você comprou o domínio em outro lugar, é só trocar os
   nameservers, o Cloudflare te guia nisso ao adicionar o site).
2. Não crie nenhum registro DNS manualmente para os subdomínios de teste — o
   `cloudflared` faz isso sozinho no passo 4.

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

Crie o arquivo de configuração:
```bash
nano ~/.cloudflared/config.yml
```

Cole (troque `SEU_UUID`, `seudominio.com`, e as portas pelas que você vai usar na
instalação — as mesmas que você vai informar no instalador, ex: `3000` e `4000`):

```yaml
tunnel: SEU_UUID
credentials-file: /root/.cloudflared/SEU_UUID.json

ingress:
  - hostname: app.seudominio.com
    service: http://localhost:3000
  - hostname: api.seudominio.com
    service: http://localhost:4000
  - service: http_status:404
```

Crie os registros DNS automaticamente:
```bash
cloudflared tunnel route dns macbtools app.seudominio.com
cloudflared tunnel route dns macbtools api.seudominio.com
```

## 6. Rodando o túnel

Para testar rapidamente (roda em primeiro plano, para com Ctrl+C):
```bash
cloudflared tunnel run macbtools
```

Para deixar rodando permanentemente como serviço (recomendado, inclusive se você
decidir manter essa VM local como ferramenta definitiva):
```bash
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```

## 7. Instalando o MacbTools (variante local, sem Certbot)

Em outro terminal SSH (deixe o `cloudflared tunnel run` rodando no primeiro, se estiver
testando em primeiro plano — ou pule isso se já instalou como serviço):

```bash
cd /root
git clone https://github.com/macbservices/macbtools-installer.git
cd macbtools-installer
chmod +x install_local install_instancia
./install_local
```

Responda às perguntas normalmente (veja a tabela completa no `README.md` deste
repositório), com estas diferenças:

- **Domínio do FRONTEND**: `app.seudominio.com` (o mesmo que você configurou no
  `config.yml`)
- **Domínio do BACKEND**: `api.seudominio.com`
- **Porta do FRONTEND** e **Porta do BACKEND**: use exatamente as mesmas portas que
  você já colocou no `config.yml` (ex: `3000` e `4000`)

O `install_local` pula as etapas de Certbot — tudo o mais é idêntico ao instalador de
produção (`install_primaria`).

## 8. Testando

Acesse `https://app.seudominio.com` no navegador do Windows. O Cloudflare vai servir
com HTTPS válido automaticamente, mesmo a VM estando na sua rede local.

## Se decidir manter essa VM como ferramenta definitiva (não só teste)

Funciona perfeitamente como instalação "de verdade" — é exatamente esse o ponto forte
do Cloudflare Tunnel. Só lembre de:
- Configurar `cloudflared` como serviço (passo 6) para sobreviver a reboots
- Configurar a VM para iniciar automaticamente com o VirtualBox/host, ou migrar para
  um servidor sempre ligado
- Fazer backup periódico do banco de dados Postgres (`pg_dump`), já que tudo fica
  local na sua máquina
