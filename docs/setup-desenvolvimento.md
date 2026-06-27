# Guia de Setup — Desenvolvimento Local

Este guia detalha o processo completo para rodar o EPIsee em ambiente de desenvolvimento local.

---

## Requisitos do Sistema

| Ferramenta | Versão mínima | Verificação |
|---|---|---|
| Python | 3.11 | `python --version` |
| Node.js | 18.0 | `node --version` |
| npm | 9.0 | `npm --version` |
| FFmpeg | qualquer recente | `ffmpeg -version` |
| Git | 2.x | `git --version` |

### Instalar FFmpeg

**Ubuntu/Debian:**
```bash
sudo apt update && sudo apt install ffmpeg -y
```

**macOS (Homebrew):**
```bash
brew install ffmpeg
```

**Windows:**
Baixe em [ffmpeg.org](https://ffmpeg.org/download.html) e adicione ao PATH.

---

## 1. Clone e Estrutura

```bash
git clone https://github.com/PauloCMatsudaA/EPIsee.git
cd EPIsee
```

O projeto possui três sub-projetos independentes:
```
EPIsee/
├── Server/    ← Backend Python/FastAPI
├── Client/    ← Frontend React/Vite
└── Mobile/    ← App React Native/Expo
```

---

## 2. Configurar o Backend

### 2.1 Ambiente virtual Python

```bash
cd Server

# Criar ambiente virtual
python -m venv venv

# Ativar (Linux/macOS)
source venv/bin/activate

# Ativar (Windows)
venv\Scripts\activate
```

Você verá `(venv)` no início do terminal quando estiver ativo.

### 2.2 Instalar dependências

```bash
pip install -r requirements.txt
```

> A instalação do `ultralytics` (YOLOv8) e do `opencv-python-headless` pode demorar alguns minutos.

### 2.3 Criar o arquivo `.env`

```bash
cp .env.example .env
```

Edite o `.env` com suas configurações. Para desenvolvimento local, o mínimo necessário é:

```env
DATABASE_URL=sqlite:///./episee.db
SECRET_KEY=postgresql+asyncpg://usuario:senha@host:5432/episee
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=1440
DEFAULT_ADMIN_EMAIL=admin@episee.com
DEFAULT_ADMIN_PASSWORD=admin123
```

Para usar câmeras reais e alertas, adicione:
```env
TELEGRAM_BOT_TOKEN=seu-token-aqui
OPENAI_API_KEY=sk-...
# ou
DEEPSEEK_API_KEY=sk-...
```

### 2.4 Executar migrações do banco

```bash
alembic upgrade head
```

Isso cria todas as tabelas no banco de dados SQLite.

### 2.5 Iniciar o servidor

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

O servidor sobe em `http://localhost:8000`.

**Logs esperados na primeira inicialização:**
```
INFO:     Started server process [12345]
INFO:     [MODEL] Baixando best.pt de https://huggingface.co/...
INFO:     [MODEL] Download: 100.0% (XX MB / XX MB)
INFO:     [MODEL] best.pt baixado com sucesso!
INFO:     [DB] Banco inicializado.
INFO:     [STARTUP] Admin padrão criado: admin@episee.com
INFO:     Application startup complete.
```

> Se o download do modelo falhar (rede lenta ou token HF necessário), você pode baixar manualmente de [huggingface.co/MatsudaPaulo/episeeyolo](https://huggingface.co/MatsudaPaulo/episeeyolo) e colocar o arquivo `best.pt` na pasta `Server/`.

---

## 3. Configurar o Frontend

Abra um **novo terminal** (mantenha o backend rodando).

```bash
cd Client
npm install
npm run dev
```

O dashboard abre em `http://localhost:5173`.

### Configurar URL da API

Se o backend não estiver em `localhost:8000`, edite `Client/src/api/api.js`:

```js
const API_BASE_URL = 'http://localhost:8000';  // ajuste aqui
```

---

## 4. Configurar o App Mobile

Abra outro terminal.

```bash
cd Mobile
npm install
npx expo start
```

O Expo Metro Bundler abre. Você pode:
- Escanear o QR code com **Expo Go** (Android/iOS)
- Pressionar `a` para emulador Android
- Pressionar `i` para simulador iOS (macOS apenas)

### Configurar URL da API no Mobile

Edite `Mobile/src/api/api.js`:

```js
// Use o IP da sua máquina na rede local (não localhost)
// pois o dispositivo físico não consegue acessar o localhost do seu PC
const API_BASE_URL = 'http://192.168.1.xxx:8000';
```

Para descobrir seu IP local:
```bash
# Linux/macOS
ip addr show | grep "inet "
# Windows
ipconfig
```

---

## 5. Login Padrão

Após o servidor inicializar, acesse `http://localhost:5173` e faça login com:

| Campo | Valor |
|---|---|
| Email | `admin@episee.com` |
| Senha | O valor de `DEFAULT_ADMIN_PASSWORD` no `.env` |

O usuário admin tem role `gestor` e `is_system_admin = true`, com acesso total ao sistema.

---

## 6. Testando a Detecção

### Com câmera IP real (RTSP)
1. Acesse **Câmeras** no dashboard
2. Adicione uma câmera com a URL RTSP (ex: `rtsp://admin:senha@192.168.1.100:554/stream`)
3. Vá em **Configurações** e inicie a detecção para a câmera

### Sem câmera (arquivo de vídeo)
O sistema tem fallback para arquivo de vídeo local. Coloque um arquivo `teste.mp4` na pasta `Server/` e inicie a detecção normalmente.

---

## 7. Comandos Úteis

### Backend
```bash
# Ver logs de detecção em tempo real
uvicorn main:app --reload --log-level debug

# Criar nova migração Alembic após alterar modelos
alembic revision --autogenerate -m "descricao da mudanca"

# Resetar banco de dados (CUIDADO: apaga todos os dados)
rm episee.db
alembic upgrade head
```

### Frontend
```bash
# Build de produção
npm run build

# Preview do build
npm run preview
```

### Mobile
```bash
# Limpar cache do Expo (útil quando há problemas de build)
npx expo start --clear

# Build para Android (APK)
npx expo build:android

# Build para iOS
npx expo build:ios
```

---

## 8. Solução de Problemas Comuns

### `ModuleNotFoundError` ao iniciar o servidor
```bash
# Certifique-se que o venv está ativado
source venv/bin/activate
pip install -r requirements.txt
```

### Erro de CORS no frontend
Verifique se a URL do frontend está na lista `ALLOWED_ORIGINS` em `main.py` ou adicione via variável `CORS_ORIGINS` no `.env`.

### App mobile não conecta na API
Use o IP da máquina na rede local, não `localhost`. Certifique-se que o firewall permite conexões na porta 8000.

### Modelo não baixa (timeout)
Defina `HF_TOKEN` no `.env` e verifique a conexão. Ou baixe manualmente e coloque `best.pt` em `Server/`.

### Câmera RTSP não conecta
Teste a URL com VLC (`Media → Open Network Stream`) antes de configurar no sistema. Verifique credenciais e firewall do NVR/câmera.

### FFmpeg não encontrado
```bash
# Verificar se está instalado
which ffmpeg
# Se não encontrado, instalar (Ubuntu)
sudo apt install ffmpeg
```
