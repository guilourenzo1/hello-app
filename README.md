# Projeto - CI/CD - GitHub Actions - Aplicação

Este projeto demonstra a implementação completa de um pipeline **CI/CD automatizado** utilizando **Docker**, **GitHub Actions**, **Kubernetes** e **ArgoCD**.
O pipeline foi desenvolvido para a aplicação `hello-app` (em **FastAPI**) e um repositório de manifests Kubernetes (`hello-manifests`), possibilitando **build**, **deploy** e **atualizações contínuas** sem intervenção manual.

---

## Etapas do Projeto

### Etapa 1 - Criação do repositório `hello-app`
_- Criar um repositório **público** para rodar a aplicação `hello-app`_   
_Criar uma pasta `hello-app` para abrigar o código-fonte da aplicação FastAPI e o pipeline CI/CD responsável por:_    
_- Construir a imagem Docker;_  
_- Publicar a imagem no Docker Hub;_  
_- Atualizar automaticamente o repositório `hello-manifests`;_  
_- Acionar o ArgoCD para o deploy automático no cluster Kubernetes._  

  
- Crie um `Dockerfile` para fazer a Build da imagem da aplicação  
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- Criar um arquivo `main.py` para rodar o FastAPI  
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World""}
```
• Criar um arquivo `requirements.txt` para abrigar as dependências do projeto  
```
fastapi
uvicorn[standard]
```

- Por fim, no hello-app, crie:    
└── .github/workflows/  
  └── ci-cd.yml  
Para ser feito a Pipeline CI/CD automatizada  
```yml
name: CI/CD - Build, Push e Atualiza Manifests

on:
  push:
    branches: [ "main" ]

env:
  IMAGE_NAME: SEU_USUARIO_DOCKER/hello-app

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout do código
      uses: actions/checkout@v4

    - name: Configurar Buildx
      uses: docker/setup-buildx-action@v2

    - name: Login no Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build e Push da imagem
      id: build
      uses: docker/build-push-action@v4
      with:
        push: true
        tags: ${{ env.IMAGE_NAME }}:${{ github.sha }}

    - name: Configurar chave SSH
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
        chmod 600 ~/.ssh/id_rsa
        ssh-keyscan github.com >> ~/.ssh/known_hosts

    - name: Clonar repositório de manifests
      run: |
        git clone LINK_SSH_DO_SEU_REPOSITORIO_MANIFESTS/hello-manifests.git manifests

    - name: Atualizar deployment.yaml com nova tag
      run: |
        cd manifests
        NEW_IMAGE="${{ env.IMAGE_NAME }}:${{ github.sha }}"
        echo "Atualizando imagem para: ${NEW_IMAGE}"
        sed -i "s|image: .*|image: ${NEW_IMAGE}|g" deployment.yaml
        git config user.email "actions@github.com"
        git config user.name "github-actions"
        git add deployment.yaml
        git commit -m "ci: atualiza imagem para ${NEW_IMAGE}" || echo "Nenhuma mudança para commitar"
        git pull --rebase origin main || true
        git push origin main
```

---

### Etapa 2 - Criação do repositório `hello-manifests`
_Quando o `hello-app` envia uma nova versão de imagem, o `deployment.yaml` é atualizado automaticamente com a nova tag.  
O ArgoCD detecta essa mudança e aplica o novo deploy._  
- Criar um repositório **público** que contém os manifests *Kubernetes* monitorados pelo *ArgoCD*  
- Criar uma pasta `hello-manifests` que contenha o *deployment* do hello-app no Kubernetes e a *exposição* do serviço (ClusterIP)  
- Crie `deployment.yaml` para fazer o deployment do hello-app no Kubernetes  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  labels:
    app: hello-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-app
  template:
    metadata:
      labels:
        app: hello-app
    spec:
      containers:
        - name: hello-app
          image: SEU_NOME_DOCKER_HUB/hello-app:latest  
          ports:
            - containerPort: 8000
          readinessProbe:
            httpGet:
              path: /
              port: 8000
            initialDelaySeconds: 3
            periodSeconds: 5
```

• Crie `service.yaml` para fazer a exposição do serviço (ClusterIP)  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-app-svc
spec:
  selector:
    app: hello-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8000
  type: ClusterIP
```

---

## Configuração de Chaves e Segredos (Secrets)

O pipeline utiliza **segredos criptografados** para autenticação segura entre o GitHub, Docker Hub e o ArgoCD.    
Todos os secrets são configurados no repositório **`hello-app`**, em:  
*Settings → Secrets and variables → Actions*
Abaixo estão todas as chaves necessárias e como criá-las:

---

### DOCKER_USERNAME
Usuário da sua conta no Docker Hub, utilizado pelo GitHub Actions para autenticar e publicar imagens.

1. Acesse [https://hub.docker.com](https://hub.docker.com)
2. Faça login na sua conta.
3. Crie o secret no GitHub:
- Vá em **hello-app → Settings → Secrets → Actions → New repository secret**
- Nome: `DOCKER_USERNAME`
- Valor: *seu nome de usuário do Docker Hub*

---

### DOCKER_PASSWORD
Token de acesso (Access Token) gerado no Docker Hub que substitui a senha e permite o push seguro de imagens.

1. Vá em [https://hub.docker.com/settings/security](https://hub.docker.com/settings/security)
2. Clique em **New Access Token**
3. Dê um nome (ex: `github-actions-ci`)
4. Em **Access Permissions**, escolha **Read & Write**
5. Clique em **Create**
6. Copie o token gerado (ele aparece só uma vez)
7. Crie um secret no GitHub:
   - Nome: `DOCKER_PASSWORD`
   - Valor: *cole o token do Docker Hub*

---

### SSH_PRIVATE_KEY
Chave privada usada para permitir que o GitHub Actions (no repositório `hello-app`) tenha acesso de **escrita** ao repositório `hello-manifests` via SSH.  
Isso possibilita o **push automático** das alterações no `deployment.yaml` sem precisar de token nem PR manual.

1. No seu terminal (Git Bash, PowerShell ou VS Code), gere o par de chaves:
   ```bash
   ssh-keygen -t rsa -b 4096 -C "github-actions@ci" -f gh_actions_manifest -N ""
   ```
Isso gera dois arquivos
```css
gh_actions_manifest       ← chave privada
gh_actions_manifest.pub   ← chave pública
```
2. Adicione a chave pública no (`.pub`) no repositório `hello-manifests`:  
- Vá em hello-manifests → Settings → Deploy Keys → Add Deploy Key
- Title: `GitHub Actions Deploy Key`
- Cole o conteúdo da chave pública (`gh_actions_manifest.pub`)
- Marque Allow write access
- Clique em Add key
3. Adicione a chave privada no repositório `hello-app`:  
- Vá em hello-app → Settings → Secrets → Actions → New repository secret
- Nome: `SSH_PRIVATE_KEY`
- Valor: cole todo o conteúdo da chave privada (`gh_actions_manifest`)

---
## Ferramentas Utilizadas 
| Ferramenta | Função |
|-------------|--------|
| **FastAPI** | Framework backend em Python |
| **Docker** | Containerização da aplicação |
| **GitHub Actions** | Pipeline de integração e entrega contínua |
| **Docker Hub** | Registro de imagens |
| **Kubernetes** | Orquestração de containers |
| **ArgoCD** | Entrega contínua e sincronização automática |

---

## Funcionamento do fluxo CI/CD 
1. Desenvolvimento
- O desenvolvedor altera o código em `hello-app` e faz push na branch `main`

2. CI
- O GitHub Actions faz build da imagem Docker
- Publica imagem no Docker Hub
- Atualiza a tag da imagem no `deployment.yaml` dentro do `hello-manifests`

3. CD  
- O ArgoCD detecta automaticamente a mudança no repositório `hello-manifests`
- Atualiza os pods do Kubernetes com a nova imagem
- A aplicação é redeployada automaticamente

---

### Pré-Requisitos para acessar ArgoCD
1. `Docker Desktop` com kubernets instalado
2. `kubectl` intalado e configurado
3. `ArgoCD` instalado no cluster
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

## Acessando o ArgoCD

1. Faça o port-forward:
```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```
2. Acesse no navegador
https://localhost:8080
3.Login:
- Usuário: admin
- Senha:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode
```
4. Habilite o Auto-Sync no app hello-app:
Vá em App Details → Sync Policy
- Marque:
  - Auto-Sync
  - Prune Resources
  - Self Heal
5. Acesse no navegador
https://localhost:8081
- Pronto, você verá sua aplicação funcionando!

---

## Exemplo de Atualização Automática
1. Edite o arquivo main.py:
```bash
return {"message": "Olá Mundo, novamente"}
```
2. Faça o push:
```bash
git add .
git commit -m "feat: nova mensagem"
git push
```
3. O GitHub Actions:
- Builda e publica nova imagem;
- Atualiza hello-manifests com a nova tag;
- O ArgoCD aplica o novo deploy automaticamente.

---

## Autor
*Guilherme Lourenzo*
