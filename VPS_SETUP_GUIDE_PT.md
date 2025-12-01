# Guia de Instalação Taiga em VPS

## ⚙️ Pré-requisitos

Antes de começar, certifique-se de ter na sua VPS:

- **Docker** versão 19.03.0 ou superior
- **Docker Compose**
- **SSH** acesso à VPS
- **Nginx** (para proxy reverso e HTTPS)
- Um **domínio** apontando para sua VPS (exemplo: `taiga.seudominio.com`)

### Instalar Docker e Docker Compose na VPS

```bash
# Atualizar pacotes
sudo apt update && sudo apt upgrade -y

# Instalar Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Instalar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verificar instalação
docker --version
docker-compose --version
```

---

## 📋 Passo 1: Clonar o Repositório

Na sua VPS, execute:

```bash
cd /opt
sudo git clone https://github.com/taigaio/taiga-docker.git
cd taiga-docker
git checkout stable
```

---

## 🔧 Passo 2: Configurar o Arquivo `.env`

Edite o arquivo `.env` com suas configurações:

```bash
sudo nano .env
```

### Configurações Essenciais para Produção:

```bash
# ===== CONFIGURAÇÕES DE URL =====
TAIGA_SCHEME=https                    # Use HTTPS em produção
TAIGA_DOMAIN=taiga.seudominio.com     # Seu domínio
SUBPATH=""                            # Deixe vazio para subdomain, ou "/taiga" para subpath
WEBSOCKETS_SCHEME=wss                 # Use WSS com HTTPS

# ===== SEGURANÇA - ALTERE ISTO! =====
SECRET_KEY="seu-secret-key-muito-seguro-aleatorio-aqui"  # Gere uma chave aleatória forte!

# ===== BANCO DE DADOS =====
POSTGRES_USER=taiga_user              # Mude do padrão
POSTGRES_PASSWORD=sua-senha-super-segura  # Mude do padrão - MUITO IMPORTANTE!

# ===== EMAIL (OPCIONAL - Desenvolvimento pode usar console) =====
EMAIL_BACKEND=console                 # Para testes: console, para produção: smtp
# EMAIL_HOST=seu-smtp.com
# EMAIL_PORT=587
# EMAIL_HOST_USER=seu-email@seudominio.com
# EMAIL_HOST_PASSWORD=sua-senha-smtp
# EMAIL_DEFAULT_FROM=taiga@seudominio.com
# EMAIL_USE_TLS=True
# EMAIL_USE_SSL=False

# ===== RABBITMQ =====
RABBITMQ_USER=rabbitmq_user           # Mude do padrão
RABBITMQ_PASS=sua-senha-rabbitmq-segura  # Mude do padrão
RABBITMQ_VHOST=taiga
RABBITMQ_ERLANG_COOKIE=seu-cookie-erlang-aleatorio  # Valor aleatório único

# ===== ATTACHMENTS =====
ATTACHMENTS_MAX_AGE=360               # Tempo de expiração dos anexos (segundos)

# ===== TELEMETRIA =====
ENABLE_TELEMETRY=True                 # Enviar dados anônimos ao Taiga (opcional)
```

### Gerar Valores Aleatórios Seguros:

```bash
# Para SECRET_KEY
openssl rand -base64 32

# Para RABBITMQ_ERLANG_COOKIE
openssl rand -hex 16
```

---

## 🚀 Passo 3: Iniciar a Aplicação

```bash
# Dar permissões aos scripts
sudo chmod +x launch-taiga.sh taiga-manage.sh

# Iniciar os serviços
sudo ./launch-taiga.sh

# Aguarde alguns minutos enquanto os containers são criados e iniciados
```

---

## 👤 Passo 4: Criar Superuser (Admin)

Após a aplicação estar rodando:

```bash
sudo ./taiga-manage.sh createsuperuser
```

Siga as instruções:
- Username: (seu nome de usuário admin)
- Email: seu-email@seudominio.com
- Password: (uma senha segura)

---

## 🔐 Passo 5: Configurar Nginx (Proxy Reverso com HTTPS)

### Instalar Nginx e Certbot (Let's Encrypt)

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

### Criar configuração Nginx

```bash
sudo nano /etc/nginx/sites-available/taiga
```

Cole a configuração (para **subdomain**):

```nginx
server {
    server_name taiga.seudominio.com;
    client_max_body_size 100M;

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_redirect off;
        proxy_pass http://localhost:9000/;
    }

    # WebSockets para eventos em tempo real
    location /events {
        proxy_pass http://localhost:9000/events;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_connect_timeout 7d;
        proxy_send_timeout 7d;
        proxy_read_timeout 7d;
    }
}
```

Ou para **subpath** (taiga.seudominio.com/taiga):

```nginx
server {
    server_name seudominio.com;
    client_max_body_size 100M;

    location /taiga/ {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_redirect off;
        proxy_pass http://localhost:9000/;
    }

    location /taiga/events {
        proxy_pass http://localhost:9000/events;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_connect_timeout 7d;
        proxy_send_timeout 7d;
        proxy_read_timeout 7d;
    }
}
```

### Ativar site e testar

```bash
sudo ln -s /etc/nginx/sites-available/taiga /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### Configurar SSL com Let's Encrypt

```bash
sudo certbot --nginx -d taiga.seudominio.com
# Ou para subpath:
# sudo certbot --nginx -d seudominio.com
```

O certbot irá:
- Criar o certificado SSL
- Atualizar automaticamente a configuração Nginx
- Configurar renovação automática (a cada 90 dias)

---

## ✅ Passo 6: Verificar se Tudo Está Funcionando

```bash
# Verificar status dos containers
sudo docker compose ps

# Ver logs em tempo real
sudo docker compose logs -f

# Acessar a aplicação
# No navegador: https://taiga.seudominio.com
# Fazer login com o superuser criado anteriormente
```

---

## 📦 Gerenciamento Básico

### Parar a aplicação
```bash
sudo docker compose down
```

### Reiniciar a aplicação
```bash
sudo docker compose restart
```

### Ver logs de um serviço específico
```bash
sudo docker compose logs taiga-back
sudo docker compose logs taiga-front
sudo docker compose logs postgres
```

### Executar comando no banco de dados
```bash
sudo ./taiga-manage.sh dbshell
```

### Fazer backup do banco de dados
```bash
sudo docker compose exec postgres pg_dump -U taiga_user taiga > backup_taiga_$(date +%Y%m%d_%H%M%S).sql
```

### Restaurar backup
```bash
sudo docker compose exec -T postgres psql -U taiga_user taiga < seu_backup.sql
```

---

## 🔄 Atualizar Taiga

```bash
cd /opt/taiga-docker
git pull origin stable
sudo docker compose down
sudo docker compose pull
sudo docker compose -f docker-compose.yml -f docker-compose-inits.yml up -d
```

---

## 🚨 Troubleshooting

### Problema: "Connection refused" ao acessar
- Certifique-se de que todos os containers estão rodando: `sudo docker compose ps`
- Verifique se o Nginx está ativo: `sudo systemctl status nginx`
- Aguarde alguns minutos para os containers iniciarem completamente

### Problema: Certificado SSL não funciona
```bash
sudo certbot renew --dry-run  # Testar renovação
sudo certbot certificates      # Listar certificados
```

### Problema: Email não está sendo enviado
- Verifique os logs: `sudo docker compose logs taiga-back | grep -i email`
- Se usando `EMAIL_BACKEND=console`, os emails aparecerão nos logs
- Para SMTP, teste as credenciais

### Problema: Permissão negada
- Execute comandos Docker com `sudo` ou adicione seu usuário ao grupo docker:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## 🎯 Próximos Passos (Opcional)

1. **Habilitar Registro Público**: Adicione ao `.env`
   ```bash
   PUBLIC_REGISTER_ENABLED=true
   ```

2. **Integração com GitHub OAuth**:
   - Crie OAuth App em: https://github.com/settings/developers
   - Adicione ao `.env` (ver seção "Customization" do manual)

3. **Integração com GitLab OAuth**: Similar ao GitHub

4. **Slack Integration**: Configure no `.env` conforme necessário

5. **Envio de Emails**: Configure SMTP quando necessário

---

## 📚 Documentação Oficial

- Manual oficial: https://community.taiga.io/t/taiga-30min-setup/170
- Repositório: https://github.com/taigaio/taiga-docker
- Documentação: https://docs.taiga.io/

---

## 💡 Dicas de Segurança

✅ **SEMPRE:**
- Use HTTPS em produção
- Mude TODAS as senhas padrão (SECRET_KEY, POSTGRES_PASSWORD, RABBITMQ_PASS)
- Mantenha o Docker e sistema operacional atualizados
- Faça backups regulares do banco de dados
- Configure um firewall na VPS
- Use senhas fortes e únicas

❌ **NUNCA:**
- Deixe EMAIL_BACKEND=console em produção
- Exponha o banco de dados para a internet
- Use HTTP em produção
- Compartilhe seu SECRET_KEY

---

## 🆘 Suporte

Se tiver problemas:
1. Consulte a comunidade: https://community.taiga.io/
2. Verifique os logs: `sudo docker compose logs`
3. Leia a documentação oficial completa
4. Abra uma issue no GitHub

---

**Boa sorte na instalação do seu Taiga! 🚀**
