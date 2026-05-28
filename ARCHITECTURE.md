# Video Dubbing Agent — Colab Notebook Architecture

> **AVISO PARA IAs**: este documento existe **especificamente para evitar
> ciclos infinitos de "correção" que quebram o notebook**. Antes de fazer
> qualquer edição, leia a seção [REGRAS DE NÃO-REGRESSÃO](#regras-de-não-regressão) e
> rode o [Checklist de regressão](#checklist-de-regressão).

---

## Sumário

- [AVISOS CRÍTICOS](#avisos-críticos)
- [Estrutura do Notebook (9 células)](#estrutura-do-notebook-9-células)
- [Arquivos modificados pelos patches](#arquivos-modificados-pelos-patches-cell-4b)
- [Pipeline de dublagem (11 estágios)](#pipeline-de-dublagem-11-estágios)
- [Problemas conhecidos](#problemas-conhecidos-e-suas-soluções)
- [Como modificar o notebook](#como-modificar)
- [REGRAS DE NÃO-REGRESSÃO](#regras-de-não-regressão)
- [Checklist de regressão](#checklist-de-regressão)

---

## AVISOS CRÍTICOS

### 1. Stack web FIXO — não faça downgrade

```
anyio    >= 4.4   < 5
fastapi  >= 0.115 < 0.117
starlette>= 0.41  < 0.42
uvicorn  >= 0.30  < 0.35
pydantic >= 2.5   < 3
```

**Por que esse stack é obrigatório:**
- O `requirements.txt` original do HF Space pede `fastapi==0.104.1`
- No Colab base (Python 3.12 + anyio 4.x já carregada), essa versão antiga
  causa `ImportError: cannot import name 'ExceptionGroup' from 'anyio._core._exceptions'`
- Sintoma típico: servidor sobe, healthcheck OK, mas `GET /` retorna **500
  Internal Server Error**
- Causa: `starlette<0.41` + `anyio 4.x` → `FileResponse` usa
  `anyio.to_thread.run_sync` que falha porque o backend de event loop não está
  registrado para `ExceptionGroup`

### 2. `numpy==1.26.4` (constraints.txt) — não suba para 2.x

- `ctranslate2 4.x` tem ABI incompatível com `numpy>=2`
- Se numpy subir: `ImportError` em `faster_whisper` ou `ctranslate2`
- Fix obrigatório: `pip install numpy==1.26.4 --force-reinstall --no-deps`
  **ANTES** de qualquer outra instalação

### 3. `huggingface_hub==0.23.4` (constraints.txt) — não suba para >=0.26

- Versões `>=0.26` removeram `HfFolder`
- O app/faster-whisper usa `HfFolder` internamente
- Fix já aplicado: shim de `HfFolder` na Célula 4

### 4. `pydantic`: NÃO pinar em `==2.5.0`

- Causa build-from-source no Python 3.12+ (não tem wheel)
- Use: `pydantic>=2.5,<3` (deixa pip escolher wheel compatível)

### 5. NÃO use `pycloudflared`

- Lib desatualizada → gera quick-tunnels que caem com erro 1101
  ("invalid character 'e' looking for beginning of value")
- Use o binário oficial baixado direto do GitHub da Cloudflare

### 6. Filtros de áudio: NÃO remova `normalize=0` do `amix`

- `amix=inputs=N` **sem** `normalize=0` divide o volume por N
- Em vídeos com muitos segmentos, isso deixa o áudio quase mudo (-48 dB)
- Sintoma: usuário relata "áudio quase inaudível" mesmo após loudnorm
- Verificar: `grep "normalize=0" services/audio_assembler.py services/audio_mixer.py`

### 7. Cloudflare quick tunnel tem limite prático de upload

- O frontend pode carregar via `trycloudflare.com`, mas uploads grandes podem
  ser recusados pelo proxy antes de chegar ao FastAPI.
- Sintoma típico: alerta do navegador com `Unexpected token '<', '<!DOCTYPE ... is not valid JSON`.
- Causa provável: resposta HTML do Cloudflare, especialmente `413 Payload Too Large`.
- Referência oficial: Cloudflare documenta limite de upload de 100 MB para
  Free/Pro e sugere chunking, DNS-only ou upgrade:
  <https://developers.cloudflare.com/support/troubleshooting/http-status-codes/4xx-client-error/error-413/>

---

## Estrutura do Notebook (9 células)

| # | Tipo | Função | Tempo estimado Colab T4 |
|---|------|--------|-------------------------|
| 0 | Markdown | Cabeçalho e instruções | 0s |
| 1 | Code | **Sistema**: apt-get ffmpeg + verificação | ~10s |
| 2 | Code | **Clone**: snapshot_download + fallback `git+GIT_LFS_SKIP_SMUDGE` | ~30s |
| 3 | Code | **Pip**: constraints + numpy + HF Hub + fastapi stack + ASR + demucs + cloudflared | ~3-5min |
| 4 | Code | **Patches defensivos**: anyio upgrade + purge sys.modules + HfFolder/coqpit/torch.load shims | ~10s |
| 4b | Code | **Patches de qualidade**: 7 patches que reescrevem arquivos do app em disco | ~20s |
| 5 | Code | **Validação leve**: GPU/ctranslate2/ffmpeg (sem download de modelo) | ~5s |
| 6 | Code | **Smoke test**: verifica os 7 patches no disco (sem rodar pipeline) | ~5s |
| 7 | Code | **Servidor**: uvicorn em thread + logs configuráveis + cloudflared (→ localtunnel → pyngrok) + heartbeat | ∞ (bloqueante) |

### Por que existem Cell 4 E Cell 4b separadas

- **Cell 4** corrige problemas de **ambiente Python** (versões de libs, módulos
  carregados em memória, shims defensivos). Roda quando o Run all chega nela.
- **Cell 4b** corrige problemas de **qualidade do app** (sobrescreve arquivos
  `.py` e `.html` do repo clonado). Roda após Cell 4 (precisa do app
  funcionando).

Se você juntar as duas, perde idempotência: ao re-rodar, a Cell 4 já não
funciona porque os módulos já foram importados com patches.

---

## Arquivos modificados pelos patches (Cell 4b)

Todos os arquivos abaixo são **sobrescritos em disco** pela Célula 4b. Se você
clonar o repo novamente (Cell 2), os patches precisam ser reaplicados (Cell 4b).

| Arquivo | Patch | Marcador para verificar |
|---------|-------|-------------------------|
| `services/transcriber.py` | PATCH 1: `BatchedInferencePipeline` GPU | `BatchedInferencePipeline` |
| `services/vocal_separator.py` | PATCH 2: Demucs `-d cuda --segment 10` | `HAS_CUDA` |
| `services/tts_generator.py` | PATCH 3: concorrência 6× + `volume=8dB` + `loudnorm` por segmento | `MAX_CONCURRENT_TTS` |
| `services/audio_assembler.py` | PATCH 4: pydub overlay + `fade_in/fade_out 50ms` + microssilência 20ms + chunked >10min | `fade_in`, `fade_out` |
| `services/audio_mixer.py` | PATCH 5: `dynaudnorm` + `loudnorm -14 LUFS` (broadcast) | `dynaudnorm` |
| `services/merger.py` | PATCH 6a: `LANG_CODE_MAP` + sufixo `__pt-br` no nome | `LANG_CODE_MAP` |
| `routes/dub.py` | PATCH 6b: `safe_name` preserva nome original do upload | `safe_name`, `.original_filename` |
| `jobs/pipeline.py` | PATCH 6c: lê `.original_filename` para `video_info["title"]` | `_orig_fp` |
| `templates/index.html` | PATCH 7: dropzone CSS+JS + diagnóstico de JSON/Cloudflare + bloqueio de upload grande em `trycloudflare.com` | `vda-dropzone`, `__vdaJsonDiagnostics`, `95 * 1024 * 1024` |

### Ordem de aplicação dos patches (importante)

A Cell 4b é estritamente sequencial: PATCH 6 depende de PATCH 6a/6b/6c
aplicados na mesma execução. NÃO embaralhe a ordem.

---

## Pipeline de dublagem (11 estágios)

```
1. Download YouTube (yt-dlp) OU Upload local (FastAPI UploadFile)
   └─ routes/dub.py grava safe_name preservando file.filename original
   └─ pipeline.py lê .original_filename para o título

2. Extract audio (ffmpeg)
   ├─ audio_stt.wav (16kHz mono — para Whisper)
   └─ audio_original_stereo.wav (44.1kHz stereo — para mix com background)

3. Vocal separation (Demucs em CUDA --segment 10 para T4)
   ├─ vocals.mp3
   └─ no_vocals.mp3 (background original sem fala)

4. Transcribe (faster-whisper BatchedInferencePipeline)
   ├─ T4: large-v3 float16 batch=16 (~8-12× speedup vs sequencial)
   ├─ CPU: tiny int8 batch=1 (fallback automático)
   └─ transcript.json

5. Speaker profiling (forçado MALE)

6. Translate (deep-translator Google free tier)

7. TTS (Edge-TTS concorrente 6×, voz masculina)
   ├─ Por segmento: highpass=60 + lowpass=9000 + volume=8dB + loudnorm=-16
   └─ Speed-match com TTS_MAX_SPEED_FACTOR=1.3

8. Audio assembly (pydub overlay)
   ├─ fade_in/fade_out 50ms + microssilência 20ms (sem cliques)
   ├─ Vídeos >10min: chunks de 10 min (sem OOM)
   └─ Resultado: dubbed_audio_full.wav

9. Audio mixing (ffmpeg)
   ├─ vocals: dynaudnorm=f=300:g=21:p=0.9
   ├─ background: volume=0.15 (15%)
   ├─ amix=inputs=2:normalize=0:weights="1.0 0.7"
   └─ loudnorm=I=-14:LRA=9:TP=-1.0 (broadcast)

10. Merge video + audio (ffmpeg -c:v copy)
    └─ Output: <NomeOriginal>__<pt-br|en|es|...>.mp4

11. Subtitles (.srt em ambos idiomas)
```

### Loudness target

- **Integrated loudness**: -14 LUFS (broadcast YouTube/streaming standard)
- **True peak**: max -1.0 dBFS (sem clipping)
- **LRA (loudness range)**: 9 LU

---

## Problemas conhecidos e suas soluções

### "cloudflared erro 1101 / invalid character 'e'"

- **Causa**: endpoint `api.trycloudflare.com` instável (bug intermitente do
  servidor Cloudflare em 2024-2025)
- **Solução**: Cell 7 tem 3 fallbacks em cascata configuráveis:
  cloudflared → localtunnel (`lt` ou `npx localtunnel`) → pyngrok
- **Se todos falharem**: Runtime → Restart session → Run all
- **Para pular Cloudflare**: antes da Célula 7, rode
  `import os; os.environ["VDA_TUNNEL"]="ngrok,localtunnel"`

### "Unexpected token '<', '<!DOCTYPE ... is not valid JSON'"

- **Causa**: o JavaScript chamou `response.json()`, mas recebeu HTML. Isso
  acontece quando o proxy/túnel devolve uma página de erro em vez da API JSON.
- **Causa mais provável em upload local**: Cloudflare recusou o corpo da
  requisição antes de chegar ao Colab. Em Free/Pro, a documentação oficial
  lista upload máximo de 100 MB para requests HTTP:
  <https://developers.cloudflare.com/cache/concepts/default-cache-behavior/#upload-limits>
- **Por que o log do Colab não mostra detalhes**: se o Cloudflare bloqueia a
  requisição, o FastAPI nunca recebe o upload.
- **Patch que resolve o diagnóstico**: PATCH 7b troca o erro genérico por uma
  mensagem com status HTTP, `content-type`, trecho da resposta, `cf-ray` quando
  existir, e bloqueia no frontend arquivos >95 MB via `trycloudflare.com`.
- **Como contornar**: use arquivo <95 MB, cole URL do YouTube, compacte/divida
  o vídeo, ou reinicie a Célula 7 com `VDA_TUNNEL="ngrok,localtunnel"`.

### "Run all trava por >5 min"

- **Causa MAIS PROVÁVEL**: Cell 5 ou Cell 6 baixando whisper-medium/large-v3 (1.5-3 GB)
- **Verificação**: Cell 5 e Cell 6 são validações **leves** — não devem carregar modelos
- **Modelo é baixado apenas no primeiro job real** (cache permanente após isso)

### "Áudio do vídeo dublado muito baixo"

- **Causa**: amix sem `normalize=0` + falta de amplificação no TTS
- **Patches que resolvem**: PATCH 3 (TTS +8dB) + PATCH 4 (amix normalize=0) + PATCH 5 (loudnorm -14 LUFS)
- **Verificar**: `ffmpeg -i output.mp4 -af volumedetect -f null - 2>&1 | grep mean_volume`
- **Esperado**: -22 a -15 dB (mesmo nível do vídeo original)

### "Nome do arquivo virou source_video__pt-br.mp4"

- **Causa**: `routes/dub.py` original salvava upload como `source_video.mp4`
- **Patch que resolve**: PATCH 6b (sanitiza `file.filename` → `safe_name`)
- **Verificar**: upload de `meu_video.mp4` deve produzir `meu_video__pt-br.mp4`

### "ImportError: ExceptionGroup ao acessar /"

- **Causa**: stack web com fastapi==0.104.1 + anyio 4.x
- **Patch que resolve**: Cell 3 instala fastapi>=0.115 + starlette>=0.41
  **sem** `--constraint` (porque constraints força HF Hub 0.23.4 que
  bloqueia upgrades)
- **Verificar**: `python -c "import importlib.metadata as m; print(m.version('fastapi'))"` deve ser >= 0.115

### "UnicodeEncodeError ao salvar index.html"

- **Causa**: emojis Unicode `⬇️` `📁` causam surrogates em alguns encodings
- **Patch que resolve**: PATCH 7 usa entidade HTML `&#11015;` e texto ASCII
  para o nome do arquivo, com `encoding="utf-8"` explícito e
  `errors="xmlcharrefreplace"`

### "Chiados/cliques entre segmentos TTS"

- **Causa**: transições abruptas entre segmentos sem fade
- **Patch que resolve**: PATCH 4 aplica `fade_in(50ms)` + `fade_out(50ms)` +
  microssilência de 20ms antes/depois de cada segmento
- **Métrica**: contagem de amplitude jumps > 10000/32768 deve ser <0.5%

---

## Como modificar

### Alterar nível de log/debug

A Célula 7 lê `VDA_LOG_LEVEL` antes de iniciar o `uvicorn`.

Para mais detalhes, rode antes da Célula 7:

```python
import os
os.environ["VDA_LOG_LEVEL"] = "DEBUG"
```

`DEBUG` mostra requests/responses no middleware (`method`, path,
`content-length`, `content-type`) e stack trace de exceções não tratadas.

Para reduzir ruído depois que corrigir o problema:

```python
import os
os.environ["VDA_LOG_LEVEL"] = "INFO"     # padrão
# ou:
os.environ["VDA_LOG_LEVEL"] = "WARNING"  # só avisos/erros
os.environ["VDA_LOG_LEVEL"] = "ERROR"    # só erros
```

Se a Célula 7 já estiver rodando, é preciso interromper a célula e rodá-la de
novo para aplicar o novo nível. Se o erro vier do Cloudflare antes de chegar no
FastAPI, `DEBUG` não mostra stack trace do app; nesse caso use o alerta novo do
frontend e/ou pule Cloudflare com `VDA_TUNNEL`.

### Escolher backend de túnel

A Célula 7 lê `VDA_TUNNEL` como lista ordenada separada por vírgula. Padrão:

```python
import os
os.environ["VDA_TUNNEL"] = "cloudflared,localtunnel,ngrok"
```

Para evitar o limite de upload do Cloudflare em vídeos grandes:

```python
import os
os.environ["VDA_TUNNEL"] = "ngrok,localtunnel"
```

Para forçar um único backend:

```python
os.environ["VDA_TUNNEL"] = "localtunnel"
```

`localtunnel` usa `lt` se existir; caso contrário tenta `npx -y localtunnel`.
`ngrok` usa `NGROK_AUTH_TOKEN` se você definir o token antes da Célula 7.

### Adicionar um novo idioma de saída

1. Editar `LANG_CODE_MAP` no PATCH 6a (dentro da Cell 4b, string `merger_src`)
2. Verificar que edge-tts suporta o idioma: `edge-tts --list-voices | grep <lang>`
3. Adicionar voz masculina em `config.py:EDGE_TTS_VOICES`

### Atualizar o modelo Whisper

- Modelos: `tiny`, `base`, `small`, `medium`, `large-v2`, `large-v3`
- Editar na Cell 4b: `WHISPER_MODEL = "large-v3" if HAS_CUDA and VRAM_GB >= 14 else "medium"`
- T4 (15 GB) aguenta `large-v3` com `batch_size=16` confortavelmente
- Modelos `large` consomem ~6 GB VRAM — Demucs precisa de ~3 GB extra → fica apertado

### Ajustar amplificação/compressão de áudio

- **TTS por segmento** (PATCH 3): valor de `volume=8dB` no filtro chain do `tts_generator.py`
- **Mix final** (PATCH 5): parâmetros do `dynaudnorm=f=300:g=21:p=0.9` e
  `loudnorm=I=-14:LRA=9:TP=-1.0` no `audio_mixer.py`
- **Sempre meça antes/depois** com `ffmpeg ... -af volumedetect` e
  `ffmpeg ... -af ebur128`

### Vídeos longos (2-3 horas)

- O pipeline já está preparado: PATCH 4 ativa modo chunked (10 min/chunk) para
  vídeos > 10 min automaticamente
- Demucs com `--segment 10` para não estourar VRAM
- faster-whisper streama segmentos (memória constante)
- Edge-TTS gera segmentos em paralelo (6 simultâneos)
- **Limite real**: cota de disco do Colab (~100 GB) + sessão de 12h (free)

---

## REGRAS DE NÃO-REGRESSÃO

| # | Regra | Verificação |
|---|-------|-------------|
| NR1 | Nunca faça downgrade de anyio, fastapi, starlette ou uvicorn | `python -c "import importlib.metadata as m; assert int(m.version('anyio').split('.')[0])>=4"` |
| NR2 | Nunca remova o purge de `sys.modules` da Célula 4 | `grep "sys.modules.pop" cell4` |
| NR3 | Nunca instale `fastapi==0.104.1` — causa ABI break com anyio 4 | `grep "fastapi==0.104" cell3` deve **NÃO** retornar |
| NR4 | Nunca use `amix` sem `normalize=0` — divide volume por N entradas | `grep "amix" patches` deve sempre conter `normalize=0` |
| NR5 | Nunca remova `--constraint constraints.txt` do pip *geral* (mas SIM da instalação fastapi/starlette) | `grep -c constraint cell3` deve ser >= 1 |
| NR6 | Nunca instale `numpy>=2` — quebra ctranslate2 | `numpy==1.26.4` no Step A do pip |
| NR7 | Nunca trunce células — `json.loads()` do .ipynb deve funcionar sem erro | `python -c "import json; json.load(open('*.ipynb'))"` |
| NR8 | Célula 5 e 6 nunca devem fazer download de modelos | `grep "WhisperModel" cell5 cell6` deve **NÃO** retornar |
| NR9 | Nunca use `pycloudflared` — binário desatualizado, erro 1101 | `grep pycloudflared *.ipynb` deve **NÃO** retornar |
| NR10 | Nunca declare entrega sem rodar a validação final da Fase 5 | rodar `python /home/user/work/nb/test_final.py` |
| NR11 | Nunca remova o `safe_name` em `routes/dub.py` | nome original do upload precisa ser preservado |
| NR12 | Nunca remova `fade_in/fade_out` em `audio_assembler.py` | sem fade = cliques audíveis entre segmentos |
| NR13 | Nunca remova o diagnóstico de HTML/JSON do frontend | `__vdaJsonDiagnostics` e `Response.prototype.json` em `templates/index.html` |
| NR14 | Nunca remova os controles da Célula 7 para debug/túnel | `VDA_LOG_LEVEL` e `VDA_TUNNEL` no notebook |

---

## Checklist de regressão

Rode após **qualquer** edição:

```bash
# 1. Stack web compatível
python3 -c "
import importlib.metadata as m
assert int(m.version('anyio').split('.')[0])>=4, 'anyio<4'
assert int(m.version('starlette').split('.')[1])>=41, 'starlette<0.41'
assert int(m.version('fastapi').split('.')[1])>=115, 'fastapi<0.115'
print('Stack: anyio',m.version('anyio'),'starlette',m.version('starlette'),'fastapi',m.version('fastapi'))
"

# 2. Patches no disco (após Cell 4b)
for f in services/transcriber.py services/audio_assembler.py services/audio_mixer.py services/merger.py routes/dub.py templates/index.html; do
  python3 -c "
s=open('$f').read()
markers={
  'services/transcriber.py':['BatchedInferencePipeline'],
  'services/audio_assembler.py':['fade_in','fade_out'],
  'services/audio_mixer.py':['dynaudnorm','normalize=0'],
  'services/merger.py':['LANG_CODE_MAP'],
  'routes/dub.py':['safe_name'],
  'templates/index.html':['vda-dropzone','__vdaJsonDiagnostics','Response.prototype.json'],
}
missing=[m for m in markers['$f'] if m not in s]
assert not missing, f'$f sem patch: {missing}'
print('$f: OK')
"
done

# 3. Pipeline E2E (smoke test com vídeo sintético)
python3 /home/user/work/nb/test_final.py 2>&1 | tail -15

# 4. JSON do .ipynb íntegro
python3 -c "
import json
nb=json.load(open('/home/user/work/nb/video_dubbing_agent_colab.ipynb'))
assert len(nb['cells'])==9, f'Esperava 9, achei {len(nb[\"cells\"])}'
src='\\n'.join(''.join(c.get('source',[])) for c in nb['cells'])
assert 'VDA_LOG_LEVEL' in src and 'VDA_TUNNEL' in src
print('Notebook: 9 células OK')
"
```

Se qualquer check falhar → **NÃO entregue**. Corrija a causa raiz, revalide
todos os checks dependentes, e só depois entregue.

---

## Histórico de versões

| Versão | Data | Mudanças principais |
|--------|------|---------------------|
| v1 | 2026-05-23 | Inicial — converter Space FastAPI para Colab T4, cloudflared |
| v2 | 2026-05-23 | URL pública funcional (3 backends de túnel) |
| v3 | 2026-05-23 | GPU + áudio normalizado + sufixo `__pt-br` + drag-and-drop |
| v4 | 2026-05-23 | Stack web compat anyio>=4, Cell 5/6 leves, validação Phase 2 |
| **v5** | **2026-05-24** | **BatchedInference + crossfade 50ms + dynaudnorm + nome original + entidades HTML** |
| **v6** | **2026-05-25** | **Diagnóstico de HTML/JSON no frontend, guarda de upload Cloudflare, `VDA_LOG_LEVEL` e `VDA_TUNNEL`** |

---

**Última validação automatizada (v6)**:

```
✅ CHECK B: Filename ZC01s_uU1LlMOLag__pt-br.mp4
✅ CHECK F: mean_volume -19.5 dB (range -25..-5)
✅ CHECK G: True peak -4.0 dBFS (sem clipping)
✅ CHECK H: HTML com vda-dropzone + __vdaJsonDiagnostics
✅ CHECK J: anyio=4.13.0 starlette=0.41.3
✅ CHECK K: pipeline E2E completou em 3.1s
```
