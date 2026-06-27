<div align="center">

<img src="https://img.shields.io/badge/Python-3.11+-blue?logo=python&logoColor=white" />
<img src="https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi&logoColor=white" />
<img src="https://img.shields.io/badge/React-18.3-61DAFB?logo=react&logoColor=black" />
<img src="https://img.shields.io/badge/YOLOv8-Ultralytics-FF5733?logo=pytorch&logoColor=white" />
<img src="https://img.shields.io/badge/Expo-54.0-000020?logo=expo&logoColor=white" />
<img src="https://img.shields.io/badge/license-MIT-green" />

# EPIsee

**Sistema inteligente de detecção de EPIs com visão computacional**

EPIsee é uma plataforma completa de segurança do trabalho que utiliza YOLOv8 para monitorar, em tempo real, o uso de Equipamentos de Proteção Individual (EPIs) em ambientes industriais. O sistema integra um backend em FastAPI, um dashboard web em React e um aplicativo móvel em React Native/Expo, oferecendo alertas automáticos, gestão de conformidade por setor e relatórios para gestores e trabalhadores.

</div>

---

## Sumário

- [Visão Geral](#visão-geral)
- [Funcionalidades](#funcionalidades)
- [Arquitetura](#arquitetura)
- [Stack Tecnológica](#stack-tecnológica)
- [Pré-requisitos](#pré-requisitos)
- [Instalação e Execução](#instalação-e-execução)
- [Variáveis de Ambiente](#variáveis-de-ambiente)
- [Estrutura do Projeto](#estrutura-do-projeto)
- [Modelo YOLO](#modelo-yolo)
- [API — Endpoints Principais](#api--endpoints-principais)
- [Deploy](#deploy)
- [Contribuição](#contribuição)
- [Licença](#licença)

---

## Visão Geral

EPIsee resolve um problema crítico em ambientes industriais: garantir que os trabalhadores estejam usando os EPIs corretos, no setor correto, de forma contínua. O sistema monitora câmeras RTSP em tempo real, detecta automaticamente os EPIs presentes na cena, registra ocorrências de não-conformidade e aciona alertas via Telegram.

**Problema:** Fiscalização manual de EPIs é ineficiente, reativa e sujeita a falhas humanas.  
**Solução:** Visão computacional contínua integrada a um sistema de gestão completo.

---

## Funcionalidades

### Detecção em Tempo Real
- Monitoramento de câmeras IP via protocolo RTSP
- Inferência com YOLOv8 customizado para 10+ classes de EPIs
- Streaming HLS para visualização ao vivo no dashboard
- Detecção por setor com EPIs obrigatórios configuráveis por gestor

### Dashboard Web (React)
- Visualização ao vivo das câmeras com player HLS
- Histórico de ocorrências com filtros por setor, data e status
- Gestão de usuários (gestor / trabalhador)
- Gestão de setores e câmeras
- Solicitações de EPI com fluxo de aprovação
- Relatórios exportáveis em PDF
- Vídeos de treinamento por tipo de EPI (NR-6)
- Notificações em tempo real

### App Mobile (React Native / Expo)
- Login e autenticação JWT
- Visualização do status de conformidade do próprio setor
- Solicitação de EPIs diretamente pelo celular
- Chatbot integrado para dúvidas sobre EPIs e NR-6
- Tela de treinamento com vídeos por EPI

### Alertas e Integrações
- Alertas automáticos via **Telegram** quando há não-conformidade
- Notificações internas no sistema
- Chatbot com IA (DeepSeek/OpenAI) para suporte a trabalhadores

---

## Arquitetura

![Arquitetura](docs/assets/arquiteturaepi.png)

O backend é o núcleo do sistema: recebe os streams RTSP, executa a inferência YOLOv8 frame a frame, persiste as ocorrências no banco de dados PostgreSQL em prod e expõe uma API REST consumida pelo dashboard e pelo app mobile.

---

## Stack Tecnológica

| Camada | Tecnologia | Versão |
|---|---|---|
| Backend | FastAPI + Uvicorn | 0.115 / 0.30 |
| ORM | SQLAlchemy (async) + Alembic | 2.0 / 1.13 |
| Banco de dados | PostgreSQL (prod) | — |
| Visão computacional | Ultralytics YOLOv8 + OpenCV | 8.3.40 / 4.10 |
| Streaming | FFmpeg + HLS | — |
| Autenticação | JWT (python-jose + passlib) | — |
| IA / Chatbot | OpenAI / DeepSeek API | — |
| Alertas | Telegram Bot API | — |
| Frontend | React 18 + Vite + Tailwind CSS | 18.3 / 5.4 / 3.4 |
| Gráficos | Recharts | 2.13 |
| PDF | jsPDF + html2canvas | — |
| Mobile | React Native + Expo | 0.81 / 54 |
| Navegação mobile | React Navigation (stack + tabs) | 6.x |
| Deploy | Railway (backend + frontend) | — |

---

## Pré-requisitos

- **Python** 3.11+
- **Node.js** 18+ e npm
- **FFmpeg** instalado no sistema (`apt install ffmpeg` / `brew install ffmpeg`)
- **Git**
- Conta no [Hugging Face](https://huggingface.co) para baixar o modelo (ou o modelo `best.pt` local)


---

## Instalação e Execução

1. Backend (Server)

```bash
cd Server

# Crie e ative o ambiente virtual
python -m venv venv
source venv/bin/activate        # Linux/macOS
# venv\Scripts\activate         # Windows

# Instale as dependências
pip install -r requirements.txt

# Configure as variáveis de ambiente
cp .env.example .env
# Edite o arquivo .env com suas configurações

# Execute as migrações do banco de dados
alembic upgrade head

# Inicie o servidor
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

O servidor estará disponível em `http://localhost:8000`.  
Documentação interativa da API (Swagger): `http://localhost:8000/docs`

> **Nota:** Na primeira inicialização, o backend baixa automaticamente o modelo `best.pt` do Hugging Face (~100MB). Para usar um modelo local, coloque o arquivo `best.pt` na pasta `Server/`.

---

### 3. Frontend (Client)

```bash
cd Client

# Instale as dependências
npm install

# Inicie em modo de desenvolvimento
npm run dev
```

O dashboard estará disponível em `http://localhost:5173`.

> Certifique-se de que a variável de API no arquivo `Client/src/api/api.js` aponta para `http://localhost:8000`.

---

### 4. App Mobile (Mobile)

```bash
cd Mobile

# Instale as dependências
npm install

# Inicie com Expo
npx expo start
```

Escaneie o QR code com o app **Expo Go** (Android/iOS) ou pressione `a` para emulador Android / `i` para iOS.

---

## Variáveis de Ambiente

Crie o arquivo `Server/.env` baseado em `Server/.env.example`:

```env
# Banco de dados
DATABASE_URL=sqlite:///./episee.db
# Para produção (PostgreSQL):
# DATABASE_URL=postgresql+asyncpg://usuario:senha@host:5432/episee

# Segurança JWT
SECRET_KEY=sua-chave-secreta-aqui
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=1440

# Inteligência Artificial (chatbot)
OPENAI_API_KEY=sk-...
DEEPSEEK_API_KEY=sk-...

# Telegram Bot (alertas)
TELEGRAM_BOT_TOKEN=seu-token-aqui
APP_URL=https://sua-url-publica.railway.app

# Usuário admin padrão (criado na primeira execução)
DEFAULT_ADMIN_EMAIL=admin@episee.com
DEFAULT_ADMIN_PASSWORD=senha-forte-aqui

# Hugging Face (download automático do modelo)
HF_TOKEN=hf_...
```

| Variável | Obrigatória | Descrição |
|---|---|---|
| `DATABASE_URL` | Sim | URL de conexão com o banco de dados |
| `SECRET_KEY` | Sim | Chave para assinatura dos tokens JWT |
| `TELEGRAM_BOT_TOKEN` | Não | Token do bot para alertas automáticos |
| `OPENAI_API_KEY` | Não | Para o chatbot com GPT |
| `DEEPSEEK_API_KEY` | Não | Alternativa ao OpenAI para o chatbot |
| `HF_TOKEN` | Não | Necessário se o repositório HF for privado |

---

## Estrutura do Projeto

```
EPIsee/
├── Server/                    # Backend FastAPI
│   ├── main.py                # Ponto de entrada da aplicação
│   ├── requirements.txt       # Dependências Python
│   ├── Dockerfile             # Containerização
│   ├── alembic/               # Migrações do banco de dados
│   │   └── versions/          # Arquivos de migração
│   ├── models/                # Pasta para o modelo best.pt
│   └── app/
│       ├── api/               # Routers FastAPI
│       │   ├── auth.py        # Autenticação (login, token)
│       │   ├── users.py       # CRUD de usuários
│       │   ├── cameras.py     # Gestão de câmeras
│       │   ├── sectors.py     # Gestão de setores
│       │   ├── detection.py   # Controle de streams de detecção
│       │   ├── occurrences.py # Histórico de ocorrências
│       │   ├── epi_requests.py# Solicitações de EPI
│       │   ├── dashboard.py   # Métricas e KPIs
│       │   ├── reports.py     # Geração de relatórios
│       │   ├── chatbot.py     # Chatbot com IA
│       │   ├── notifications.py # Notificações
│       │   ├── telegram.py    # Webhook Telegram
│       │   └── training_videos.py # Vídeos de treinamento
│       ├── core/
│       │   ├── config.py      # Configurações (Pydantic Settings)
│       │   ├── database.py    # Sessão async do banco
│       │   ├── security.py    # Hashing e JWT
│       │   ├── deps.py        # Dependências (get_current_user)
│       │   └── sector_epi_config.py # EPIs obrigatórios por setor
│       ├── models/            # Modelos SQLAlchemy (ORM)
│       │   ├── user.py
│       │   ├── sector.py
│       │   ├── camera.py
│       │   ├── occurrence.py
│       │   ├── epi_request.py
│       │   ├── notification.py
│       │   └── training_video.py
│       ├── schemas/           # Schemas Pydantic (validação)
│       └── services/          # Lógica de negócio
│           ├── detection_service_real.py  # Engine de detecção YOLOv8
│           ├── alert_service.py           # Disparador de alertas
│           ├── chatbot_service.py         # Integração IA
│           ├── telegram_service.py        # Envio de mensagens Telegram
│           └── tts_service.py             # Text-to-speech
│
├── Client/                    # Frontend React
│   ├── src/
│   │   ├── App.jsx            # Roteamento principal
│   │   ├── api/api.js         # Cliente Axios
│   │   ├── components/        # Componentes reutilizáveis
│   │   ├── pages/             # Páginas da aplicação
│   │   ├── contexts/          # AuthContext (JWT)
│   │   ├── hooks/             # useAuth
│   │   └── styles/            # CSS modular
│   ├── vite.config.js
│   └── tailwind.config.js
│
└── Mobile/                    # App React Native
    ├── App.js                 # Ponto de entrada Expo
    ├── src/
    │   ├── api/api.js         # Cliente Axios mobile
    │   ├── components/        # Componentes nativos
    │   ├── screens/           # Telas do app
    │   ├── contexts/          # AuthContext mobile
    │   ├── navigation/        # React Navigation
    │   └── hooks/             # useChatbot
    └── app.json
```

---

## Modelo YOLO

O modelo customizado (`best.pt`) foi treinado com o dataset **Construction Site Safety** e fine-tuned para o contexto de EPIs industriais. Ele detecta as seguintes classes:

| Classe | EPI |
|---|---|
| `helmet` | Capacete de segurança |
| `safety-vest` | Colete refletivo |
| `glasses` | Óculos de proteção |
| `gloves` | Luvas |
| `face-mask-medical` | Máscara facial |
| `face-guard` | Protetor facial |
| `earmuffs` | Protetor auricular |
| `medical-suit` | Macacão de proteção |
| `safety-suit` | Roupa de segurança |
| `person` | Pessoa detectada |
| `head` | Cabeça (para inferir ausência de capacete) |

O modelo está hospedado publicamente no Hugging Face: [MatsudaPaulo/episeeyolo](https://huggingface.co/MatsudaPaulo/episeeyolo).

**Parâmetros de inferência:**
- Confiança mínima: `0.45`
- Intervalo entre frames: `0.3s`
- IoU capacete/cabeça: `0.25`
- IoU colete/pessoa: `0.15`

---

## API — Endpoints Principais

A documentação completa e interativa está disponível em `/docs` (Swagger UI) após iniciar o servidor.

### Autenticação
| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/api/auth/login` | Login — retorna token JWT |
| `GET` | `/api/auth/me` | Dados do usuário autenticado |

### Detecção
| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/api/detection/start/{camera_id}` | Inicia detecção em uma câmera |
| `POST` | `/api/detection/stop/{camera_id}` | Para a detecção |
| `GET` | `/api/detection/status` | Status de todas as câmeras |
| `GET` | `/hls/{camera_id}/stream.m3u8` | Stream HLS da câmera |

### Ocorrências
| Método | Endpoint | Descrição |
|---|---|---|
| `GET` | `/api/occurrences/` | Lista ocorrências (com filtros) |
| `GET` | `/api/occurrences/{id}` | Detalhes de uma ocorrência |

### Dashboard
| Método | Endpoint | Descrição |
|---|---|---|
| `GET` | `/api/dashboard/summary` | KPIs gerais |
| `GET` | `/api/dashboard/compliance-trend` | Tendência de conformidade |

### Gestão
| Método | Endpoint | Descrição |
|---|---|---|
| `GET/POST` | `/api/sectors/` | Listar / criar setores |
| `GET/POST` | `/api/cameras/` | Listar / criar câmeras |
| `GET/POST` | `/api/users/` | Listar / criar usuários |
| `GET/POST` | `/api/epi-requests/` | Solicitações de EPI |

--------

<div align="center">
  Desenvolvido por <a href="https://github.com/PauloCMatsudaA">Paulo Matsuda</a> — TCC Engenharia de Software · Campo Real
</div>
