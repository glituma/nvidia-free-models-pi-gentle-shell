# Modelos gratuitos de NVIDIA en Gentle Shell

🇬🇧 [Read in English](README.en.md)

Un perfil dedicado de `gentle-shell` que usa solo cuatro modelos gratuitos de NVIDIA Build, pensado para proyectos MVP y de prueba.

> **Plataforma:** macOS con la shell fish. El paso del Llavero (3) y el alias (7) son específicos de macOS/fish.

| Modelo | ID | Contexto | Salida máx. | Imágenes |
|---|---|---|---|---|
| Kimi K3 (por defecto) | `moonshotai/kimi-k3` | 1M | 131K | sí |
| GLM 5.3 | `z-ai/glm-5.3` | 1M | 131K | no |
| GLM 5.3 Flash | `z-ai/glm-5.3-flash` | 1M | 131K | sí |
| DeepSeek V4.1 Flash | `deepseek-ai/deepseek-v4.1-flash` | 131K* | 16K* | no |

\* Valores conservadores definidos manualmente. pi no incluye metadatos oficiales para este modelo.

---

## 0. Requisitos previos

- Node.js y npm
- [pi](https://www.npmjs.com/package/@earendil-works/pi-coding-agent) y [Gentle Shell](https://github.com/Gentleman-Programming/gentle-shell):

```fish
npm install -g @earendil-works/pi-coding-agent gentle-pi
```

Probado con pi `0.87.1` y gentle-pi `3.7.0`.

## 1. Crear la cuenta de NVIDIA y la API key

1. Entra a <https://build.nvidia.com> e inicia sesión o crea una cuenta de NVIDIA (no pide tarjeta de crédito).
2. Verifica tu correo si te lo solicita.
3. Abre la página de cualquier modelo (por ejemplo, Kimi K3) y haz clic en **Get API Key**, o ve a la sección de API keys de tu cuenta.
4. Genera una key. Empieza con `nvapi-`. Cópiala en ese momento, porque puede que no vuelva a mostrarse.

**Datos del endpoint**

- URL base: `https://integrate.api.nvidia.com/v1`
- Formato: compatible con OpenAI (`/chat/completions`)
- Una sola key sirve para todos los modelos del catálogo.

**Límites y advertencias**

- El nivel gratuito es para prototipos y tiene un límite de peticiones por minuto. Para producción se necesita un plan de pago o NIM autoalojado.
- **Hay colas.** En horas de alta demanda las respuestas pueden tardar minutos o no llegar. En nuestras pruebas del 26-09-2026, GLM 5.3 Flash tardó 78 s en responder "OK" y DeepSeek V4.1 Flash no respondió en 4 minutos. Kimi K3 respondió con normalidad.
- Los prompts pasan por los servidores de NVIDIA. Nunca envíes datos sensibles como datos de clientes, credenciales o información financiera.

## 2. Verificar que los modelos están disponibles

El catálogo de modelos es público, así que esta comprobación no necesita key:

```fish
curl -s https://integrate.api.nvidia.com/v1/models \
  | python3 -c "import json,sys; [print(m['id']) for m in json.load(sys.stdin)['data'] if any(k in m['id'] for k in ['kimi','glm','deepseek'])]"
```

Debe incluir:

```text
deepseek-ai/deepseek-v4.1-flash
moonshotai/kimi-k3
z-ai/glm-5.3
z-ai/glm-5.3-flash
```

## 3. Guardar la API key en el Llavero de macOS

La key nunca se escribe en un archivo de configuración. pi la lee del Llavero en cada petición.

```fish
security add-generic-password -a $USER -s nvidia-api-key -w
```

El comando pide la key sin mostrarla en pantalla.

Comandos útiles del Llavero:

```fish
# Comprobar que se guardó (muestra la key; evítalo si compartes pantalla)
security find-generic-password -a $USER -s nvidia-api-key -w

# Reemplazar la key (rotación)
security add-generic-password -U -a $USER -s nvidia-api-key -w

# Eliminar la key
security delete-generic-password -a $USER -s nvidia-api-key
```

## 4. Probar la API directamente (opcional)

```fish
set -x NVIDIA_API_KEY (security find-generic-password -a $USER -s nvidia-api-key -w)

curl -s https://integrate.api.nvidia.com/v1/chat/completions \
  -H "Authorization: Bearer $NVIDIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"moonshotai/kimi-k3","messages":[{"role":"user","content":"Hola, ¿quién eres?"}],"max_tokens":200}'

set -e NVIDIA_API_KEY
```

## 5. Cómo funcionan los perfiles en Gentle Shell

`gentle-shell` envuelve a `pi`. Cada perfil es un **agent home**: un directorio con su propia configuración, modelos, credenciales y sesiones.

| Home | Ruta | Se selecciona con |
|---|---|---|
| Aislado (por defecto) | `~/.gentle-shell/agent` | `gentle-shell` o `--isolated` |
| pi estándar | `~/.pi/agent` | `--link` |
| Personalizado | cualquier directorio | `--home <ruta>` |

Nuestro perfil es un home personalizado en `~/.gentle-shell/nvidia`. Está totalmente aislado: no hereda credenciales, sesiones ni proveedores de los otros perfiles.

## 6. Crear el perfil

Instalación rápida desde este repositorio:

```fish
git clone https://github.com/glituma/nvidia-free-models-pi-gentle-shell.git
mkdir -p ~/.gentle-shell/nvidia
chmod 700 ~/.gentle-shell/nvidia
cp nvidia-free-models-pi-gentle-shell/profile/*.json ~/.gentle-shell/nvidia/
```

A continuación se explican los dos archivos.

### `profile/models.json`

pi ya incluye un proveedor `nvidia` con metadatos oficiales para Kimi K3, GLM 5.3 y GLM 5.3 Flash (tamaño de contexto, razonamiento, soporte de imágenes). Este archivo solo:

1. Entrega la API key mediante un comando del Llavero (el `!` inicial le indica a pi que ejecute un comando).
2. Agrega DeepSeek V4.1 Flash, que no viene incluido. Sus opciones `compat` replican las de los modelos NVIDIA integrados.

```json
{
  "providers": {
    "nvidia": {
      "apiKey": "!security find-generic-password -a \"$USER\" -s nvidia-api-key -w",
      "models": [
        {
          "id": "deepseek-ai/deepseek-v4.1-flash",
          "name": "DeepSeek V4.1 Flash",
          "headers": { "NVCF-POLL-SECONDS": "3600" },
          "reasoning": true,
          "input": ["text"],
          "contextWindow": 131072,
          "maxTokens": 16384,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
          "compat": {
            "supportsStore": false,
            "supportsDeveloperRole": false,
            "supportsReasoningEffort": false,
            "maxTokensField": "max_tokens",
            "supportsStrictMode": false
          }
        }
      ]
    }
  }
}
```

> No redefinas aquí los tres modelos integrados. Una entrada en `models` reemplaza la definición integrada, y los metadatos oficiales son mejores que valores escritos a mano.

### `profile/settings.json`

`enabledModels` limita la selección inicial y el cambio con `Ctrl+P` a los cuatro modelos, aunque el proveedor `nvidia` integrado expone unos 80.

```json
{
  "defaultProvider": "nvidia",
  "defaultModel": "moonshotai/kimi-k3",
  "enabledModels": [
    "nvidia/moonshotai/kimi-k3",
    "nvidia/z-ai/glm-5.3",
    "nvidia/z-ai/glm-5.3-flash",
    "nvidia/deepseek-ai/deepseek-v4.1-flash"
  ],
  "theme": "Gentleman-Cute"
}
```

### Validar el perfil

```fish
gentle-shell --home ~/.gentle-shell/nvidia --list-models | grep -E "provider|kimi-k3|glm-5.3|deepseek-v4.1"
```

Salida esperada:

```text
provider  model                            context  max-out  thinking  images
nvidia    deepseek-ai/deepseek-v4.1-flash  131.1K   16.4K    yes       no
nvidia    moonshotai/kimi-k3               1.0M     131.1K   yes       yes
nvidia    z-ai/glm-5.3                     1M       131.1K   yes       no
nvidia    z-ai/glm-5.3-flash               1M       131.1K   yes       yes
```

## 7. Crear el alias

```fish
alias pinv 'gentle-shell --home ~/.gentle-shell/nvidia'
funcsave pinv
```

`funcsave` lo guarda en `~/.config/fish/functions/pinv.fish`, así que queda disponible en cada nueva terminal.

## 8. Uso diario

```fish
cd ~/dev/mi-mvp
pinv
```

Dentro de la sesión:

| Acción | Cómo |
|---|---|
| Cambiar entre los 4 modelos | `Ctrl+P` |
| Elegir un modelo de la lista | `/model` |
| Cambiar el nivel de razonamiento | `/thinking` |
| Guardar el modelo actual por defecto | `Ctrl+S` en `/model` |

Uso sugerido:

- **Kimi K3**: programación general y trabajo como agente (por defecto).
- **GLM 5.3**: segunda opinión en tareas complejas.
- **GLM 5.3 Flash / DeepSeek V4.1 Flash**: iteraciones rápidas y cambios simples.

## Solución de problemas

| Síntoma | Causa probable | Solución |
|---|---|---|
| `401 Unauthorized` | Key ausente o incorrecta en el Llavero | Repite el paso 3 con `-U` |
| El Llavero pide contraseña en cada petición | `security` no tiene acceso permitido | Elige **Permitir siempre** en el aviso |
| `429 Too Many Requests` | Límite del nivel gratuito | Espera un minuto o cambia de modelo |
| El modelo se queda pensando, sin error | Cola del nivel gratuito: los modelos NVIDIA envían `NVCF-POLL-SECONDS: 3600`, así que la petición espera en cola hasta 1 hora en lugar de fallar | Cancela con `Esc`, cambia de modelo con `Ctrl+P` y vuelve a probar más tarde |
| Un modelo no aparece en `/model` | Error en `enabledModels` o el modelo salió del catálogo | Repite la comprobación del paso 2 |
| Respuestas de DeepSeek cortadas | `maxTokens` conservador | Sube `maxTokens` en `models.json` cuando se conozcan los límites reales |

## Desinstalación

```fish
rm -rf ~/.gentle-shell/nvidia
functions -e pinv; rm -f ~/.config/fish/functions/pinv.fish
security delete-generic-password -a $USER -s nvidia-api-key
```

## Licencia

[MIT](LICENSE)
