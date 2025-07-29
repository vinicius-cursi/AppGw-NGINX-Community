# Configuração de Cluster NGINX HA com Balanceamento Avançado

Este documento detalha a configuração completa para um cluster de NGINX em Alta Disponibilidade (HA) no modelo Ativo/Passivo, utilizando **Keepalived** para failover automático.

A configuração inclui regras avançadas de balanceamento de carga:
* **Persistência de Sessão**: Garante que um usuário seja direcionado sempre para o mesmo servidor backend.
* **Balanceamento por Peso**: Distribui mais tráfego para os servidores das aplicações mais críticas.
* **Health Checks Passivos**: Remove automaticamente um servidor do balanceamento após 5 falhas de conexão e o adiciona de volta quando o serviço se normaliza.

---

## Diagrama da Arquitetura

O diagrama abaixo ilustra o fluxo de tráfego, desde o cliente até as aplicações backend, passando pelo cluster NGINX HA.

```mermaid
graph TD
    subgraph "Rede Externa"
        A[Cliente Browser] -- DNS aponta para o VIP --> B((🌐<br>IP Virtual / VIP<br>192.168.1.10))
    end

    subgraph "Rede Corporativa / VPN"
        ADMIN[💻 Administrador] -- Acesso Seguro --> B
    end

    subgraph "Cluster NGINX HA (Ativo/Passivo)"
        style NGINX2 fill:#f9f,stroke:#333,stroke-width:2px,stroke-dasharray: 5 5

        subgraph "Servidor Primário"
            NGINX1(🟢 NGINX-01<br><b>Estado: MASTER</b>)
        end
        subgraph "Servidor Secundário"
            NGINX2(🟡 NGINX-02<br><b>Estado: BACKUP</b>)
        end

        B -- Fluxo de Tráfego Público --> NGINX1
        ADMIN -- Acesso à /nginx_status --> NGINX1
        NGINX1 <.-> |VRRP Heartbeat<br>via Keepalived| NGINX2
    end

    subgraph "Rede Interna (Servidores Backend)"
        C1(Upstream: app01_backend)
        C2(Upstream: app02_backend)
        C3(Upstream: app03_backend)
        C4(Upstream: app04_backend)
        C5(Upstream: app05_backend)
        C6(Upstream: lbprobe_backend)
    end

    NGINX1 -- proxy_pass (/app01) --> C1
    NGINX1 -- proxy_pass (/app02) --> C2
    NGINX1 -- proxy_pass (/app03) --> C3
    NGINX1 -- proxy_pass (/app04) --> C4
    NGINX1 -- proxy_pass (/app05) --> C5
    NGINX1 -- proxy_pass (/lbprobe) --> C6
````

-----

## Passo 1: Configuração do NGINX
Este arquivo deve ser **idêntico em ambos os servidores NGINX** (NGINX-01 e NGINX-02). Ele contém a lógica de roteamento, balanceamento e os health checks passivos.

**Arquivo:** `/etc/nginx/conf.d/mydomain.conf`

```nginx
# --- Upstreams (Pools de Servidores Backend) ---
#
# Diretivas utilizadas:
# least_conn: Envia novas requisições para o servidor com o menor número de conexões ativas.
# sticky cookie: Garante que um cliente permaneça no mesmo servidor durante sua sessão (requer módulo extra).
# max_fails=5: Marca um servidor como indisponível após 5 tentativas de conexão falhas.
# fail_timeout=30s: Após ser marcado como indisponível, o NGINX não tentará se conectar a ele por 30 segundos.
#                   Depois desse tempo, ele volta a receber tráfego gradualmente.
# weight: Define a prioridade de tráfego para cada aplicação.

upstream app01_backend {
    least_conn;
    sticky cookie srv_id expires=1h domain=.mydomain.com path=/app01/;
    server 192.168.1.101:5000 weight=4 max_fails=5 fail_timeout=30s;
    server 192.168.1.102:5000 weight=4 max_fails=5 fail_timeout=30s;
    server 192.168.1.103:5000 weight=4 max_fails=5 fail_timeout=30s;
    server 192.168.1.104:5000 weight=4 max_fails=5 fail_timeout=30s;
}

upstream app02_backend {
    least_conn;
    sticky cookie srv_id expires=1h domain=.mydomain.com path=/app02/;
    server 192.168.1.101:5001 weight=4 max_fails=5 fail_timeout=30s;
    server 192.168.1.102:5001 weight=4 max_fails=5 fail_timeout=30s;
    server 192.168.1.103:5001 weight=4 max_fails=5 fail_timeout=30s;
    server 192.168.1.104:5001 weight=4 max_fails=5 fail_timeout=30s;
}

upstream app03_backend {
    least_conn;
    sticky cookie srv_id expires=1h domain=.mydomain.com path=/app03/;
    server 192.168.1.101:5002 weight=3 max_fails=5 fail_timeout=30s;
    server 192.168.1.102:5002 weight=3 max_fails=5 fail_timeout=30s;
    server 192.168.1.103:5002 weight=3 max_fails=5 fail_timeout=30s;
    server 192.168.1.104:5002 weight=3 max_fails=5 fail_timeout=30s;
}

upstream app04_backend {
    least_conn;
    sticky cookie srv_id expires=1h domain=.mydomain.com path=/app04/;
    server 192.168.1.101:5003 weight=2 max_fails=5 fail_timeout=30s;
    server 192.168.1.102:5003 weight=2 max_fails=5 fail_timeout=30s;
    server 192.168.1.103:5003 weight=2 max_fails=5 fail_timeout=30s;
    server 192.168.1.104:5003 weight=2 max_fails=5 fail_timeout=30s;
}

upstream app05_backend {
    least_conn;
    sticky cookie srv_id expires=1h domain=.mydomain.com path=/app05/;
    server 192.168.1.101:5004 weight=1 max_fails=5 fail_timeout=30s;
    server 192.168.1.102:5004 weight=1 max_fails=5 fail_timeout=30s;
    server 192.168.1.103:5004 weight=1 max_fails=5 fail_timeout=30s;
    server 192.168.1.104:5004 weight=1 max_fails=5 fail_timeout=30s;
}

upstream lbprobe_backend {
    least_conn;
    server 192.168.1.101:5006;
    server 192.168.1.102:5006;
    server 192.168.1.103:5006;
    server 192.168.1.104:5006;
}

# --- Bloco do Servidor ---
server {
    listen 80;
    server_name www.mydomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name www.mydomain.com;

    ssl_certificate /etc/nginx/ssl/mydomain.pem;
    ssl_certificate_key /etc/nginx/ssl/mydomain.pem;
    ssl_password_file /etc/nginx/ssl/ssl_pass.txt;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # --- Roteamento das Aplicações ---
    location /app01/ { proxy_pass http://app01_backend; }
    location /app02/ { proxy_pass http://app02_backend; }
    location /app03/ { proxy_pass http://app03_backend; }
    location /app04/ { proxy_pass http://app04_backend; }
    location /app05/ { proxy_pass http://app05_backend; }
    location /lbprobe/ { proxy_pass http://lbprobe_backend; }

    # --- Bloco para a Página de Status ---
    location /nginx_status {
        stub_status;
        allow 10.0.0.0/8;
        allow 192.168.0.5;
        deny all;
    }
}
```

-----

## Passo 2: Configuração do Keepalived para HA

Instale o Keepalived em ambos os servidores NGINX (`sudo apt install keepalived` ou `sudo yum install keepalived`).

#### **1. Script de Verificação do NGINX**

Crie este script em **ambos os servidores**. O Keepalived o usará para garantir que o processo do NGINX está ativo. Se não estiver, ele iniciará o failover.

Este conteúdo deve ser salvo no arquivo `/etc/keepalived/check_nginx.sh` em **ambos** os servidores. Depois, torne-o executável com `sudo chmod +x /etc/keepalived/check_nginx.sh`.

```bash
#!/bin/bash
# Verifica se o processo do NGINX existe
if pgrep nginx > /dev/null; then
    exit 0 # Sucesso, NGINX está rodando
else
    exit 1 # Falha, NGINX não está rodando
fi
```

Torne o script executável: `sudo chmod +x /etc/keepalived/check_nginx.sh`

-----

### **2. Configuração do Keepalived - Servidor NGINX-01 (MASTER)**

Este conteúdo deve ser salvo no arquivo `/etc/keepalived/keepalived.conf` apenas no **servidor NGINX-01 (MASTER)**.

**Arquivo:** `/etc/keepalived/keepalived.conf`

```nginx
# Bloco global
global_defs {
   router_id NGINX_MASTER
}

# Script para monitorar o NGINX
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2  # Executar a cada 2 segundos
    weight 50   # Adicionar 50 à prioridade se o script for bem-sucedido
}

# Instância VRRP para gerenciar o IP Virtual
vrrp_instance VI_1 {
    state MASTER                # Este é o servidor PRIMÁRIO
    interface ens160            # IMPORTANTE: Altere para sua interface de rede (ex: eth0)
    virtual_router_id 51        # Deve ser o mesmo em ambos os servidores
    priority 150                # A prioridade mais alta se torna MASTER

    advert_int 1                # Anúncios VRRP a cada 1 segundo

    authentication {
        auth_type PASS
        auth_pass sua_senha_secreta # Altere e use a mesma senha nos dois servidores
    }

    # O IP Virtual (VIP) que será compartilhado
    virtual_ipaddress {
        192.168.1.10/24         # IMPORTANTE: Altere para o VIP desejado
    }

    # Associar o script de monitoramento
    track_script {
        check_nginx
    }
}
```

-----

### **3. Configuração do Keepalived - Servidor NGINX-02 (BACKUP)**

Este conteúdo deve ser salvo no arquivo `/etc/keepalived/keepalived.conf` apenas no **servidor NGINX-02 (BACKUP)**.

**Arquivo:** `/etc/keepalived/keepalived.conf`

```nginx
# Bloco global
global_defs {
   router_id NGINX_BACKUP
}

# Script para monitorar o NGINX (idêntico ao MASTER)
vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 50
}

# Instância VRRP para gerenciar o IP Virtual
vrrp_instance VI_1 {
    state BACKUP                # Este é o servidor SECUNDÁRIO
    interface ens160            # IMPORTANTE: Deve ser a mesma interface do MASTER
    virtual_router_id 51        # Deve ser o mesmo do MASTER
    priority 100                # Prioridade menor o torna um BACKUP

    advert_int 1

    authentication {
        auth_type PASS
        auth_pass sua_senha_secreta # Deve ser a mesma senha do MASTER
    }

    # O mesmo IP Virtual do MASTER
    virtual_ipaddress {
        192.168.1.10/24
    }

    track_script {
        check_nginx
    }
}
```

Após criar os arquivos, inicie e habilite o serviço Keepalived em ambos os servidores:

```bash
sudo systemctl start keepalived
sudo systemctl enable keepalived
```
