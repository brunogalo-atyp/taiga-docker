# Configurar CI/CD Automático para Taiga

## 📋 O que este workflow faz?

Quando você faz `push` na branch `stable` ou `main`:

1. ✅ **Build** - Compila as imagens Docker (taiga-back, taiga-front, taiga-gateway)
2. 📦 **Push** - Envia as imagens para seu Docker Hub
3. 🚀 **Deploy** - Conecta na VPS via SSH e faz o deploy automático
4. 🔔 **Notificação** - Envia resultado no Slack (opcional)

---

## 🔐 Configurar Secrets no GitHub

Você precisa adicionar os seguintes secrets no repositório GitHub:

### 1. Acessar Settings do Repositório

1. Vá para: `https://github.com/seu-usuario/taiga-docker/settings/secrets/actions`
2. Clique em **"New repository secret"**

### 2. Adicionar Secrets Necessários

#### 🐳 Docker Hub
```
Nome: DOCKER_USERNAME
Valor: seu-usuario-docker-hub
```

```
Nome: DOCKER_PASSWORD
Valor: seu-token-acesso-docker-hub
```

> 💡 Para gerar token Docker Hub: https://hub.docker.com/settings/security

#### 🖥️ VPS SSH
```
Nome: VPS_HOST
Valor: seu-ip-vps ou seu-dominio.com
Exemplo: 123.45.67.89
```

```
Nome: VPS_USER
Valor: seu-usuario-ssh-na-vps
Exemplo: ubuntu ou root
```

```
Nome: VPS_PORT
Valor: porta-ssh-na-vps
Exemplo: 22 (padrão)
```

```
Nome: VPS_SSH_KEY
Valor: sua-chave-privada-ssh
```

> ⚠️ **IMPORTANTE**: Para gerar a chave SSH:
> 
> **No seu computador local:**
> ```bash
> ssh-keygen -t ed25519 -f taiga_deploy_key -N ""
> cat taiga_deploy_key
> ```
> 
> **Copie todo o conteúdo da chave PRIVADA** para o secret `VPS_SSH_KEY`
>
> **Na VPS:**
> ```bash
> mkdir -p ~/.ssh
> echo "sua-chave-publica" >> ~/.ssh/authorized_keys
> chmod 600 ~/.ssh/authorized_keys
> chmod 700 ~/.ssh
> ```

#### 📢 Slack (Opcional)
```
Nome: SLACK_WEBHOOK_URL
Valor: sua-webhook-url-slack
```

> 💡 Para criar webhook Slack: https://api.slack.com/messaging/webhooks

---

## 📝 Como usar o workflow

### Opção 1: Push na branch `stable` (Recomendado)
```bash
git add .
git commit -m "Atualização do Taiga"
git push origin stable
```

O workflow será acionado automaticamente!

### Opção 2: Disparo manual
1. Vá para: `https://github.com/seu-usuario/taiga-docker/actions`
2. Selecione **"Build and Deploy Taiga to VPS"**
3. Clique em **"Run workflow"**
4. Escolha a branch: `stable` ou `main`
5. Clique em **"Run workflow"**

---

## 🔍 Acompanhar o Deployment

### Ver logs em tempo real
1. Vá para: `https://github.com/seu-usuario/taiga-docker/actions`
2. Clique no workflow em execução
3. Clique em **"build"** ou **"deploy"** para ver os logs
4. Acompanhe o progresso em tempo real

### Verificar sucesso
- Você verá um ✅ verde quando o deployment for bem-sucedido
- Uma notificação será enviada ao Slack (se configurado)
- A aplicação estará rodando na VPS

### Solucionar problemas
Se algo der errado:
1. Clique no job que falhou
2. Expanda os logs para ver o erro
3. Causas comuns:
   - Credenciais Docker Hub incorretas
   - SSH key inválida ou sem permissões
   - Repositório não existe no Docker Hub
   - Porta SSH incorreta

---

## 🛠️ Personalizar o Workflow

### Adicionar mais validações
Edite `.github/workflows/docker-build-deploy.yml`:

```yaml
- name: Health Check
  run: |
    curl -f http://localhost:9000 || exit 1
    echo "✓ Taiga is responding"
```

### Fazer backup antes do deploy
```yaml
- name: Backup Database
  script: |
    docker compose exec -T postgres pg_dump -U taiga_user taiga > backup_$(date +%Y%m%d_%H%M%S).sql
```

### Deploy apenas em branch específica
Edite a seção `on`:
```yaml
on:
  push:
    branches:
      - stable  # Apenas branch stable faz deploy
  workflow_dispatch:
```

### Adicionar aprovação antes do deploy
```yaml
deploy:
  needs: build
  environment: production  # Requer aprovação manual
```

---

## 🔔 Configurar Notificações Slack

### Passo 1: Criar App no Slack
1. Acesse: https://api.slack.com/apps
2. Clique em **"Create New App"**
3. Escolha **"From scratch"**
4. Nome: `Taiga CI/CD`
5. Workspace: seu workspace Slack

### Passo 2: Habilitar Webhooks
1. Na lateral, clique em **"Incoming Webhooks"**
2. Ative **"Incoming Webhooks"**
3. Clique em **"Add New Webhook to Workspace"**
4. Selecione o canal (ex: #deployments)
5. Autorize

### Passo 3: Copiar URL
1. Copie a **Webhook URL** gerada
2. Adicione como secret `SLACK_WEBHOOK_URL` no GitHub

---

## 📊 Workflow Diagram

```
Push to stable/main
        ↓
   ┌────────────────┐
   │  BUILD STAGE   │
   │                │
   │ • Checkout     │
   │ • Build Back   │
   │ • Build Front  │
   │ • Build Gateway│
   │ • Push images  │
   └────────┬───────┘
            ↓
   ┌────────────────┐
   │ DEPLOY STAGE   │
   │                │
   │ • SSH to VPS   │
   │ • Git pull     │
   │ • Docker pull  │
   │ • Restart svc  │
   │ • Verify       │
   └────────┬───────┘
            ↓
   ┌────────────────┐
   │   NOTIFY       │
   │                │
   │ • Slack        │
   │ • Email        │
   └────────────────┘
```

---

## ⚠️ Segurança e Boas Práticas

✅ **FAÇA:**
- Guarde as SSH keys em local seguro
- Revogue secrets se vazarem
- Use tokens com expiração no Docker Hub
- Limite permissões SSH (específicas ao usuário deploy)
- Faça backups antes de cada deploy

❌ **NUNCA:**
- Commite secrets no repositório
- Compartilhe chaves SSH
- Use conta root para deploy
- Deixe credenciais em logs públicos

---

## 🆘 Troubleshooting

### Erro: "Permission denied (publickey)"
- Verifique se a chave SSH pública está em `~/.ssh/authorized_keys` na VPS
- Teste manualmente: `ssh -i taiga_deploy_key user@vps_ip`

### Erro: "Docker pull failed"
- Verifique credenciais Docker Hub
- Confirme que as imagens existem no repositório

### Erro: "Connection timeout"
- Verifique IP da VPS
- Confirme porta SSH
- Teste firewall: `telnet vps_ip porta`

### Deployment rápido demais?
- Aumente o `sleep 30` no script de deploy
- Adicione verificações de health check

---

## 📚 Documentação Oficial

- GitHub Actions: https://docs.github.com/en/actions
- Docker Hub API: https://docs.docker.com/docker-hub/api/
- Slack Webhooks: https://api.slack.com/messaging/webhooks

---

**Seu Taiga agora tem CI/CD automático! 🚀**
