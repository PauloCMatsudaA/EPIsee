# Detecção de EPIs

## Visão Geral

O módulo de detecção é o coração do EPIsee. Ele usa YOLOv8 para analisar frames de câmeras em tempo real e determinar se os trabalhadores estão usando os EPIs obrigatórios para o setor monitorado.

---

## Modelo YOLOv8 (`best.pt`)

O modelo foi treinado e fine-tuned a partir da arquitetura YOLOv8 com o dataset SH17. Ele detecta simultaneamente pessoas, cabeças e diferentes tipos de EPIs, permitindo inferências de conformidade complexas.

**Fonte do modelo:** [huggingface.co/MatsudaPaulo/episeeyolo](https://huggingface.co/MatsudaPaulo/episeeyolo)

### Classes Detectadas

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
| `shoes` | Calçado de segurança |
| `person` | Pessoa detectada |
| `head` | Cabeça (para inferir ausência de capacete) |
| `face` | Rosto (para inferir EPIs faciais) |
| `ear` | Orelha (para inferir ausência de protetor auricular) |
| `hands` | Mãos (para inferir ausência de luvas) |
| `foot` | Pé (para inferir ausência de calçado de segurança) |
| `tools` | Ferramentas (contexto operacional) |

---

## Parâmetros de Detecção

Definidos em `detection_service_real.py`:

```python
CONFIANCA_MINIMA  = 0.45   # Threshold de confiança mínima para aceitar uma detecção
INTERVALO_SALVAR  = 30     # Intervalo em segundos entre salvamentos de ocorrência
YOLO_INTERVALO    = 0.3    # Intervalo em segundos entre inferências
IOUI_HELMET_HEAD  = 0.25   # IoU mínimo para confirmar capacete sobre cabeça
IOUI_VEST_PERSON  = 0.15   # IoU mínimo para confirmar colete sobre pessoa
```

### Por que grupos de IoU diferentes?

**EPIs Faciais** (capacete, óculos, máscara...):
A bounding box do EPI facial precisa ter sobreposição significativa com a bounding box da cabeça detectada. Isso garante que o EPI está realmente sendo usado na face, não apenas presente na cena.

**EPIs de Torso** (colete):
A bounding box do colete precisa ter sobreposição com a bounding box da pessoa. O threshold menor (0.15) é usado porque coletes muitas vezes aparecem com sobreposição parcial dependendo do ângulo da câmera.

**EPIs Sem Interseção** (luvas, macacão):
Luvas e macacões são verificados apenas pela presença na cena, sem análise de IoU, pois sua localização relativa à pessoa varia muito.

---

## Lógica de Conformidade por Setor

### Configuração

Cada setor define seus EPIs obrigatórios. Isso pode ser feito de dois formas:

**1. Via dashboard (recomendado):**  
Gestor acessa Setores → edita o setor → define EPIs obrigatórios.  
Armazenado em `sectors.epis_obrigatorios` como JSON.

**2. Via código (fallback estático):**  
`Server/app/core/sector_epi_config.py`:
```python
EPIS_POR_SETOR: dict[int, set[str]] = {
    1: {"helmet", "safety-vest", "glasses"},   # Ex: Construção Civil
    2: {"helmet", "gloves", "safety-vest"},    # Ex: Metalurgia
    3: {"safety-vest", "glasses"},             # Ex: Laboratório
    4: {"safety-vest", "helmet"},              # Ex: Logística
}
EPIS_OBRIGATORIOS_PADRAO: set[str] = {"safety-vest"}  # Fallback global
```

### Algoritmo de Verificação

```
Para cada frame analisado:
  1. Detectar todas as bboxes com confiança ≥ 0.45
  2. Separar detecções por classe (person, head, EPIs)
  3. Para cada pessoa detectada:
     a. Identificar cabeças próximas (IoU com pessoa)
     b. Verificar EPIs faciais sobre as cabeças (IoU ≥ 0.25)
     c. Verificar colete sobre pessoa (IoU ≥ 0.15)
     d. Verificar luvas/macacão na cena
  4. Montar conjunto de EPIs detectados por pessoa
  5. Comparar com EPIs obrigatórios do setor da câmera
  6. Se {EPIs obrigatórios} ⊄ {EPIs detectados}: → NÃO CONFORME
  7. Caso contrário: → CONFORME
```

---

## Streaming HLS

O sistema converte o stream RTSP em HLS para exibição no dashboard:

```
Camera RTSP
    │
    ▼
OpenCV (leitura de frames)
    │
    ▼
YOLOv8 (inferência + anotação visual)
    │ frame anotado
    ▼
FFmpeg (encode H.264 → HLS)
    │
    ▼
hls_streams/{camera_id}/stream.m3u8
    │
    ▼
hls.js (player no browser)
```

O diretório `hls_streams/` é criado automaticamente na raiz do servidor.

---

## Ciclo de Vida de uma Câmera

```
Estado: INATIVA
    │
    │  POST /api/detection/start/{id}
    ▼
Estado: INICIANDO
    │ (asyncio task criada)
    │ (processo FFmpeg iniciado)
    ▼
Estado: ATIVA
    │ (loop de inferência rodando)
    │ (HLS disponível em /hls/{id}/stream.m3u8)
    │
    │  POST /api/detection/stop/{id}
    ▼
Estado: PARANDO
    │ (task cancelada)
    │ (FFmpeg encerrado)
    ▼
Estado: INATIVA
```

---

## Alertas de Não-Conformidade

Quando uma não-conformidade é detectada:

1. **Ocorrência registrada** no banco com:
   - `status = "nao_conforme"`
   - `epi_detected` = lista de EPIs encontrados na cena
   - `image_path` = caminho do frame capturado
   - `confidence` = confiança média das detecções

2. **Alerta Telegram** enviado (se configurado):
   ```
   ⚠️ EPIsee Alert
   
   Setor: Soldagem
   Câmera: Câmera Entrada
   Hora: 14:32:15
   
   EPIs ausentes: helmet, glasses
   EPIs detectados: safety-vest
   
   [imagem do frame]
   ```

3. **Notificação interna** criada para o gestor do setor.

---

## Performance e Recursos

O serviço de detecção roda de forma assíncrona, usando:
- `asyncio.Task` por câmera ativa
- `Thread` separada para a inferência YOLOv8 (evita bloqueio do event loop)
- Queue para comunicação entre thread de inferência e loop async

---

## Adicionar uma Nova Classe de EPI

1. Retreinar ou fine-tune o modelo YOLOv8 com a nova classe
2. Atualizar `CLASSES_EPI` em `detection_service_real.py`:
   ```python
   CLASSES_EPI = {
       "glasses", "face-mask-medical", "face-guard",
       "earmuffs", "gloves", "safety-vest",
       "helmet", "medical-suit", "safety-suit",
       "nova-classe",  # ← adicionar aqui
   }
   ```
3. Classificar o novo EPI no grupo correto (facial, torso ou sem interseção)
4. Atualizar as configurações de setores conforme necessário
