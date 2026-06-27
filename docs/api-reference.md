# Referência da API

A API segue o padrão REST com payloads JSON. Todos os endpoints (exceto `/api/auth/login`) exigem o header:

```
Authorization: Bearer <access_token>
```

A documentação interativa (Swagger UI) está disponível em `/docs` após iniciar o servidor.

---

## Autenticação

### `POST /api/auth/login`
Autentica o usuário e retorna o token de acesso.

**Body:**
```json
{
  "username": "admin@episee.com",
  "password": "sua-senha"
}
```

**Resposta `200`:**
```json
{
  "access_token": "eyJhbGci...",
  "token_type": "bearer"
}
```

---

### `GET /api/auth/me`
Retorna os dados do usuário autenticado.

**Resposta `200`:**
```json
{
  "id": 1,
  "name": "Paulo Matsuda",
  "email": "admin@episee.com",
  "role": "gestor",
  "sector_id": null,
  "is_system_admin": true
}
```

---

## Câmeras

### `GET /api/cameras/`
Lista todas as câmeras.

**Resposta `200`:** Array de objetos câmera.

---

### `POST /api/cameras/`
Cria uma nova câmera. Requer role `gestor`.

**Body:**
```json
{
  "name": "Câmera Entrada",
  "location": "Portão principal",
  "sector_id": 1,
  "rtsp_url": "rtsp://192.168.1.100:554/stream",
  "is_active": true
}
```

---

### `PATCH /api/cameras/{camera_id}`
Atualiza dados de uma câmera.

---

### `DELETE /api/cameras/{camera_id}`
Remove uma câmera.

---

## Detecção

### `POST /api/detection/start/{camera_id}`
Inicia o stream de detecção para a câmera especificada. O backend:
1. Conecta ao RTSP da câmera
2. Inicia o loop de inferência YOLOv8
3. Inicia o FFmpeg para gerar o stream HLS

**Resposta `200`:**
```json
{
  "status": "started",
  "camera_id": 1
}
```

---

### `POST /api/detection/stop/{camera_id}`
Para o stream e libera recursos (processo FFmpeg + task asyncio).

---

### `GET /api/detection/status`
Retorna o status de todas as câmeras com detecção em andamento.

**Resposta `200`:**
```json
{
  "active_cameras": [1, 3],
  "details": {
    "1": { "running": true, "frames_processed": 1420 },
    "3": { "running": true, "frames_processed": 887 }
  }
}
```

---

### `GET /hls/{camera_id}/stream.m3u8`
Endpoint de streaming HLS. Consumido diretamente pelo player `hls.js` no frontend.

---

## Ocorrências

### `GET /api/occurrences/`
Lista ocorrências com suporte a filtros via query params.

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `sector_id` | int | Filtrar por setor |
| `camera_id` | int | Filtrar por câmera |
| `status` | string | `conforme` ou `nao_conforme` |
| `start_date` | string | Data inicial (ISO 8601) |
| `end_date` | string | Data final (ISO 8601) |
| `skip` | int | Paginação — offset |
| `limit` | int | Paginação — máximo de itens |

**Resposta `200`:**
```json
[
  {
    "id": 42,
    "camera_id": 1,
    "sector_id": 2,
    "timestamp": "2026-06-25T14:30:00",
    "status": "nao_conforme",
    "epi_detected": ["safety-vest"],
    "confidence": 0.87,
    "image_path": "/static/occurrences/42.jpg"
  }
]
```

---

### `GET /api/occurrences/{id}`
Retorna detalhes de uma ocorrência específica.

---

## Dashboard

### `GET /api/dashboard/summary`
KPIs gerais do sistema.

**Resposta `200`:**
```json
{
  "total_occurrences_today": 156,
  "non_compliant_today": 23,
  "compliance_rate": 85.3,
  "active_cameras": 4,
  "pending_epi_requests": 7
}
```

---

### `GET /api/dashboard/compliance-trend`
Tendência de conformidade dos últimos N dias.

| Parâmetro | Tipo | Default | Descrição |
|---|---|---|---|
| `days` | int | 7 | Número de dias a considerar |

---

## Setores

### `GET /api/sectors/`
Lista todos os setores com seus EPIs obrigatórios.

---

### `POST /api/sectors/`
Cria um novo setor. Requer role `gestor`.

**Body:**
```json
{
  "name": "Soldagem",
  "description": "Área de soldagem elétrica",
  "epis_obrigatorios": ["helmet", "gloves", "glasses"]
}
```

---

### `PATCH /api/sectors/{sector_id}`
Atualiza dados do setor, incluindo a lista de EPIs obrigatórios.

---

## Usuários

### `GET /api/users/`
Lista usuários. Gestores veem todos; trabalhadores veem apenas do próprio setor.

---

### `POST /api/users/`
Cria um novo usuário. Requer role `gestor`.

**Body:**
```json
{
  "name": "João Silva",
  "email": "joao@empresa.com",
  "password": "senha123",
  "role": "trabalhador",
  "sector_id": 2,
  "phone": "+5541999999999"
}
```

---

## Solicitações de EPI

### `GET /api/epi-requests/`
Lista solicitações. Trabalhadores veem apenas as próprias; gestores veem todas do setor.

---

### `POST /api/epi-requests/`
Abre uma nova solicitação de EPI.

**Body:**
```json
{
  "epi_type": "helmet",
  "reason": "Capacete danificado durante o turno"
}
```

---

### `PATCH /api/epi-requests/{id}`
Aprova ou rejeita uma solicitação. Requer role `gestor`.

**Body:**
```json
{
  "status": "aprovada",
  "motivo_rejeicao": null
}
```

---

## Chatbot

### `POST /api/chatbot/message`
Envia uma mensagem para o chatbot com IA (DeepSeek/OpenAI).

**Body:**
```json
{
  "message": "Quando devo usar óculos de proteção?",
  "history": []
}
```

**Resposta `200`:**
```json
{
  "response": "Os óculos de proteção devem ser usados em operações com risco de projeção de partículas, respingos químicos ou radiação intensa, conforme NR-6 item 6.3.1..."
}
```

---

## Telegram

### `POST /api/telegram/webhook`
Webhook para receber atualizações do bot Telegram. Configurado automaticamente pelo servidor.

---

## Notificações

### `GET /api/notifications/`
Lista notificações do usuário autenticado.

---

### `PATCH /api/notifications/{id}/read`
Marca uma notificação como lida.

---

## Vídeos de Treinamento

### `GET /api/training/epi-types`
Lista todos os tipos de EPI com informações de uso, norma NR-6 e vídeos associados.

---

### `GET /api/training/epi-types/{id}/videos`
Lista os vídeos de treinamento de um tipo de EPI específico.

---

## Relatórios

### `GET /api/reports/occurrences`
Retorna dados formatados para exportação de relatório de ocorrências.

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `sector_id` | int | Filtrar por setor |
| `start_date` | string | Data inicial |
| `end_date` | string | Data final |

---

## Códigos de Status

| Código | Significado |
|---|---|
| `200` | Sucesso |
| `201` | Recurso criado |
| `400` | Dados inválidos |
| `401` | Não autenticado (token ausente ou inválido) |
| `403` | Não autorizado (role insuficiente) |
| `404` | Recurso não encontrado |
| `422` | Erro de validação (Pydantic) |
| `500` | Erro interno do servidor |
