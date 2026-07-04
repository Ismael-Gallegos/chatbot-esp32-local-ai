# Setup del entorno de IA local (laptop ASUS)

Este documento describe cómo se configuró el entorno de inteligencia artificial local para el chatbot ESP32, incluyendo problemas encontrados y sus soluciones.

## Hardware de referencia

- Laptop ASUS Gaming, Intel i5-11a, 32GB RAM
- GPU: NVIDIA GeForce RTX 2050 (4GB VRAM)
- Windows con PowerShell 7

## 1. Entorno Python aislado

El sistema tenía Python 3.14 instalado, demasiado nuevo para el ecosistema de ML (PyTorch, Coqui TTS aún no publican soporte estable para esa versión). Se creó un entorno virtual dedicado con Python 3.11 sin modificar el Python del sistema:

```powershell
py -3.11 -m venv venv-ia
.\venv-ia\Scripts\Activate.ps1
python --version   # debe mostrar 3.11.x
```

**Por qué:** aísla las dependencias del proyecto de IA sin afectar otros usos de Python en la máquina, y evita incompatibilidades con versiones de Python demasiado recientes que PyTorch aún no soporta oficialmente.

## 2. Problema recurrente: Kaspersky bloquea pip

Kaspersky Premium hace inspección SSL del tráfico HTTPS, lo cual rompe la verificación de certificados de `files.pythonhosted.org` y `pypi.org`, causando errores como:

```
SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: self-signed certificate in certificate chain'))
```

**Solución temporal (usada en este proyecto):** agregar `--trusted-host` a cada instalación:

```powershell
pip install <paquete> --trusted-host files.pythonhosted.org --trusted-host pypi.org --trusted-host download.pytorch.org
```

**Solución permanente recomendada:** en Kaspersky Premium, ir a Configuración avanzada → Red → Analizar conexiones cifradas → Excepciones, y agregar `files.pythonhosted.org`, `pypi.org`, y `download.pytorch.org`.

## 3. PyTorch con soporte CUDA

```powershell
pip install torch torchaudio --index-url https://download.pytorch.org/whl/cu121 --trusted-host download.pytorch.org --trusted-host files.pythonhosted.org --trusted-host pypi.org
```

Verificación de que CUDA está disponible:

```powershell
python -c "import torch; print('CUDA disponible:', torch.cuda.is_available()); print('GPU:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'N/A')"
```

Resultado esperado: `CUDA disponible: True` y `GPU: NVIDIA GeForce RTX 2050`.

## 4. Coqui TTS (texto a voz)

Se usa el fork comunitario mantenido `coqui-tts` (el paquete original `TTS` de la empresa Coqui está discontinuado):

```powershell
pip install coqui-tts --trusted-host files.pythonhosted.org --trusted-host pypi.org
```

### Problema encontrado: incompatibilidad con transformers 5.x

Al instalar, pip resuelve automáticamente la versión más nueva de `transformers` (5.x), pero el código de XTTS/Tortoise dentro de `coqui-tts` depende de una función interna (`isin_mps_friendly`) que fue eliminada en esa versión, causando:

```
ImportError: cannot import name 'isin_mps_friendly' from 'transformers.pytorch_utils'
```

**Solución confirmada por el mantenedor del proyecto:**

```powershell
pip install "transformers==4.57.6" --trusted-host files.pythonhosted.org --trusted-host pypi.org
```

### Prueba de humo

```powershell
python -c "from TTS.api import TTS; import torch; tts = TTS('tts_models/en/ljspeech/tacotron2-DDC').to('cuda' if torch.cuda.is_available() else 'cpu'); tts.tts_to_file(text='Hello Ismael, this is your local voice assistant working.', file_path='test_output.wav'); print('Listo, revisa test_output.wav')"
```

Resultado: genera `test_output.wav` correctamente usando GPU.

## 5. Transcripción de voz (STT): faster-whisper en vez de whisper.cpp

Se evaluaron dos opciones para speech-to-text local:

| Opción | Instalación | Rendimiento en Windows + NVIDIA |
|---|---|---|
| whisper.cpp | Compilar desde cero (VS Build Tools + CUDA Toolkit + CMake + SDL2) | Sin binarios CUDA precompilados oficiales para Windows |
| faster-whisper | `pip install` | Backend CTranslate2, más optimizado en CUDA sobre Windows/NVIDIA que whisper.cpp |

**Decisión:** se eligió `faster-whisper` por simplicidad de instalación (reutiliza el mismo entorno `venv-ia` sin herramientas de compilación adicionales) y mejor rendimiento reportado en esta combinación de sistema operativo y GPU.

```powershell
pip install faster-whisper --trusted-host files.pythonhosted.org --trusted-host pypi.org
```

### Prueba de humo (ciclo completo TTS → STT)

```powershell
python -c "from faster_whisper import WhisperModel; model = WhisperModel('small', device='cuda', compute_type='float16'); segments, info = model.transcribe('test_output.wav'); print('Idioma detectado:', info.language); [print(segment.text) for segment in segments]"
```

Resultado: transcribió correctamente el audio generado por Coqui TTS, confirmando el ciclo completo texto → voz → texto.

## 6. LLM: Ollama + Phi-3:14b

Ya estaba instalado previamente. Verificación:

```powershell
ollama list
ollama --version
```

Debe aparecer `phi3:14b` en la lista de modelos disponibles.

## Estado final del pipeline

| Componente | Herramienta | Estado |
|---|---|---|
| LLM | Ollama + Phi-3:14b | ✅ Funcionando |
| TTS (texto→voz) | Coqui TTS 0.27.5 (transformers==4.57.6) | ✅ Funcionando con GPU |
| STT (voz→texto) | faster-whisper (modelo `small`, CUDA, float16) | ✅ Funcionando con GPU |

## Próximo paso

Firmware del ESP32 (Arduino IDE): configuración de WiFi, mapa de pines para TFT ILI9341 + INMP441 + PAM8403, y comunicación con este pipeline vía WebSocket.