# Configurando Câmeras

Este guia cobre tudo que você precisa saber para adicionar câmeras ao sistema, desde descobrir a URL RTSP até iniciar a detecção em tempo real.

---

## O que é RTSP?

RTSP (Real Time Streaming Protocol) é o protocolo que câmeras IP usam para transmitir vídeo em rede. O EPIsee se conecta diretamente a esse stream, sem precisar de software da câmera instalado.

Toda câmera IP tem uma URL RTSP com o seguinte formato geral:

```
rtsp://<usuario>:<senha>@<ip>:<porta>/<caminho>
```

Exemplo real:
```
rtsp://admin:12345@192.168.1.100:554/stream1
```

---

## Passo 1 — Descobrir a URL RTSP da sua câmera

Cada fabricante tem um padrão diferente. Consulte a tabela abaixo:

| Fabricante | URL RTSP padrão |
|---|---|
| **TP-Link Tapo** (C310, C320, etc.) | `rtsp://<usuario>:<senha>@<ip>/stream1` |
| **Hikvision** | `rtsp://admin:<senha>@<ip>:554/Streaming/Channels/101` |
| **Dahua** | `rtsp://admin:<senha>@<ip>:554/cam/realmonitor?channel=1&subtype=0` |
| **Intelbras** | `rtsp://admin:<senha>@<ip>:554/cam/realmonitor?channel=1&subtype=0` |
| **Axis** | `rtsp://<usuario>:<senha>@<ip>/axis-media/media.amp` |
| **Reolink** | `rtsp://admin:<senha>@<ip>:554/h264Preview_01_main` |
| **Genérica / ONVIF** | `rtsp://<usuario>:<senha>@<ip>:554/onvif1` |

> Se sua câmera não está na lista, procure no manual ou no site [ispydb.com](https://www.ispydb.com) pelo modelo exato.

### Câmera TP-Link Tapo (usada no projeto)

A Tapo exige uma conta de usuário criada dentro do app antes de usar RTSP:

1. Abra o **app Tapo** no celular
2. Acesse a câmera → toque nos três pontos (⋮) → **Configurações**
3. Vá em **Avançado** → **Configurações de Conta**
4. Crie um usuário e senha (ex.: `episee` / `senha123`)
5. A URL RTSP será:
   ```
   rtsp://episee:senha123@192.168.1.xxx/stream1
   ```

---

## Passo 2 — Descobrir o IP da câmera

Se você não sabe o IP da câmera na rede, use um desses métodos:

**Via app do fabricante:**  
A maioria dos apps (Tapo, iCSee, DMSS) mostra o IP na tela de detalhes da câmera.

**Via roteador:**  
Acesse o painel do seu roteador (geralmente `192.168.0.1` ou `192.168.1.1`) e procure a lista de dispositivos conectados.

**Via terminal (Linux/macOS):**
```bash
# Varre a rede local em busca de dispositivos ativos
nmap -sn 192.168.1.0/24
```

**Via terminal (Windows):**
```cmd
arp -a
```

---

## Passo 3 — Testar a URL antes de cadastrar

Antes de adicionar a câmera no EPIsee, confirme que a URL funciona:

**Com VLC:**
1. Abra o VLC
2. `Mídia` → `Abrir fluxo de rede` (Ctrl+N)
3. Cole a URL RTSP e clique em Reproduzir
4. Se o vídeo abrir, a URL está correta

**Com FFmpeg (terminal):**
```bash
ffplay rtsp://usuario:senha@192.168.1.100/stream1
```

**Com Python (teste rápido):**
```python
import cv2

url = "rtsp://usuario:senha@192.168.1.100/stream1"
cap = cv2.VideoCapture(url)

if cap.isOpened():
    print("Conectado com sucesso!")
    ret, frame = cap.read()
    print(f"Frame lido: {ret} — resolucao: {frame.shape if ret else 'N/A'}")
else:
    print("Falha ao conectar. Verifique a URL, usuario e senha.")

cap.release()
```

---

## Passo 4 — Criar um setor (se ainda não existe)

Toda câmera precisa estar vinculada a um setor, pois os EPIs obrigatórios são definidos por setor.

1. Acesse o dashboard em `http://localhost:5173`
2. Vá em **Setores** no menu lateral
3. Clique em **Novo Setor**
4. Preencha:
   - **Nome:** Ex.: `Soldagem`, `Logística`, `Laboratório`
   - **Descrição:** Opcional
   - **EPIs obrigatórios:** Selecione quais EPIs são exigidos nesse setor

### EPIs disponíveis para seleção

| Identificador | Nome |
|---|---|
| `helmet` | Capacete de segurança |
| `safety-vest` | Colete refletivo |
| `glasses` | Óculos de proteção |
| `gloves` | Luvas |
| `face-mask-medical` | Máscara facial / N95 |
| `face-guard` | Protetor facial completo |
| `earmuffs` | Protetor auricular |
| `medical-suit` | Macacão descartável |
| `safety-suit` | Roupa de segurança |

---

## Passo 5 — Cadastrar a câmera no sistema

### Via dashboard (recomendado)

1. Acesse **Câmeras** no menu lateral
2. Clique em **Nova Câmera**
3. Preencha os campos:

| Campo | Descrição | Exemplo |
|---|---|---|
| **Nome** | Nome identificador da câmera | `Entrada Principal` |
| **Localização** | Onde fisicamente ela está | `Portão Lateral - Bloco B` |
| **Setor** | Setor ao qual pertence | `Soldagem` |
| **URL RTSP** | Endereço do stream da câmera | `rtsp://admin:1234@192.168.1.50/stream1` |
| **Ativa** | Se a câmera deve ser monitorada | `Sim` |

4. Clique em **Salvar**

### Via API (para integrações ou scripts)

```bash
curl -X POST http://localhost:8000/api/cameras/ \
  -H "Authorization: Bearer SEU_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Entrada Principal",
    "location": "Portao Lateral - Bloco B",
    "sector_id": 1,
    "rtsp_url": "rtsp://admin:1234@192.168.1.50/stream1",
    "is_active": true
  }'
```

---

## Passo 6 — Iniciar a detecção

Após cadastrar, volte para a tela de **Câmeras** e clique em **Iniciar Detecção** na câmera desejada.

O sistema vai:
1. Conectar ao stream RTSP via OpenCV
2. Iniciar o processo FFmpeg para gerar o stream HLS
3. Criar uma `asyncio.Task` que analisa frames com YOLOv8 a cada **0.3 segundos**
4. Disponibilizar o stream ao vivo no player do dashboard

> O modelo `best.pt` precisa estar presente em `Server/best.pt` para a detecção funcionar. Se o modelo não for encontrado, o stream HLS sobe normalmente, mas sem análise de EPIs.

### Verificar se está rodando

Acesse `http://localhost:8000/api/cameras` e verifique o campo `last_seen` — ele é atualizado sempre que a câmera é iniciada. Ou veja o stream ao vivo diretamente no dashboard.

---

## Usando um arquivo de vídeo (sem câmera física)

O EPIsee tem suporte a fallback por arquivo de vídeo, útil para testes e demonstrações.

Coloque um arquivo chamado `teste.mp4` na pasta `Server/` e inicie a detecção normalmente. O sistema detecta automaticamente a ausência de câmera e usa o arquivo no lugar.

```bash
# Exemplo: copiar um vídeo de teste para a pasta correta
cp meu_video.mp4 Server/teste.mp4
```

> O arquivo deve estar no formato MP4 com codec H.264. Outros formatos suportados pelo OpenCV também funcionam (AVI, MKV, MOV).

---

## Múltiplas câmeras simultâneas

O EPIsee suporta N câmeras rodando ao mesmo tempo. Cada câmera ocupa:
- Uma `asyncio.Task` para a inferência YOLOv8
- Um processo `subprocess.Popen` do FFmpeg para o HLS

Cada câmera é identificada pelo seu `camera_id` e tem seu próprio diretório de stream:
```
Server/
└── hls_streams/
    ├── 1/          ← câmera ID 1
    │   ├── stream.m3u8
    │   └── segment0.ts, segment1.ts ...
    ├── 2/          ← câmera ID 2
    │   └── stream.m3u8
    └── ...
```

O stream de cada câmera é acessível em:
```
http://localhost:8000/hls/<camera_id>/stream.m3u8
```

### Recomendações de hardware por câmera

| Câmeras simultâneas | Hardware recomendado |
|---|---|
| 1–2 | CPU moderna (4+ cores), 8 GB RAM |
| 3–5 | CPU 8+ cores, 16 GB RAM, GPU NVIDIA recomendada |
| 6+ | GPU NVIDIA com CUDA obrigatória, 32 GB RAM |

> Com GPU NVIDIA, o YOLOv8 usa CUDA automaticamente — não é necessária nenhuma configuração adicional além de ter o driver instalado.

---

## Parando a detecção

### Via dashboard
Clique em **Parar Detecção** na câmera desejada.

### Via API
```bash
curl -X POST http://localhost:8000/api/cameras/1/stop-detection \
  -H "Authorization: Bearer SEU_TOKEN"
```

Isso cancela a `asyncio.Task` e encerra o processo FFmpeg da câmera. Os arquivos HLS no disco são mantidos até o próximo start ou até a câmera ser deletada.

---

## Removendo uma câmera

1. Pare a detecção antes de remover (recomendado)
2. No dashboard, clique em **Excluir** na câmera desejada

Ao excluir, o sistema também remove automaticamente a pasta `hls_streams/<camera_id>/` do disco.

---

## Solução de Problemas

### Stream não abre / câmera não conecta

```
Verifique:
1. A URL RTSP está correta? Teste com VLC antes.
2. O usuário e senha da câmera estão certos?
3. A câmera e o servidor estão na mesma rede?
4. O firewall do servidor está bloqueando a porta 554?
5. A câmera suporta RTSP? (câmeras muito antigas podem não suportar)
```

Teste de porta no terminal:
```bash
# Verifica se a porta RTSP da câmera está acessível
nc -zv 192.168.1.100 554
```

---

### Detecção não inicia (`best.pt não encontrado`)

O modelo não está na pasta correta. Coloque o arquivo em `Server/best.pt`. O sistema vai encontrá-lo automaticamente no próximo start.

Se você tem o `HF_TOKEN` configurado no `.env`, o download acontece automaticamente na inicialização do servidor.

---

### Stream HLS disponível mas sem vídeo no player

O FFmpeg pode não ter sido instalado ou não foi encontrado no PATH.

```bash
# Verificar se o FFmpeg está instalado
ffmpeg -version

# Instalar no Ubuntu/Debian
sudo apt install ffmpeg -y

# Verificar se o servidor consegue encontrá-lo
which ffmpeg
```

Após instalar, reinicie o servidor (`uvicorn main:app --reload`).

---

### Alta latência no stream ao vivo

O HLS tem latência inerente de 2–6 segundos por padrão. Isso é esperado. Para reduzir:

- Diminua o valor de `hls_time` e `hls_list_size` nos parâmetros FFmpeg dentro de `iniciar_hls()` em `detection_service_real.py`
- Use uma câmera com resolução menor (720p em vez de 1080p)
- Garanta que servidor e câmera estejam na mesma rede local (evite Wi-Fi congestionado)

---

### Câmera aparece como offline após reiniciar o servidor

As `asyncio.Task` e os processos FFmpeg são encerrados quando o servidor para. Você precisa clicar em **Iniciar Detecção** novamente após reiniciar.

Para iniciar todas as câmeras ativas automaticamente ao subir o servidor, você pode adicionar um evento `startup` no `main.py`:

```python
@app.on_event("startup")
async def auto_start_cameras():
    async with AsyncSessionLocal() as db:
        result = await db.execute(select(Camera).where(Camera.is_active == True))
        cameras = result.scalars().all()
        for cam in cameras:
            if cam.rtsp_url:
                iniciar_hls(cam.id, cam.rtsp_url)
                task = asyncio.create_task(
                    processar_stream_camera(cam.id, cam.rtsp_url, cam.sector_id or 1)
                )
                tarefas_deteccao[cam.id] = task
```
