# OpenClaw en WSL2: laboratorio de agente de IA personal

Documentación del montaje de [OpenClaw](https://openclaw.ai) sobre Windows 11 con WSL2 (Ubuntu), conectado a un modelo vía suscripción de ChatGPT y accesible desde Telegram.

Versión instalada: `2026.9.5`
Entorno: Windows 11 + WSL2 (Ubuntu) + Node.js 24
Fecha del montaje: octubre 2026

---

## Qué es OpenClaw

OpenClaw es un asistente de IA de código abierto que corre en una máquina propia y se usa desde aplicaciones de chat (Telegram, WhatsApp, Discord, Slack, Signal y otras). A diferencia de un chat web, es un **agente**: ejecuta comandos, lee y escribe archivos, navega la web y conserva memoria entre conversaciones.

No tiene suscripción ni versión hospedada. El software es gratuito; el costo lo pone el modelo de IA que se conecte (API key o suscripción existente).

### Conceptos

| Concepto | Qué es |
|---|---|
| **Gateway** | El proceso central. Recibe mensajes, llama al modelo y ejecuta herramientas. |
| **Modelo** | La IA que razona. OpenClaw no trae una propia; se conecta a Anthropic, OpenAI, OpenRouter o modelos locales. |
| **Canales** | Por dónde se le habla: Telegram, WhatsApp, Discord, Slack y demás. |
| **Memoria** | Persiste preferencias, proyectos y contexto entre sesiones. |
| **Plugins y skills** | Capacidades extra: navegador, transferencia de archivos, geolocalización, voz, cron. |

---

## Advertencia de seguridad

El agente ejecuta comandos y toca archivos del sistema. **No instalar en equipos corporativos, ni en máquinas con acceso a datos sensibles o a redes institucionales.** Este laboratorio corre en una laptop personal.

Además:

- El token del bot de Telegram y el token del Gateway son credenciales. No se comparten ni se suben al repositorio.
- La política de emparejamiento (`pairing`) debe quedar activa. Nunca usar `dmPolicy: "open"` con `allowFrom: ["*"]`, porque permitiría que cualquier cuenta de Telegram que encuentre el bot le dé órdenes al agente.
- El Gateway queda en `loopback` (127.0.0.1), accesible solo desde la propia máquina.

---

## Requisitos

- Windows 10/11 con WSL2 habilitado
- Node.js 24.16 o superior (26 recomendado)
- Una suscripción de ChatGPT o una API key de algún proveedor
- 8 GB de RAM como mínimo (16 GB es más cómodo)

---

## Instalación paso a paso

### 1. Instalar Ubuntu en WSL2

Desde PowerShell como administrador:

```powershell
wsl --install -d Ubuntu
```

Verificar que quedó en versión 2:

```powershell
wsl -l -v
```

La columna `VERSION` debe decir `2`. Si dice `1`:

```powershell
wsl --set-version Ubuntu 2
```

### 2. Habilitar systemd

Es obligatorio para que el Gateway quede como servicio de arranque automático. Dentro de Ubuntu:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

Reiniciar WSL desde PowerShell:

```powershell
wsl --shutdown
```

Volver a abrir Ubuntu y verificar:

```bash
ps -p 1 -o comm=
```

Debe devolver `systemd`.

### 3. Instalar Node.js a nivel de sistema

OpenClaw funciona con Node instalado por nvm, pero el servicio advierte que puede romperse al cambiar de versión. Instalarlo a nivel de sistema evita ese problema:

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt-get install -y nodejs
/usr/bin/node --version
```

### 4. Instalar OpenClaw

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

El instalador lanza el asistente de onboarding automáticamente.

### 5. Onboarding

En el asistente:

1. **Flujo**: elegir `Quick start` para una configuración rápida, o `Custom setup` para el flujo guiado completo.
2. **Proveedor de modelo**: elegir `OpenAI` y, como método de autenticación, la opción de **suscripción de ChatGPT / Codex (OAuth)**, no la de API key. Esa opción usa la suscripción ya pagada en lugar de cobrar por uso.
3. **Token del Gateway**: elegir modo `Token` y dejar que OpenClaw **genere** uno. No es una contraseña existente; es una credencial nueva que protege el acceso al Gateway. Se guarda en `~/.openclaw/openclaw.json`.

Si el login por navegador no retorna a la terminal (ocurre en WSL porque el navegador está en Windows), usar el flujo de código de dispositivo:

```bash
openclaw models auth login --provider openai --device-code
```

Para rehacer la configuración en cualquier momento:

```bash
openclaw configure
```

### 6. Instalar el Gateway como servicio

```bash
openclaw gateway install
openclaw gateway status
```

El estado correcto muestra:

```
Runtime: running
Connectivity probe: ok
Listening: 127.0.0.1:18789
```

Si queda alguna advertencia de `Service config issue` relacionada con nvm, regenerar el servicio con el Node del sistema:

```bash
openclaw gateway install --force
```

### 7. Abrir el dashboard

```bash
openclaw dashboard --no-open
```

Copia al portapapeles una URL con emparejamiento de un solo uso. Pegarla en el navegador de Windows.

**Error frecuente:** si la pantalla de conexión muestra `wss://127.0.0.1:18789`, cambiarlo a `ws://`. El prefijo `wss://` es para TLS, y un Gateway local no lo usa.

---

## Conectar Telegram

### 1. Crear el bot

En Telegram, abrir **@BotFather** y enviar `/newbot`. Pide un nombre visible y un username terminado en `bot`. Al final entrega el token.

### 2. Registrar el token

```bash
openclaw channels add --channel telegram --token <TOKEN_DEL_BOT>
```

Telegram no usa `openclaw channels login`. El token se configura y listo.

### 3. Reiniciar el Gateway

El canal nuevo solo se activa cuando el Gateway recarga la configuración:

```bash
openclaw gateway restart
```

Confirmar en los logs que el proveedor arrancó:

```bash
openclaw logs --follow
```

Debe aparecer una línea similar a `channels/telegram [default] starting provider (@tu_bot)`.

### 4. Emparejar la cuenta

Enviar cualquier mensaje al bot desde Telegram. El código de emparejamiento **se genera con ese primer mensaje**, no antes. Luego:

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODIGO>
```

Al aprobar, OpenClaw registra la cuenta y la configura como owner de comandos si la lista estaba vacía:

```
Approved telegram sender 34824560.
Command owner configured telegram:34824560 (commands.ownerAllowFrom was empty).
```

A partir de ahí el bot responde desde el celular.

---

## Operación diaria

### Iniciar

Abrir Ubuntu desde el menú Inicio (o `wsl -d Ubuntu` desde PowerShell). El servicio arranca solo con systemd.

```bash
openclaw gateway status
openclaw dashboard --no-open
```

### Detener

| Acción | Comando |
|---|---|
| Detener solo OpenClaw | `systemctl --user stop openclaw-gateway` |
| Iniciar OpenClaw | `systemctl --user start openclaw-gateway` |
| Apagar WSL completo | `wsl --shutdown` (desde PowerShell) |

### Diagnóstico

```bash
openclaw doctor          # chequeos guiados
openclaw status          # estado general
openclaw logs --follow   # logs en vivo
openclaw update          # actualizar versión
```

---

## Problemas encontrados y soluciones

| Problema | Causa | Solución |
|---|---|---|
| `which openclaw` no devuelve nada desde PowerShell | nvm solo carga en shells interactivos | Usar `wsl -d Ubuntu -- bash -lc "..."` o entrar a la terminal |
| Dos distribuciones de Ubuntu y no se sabe cuál tiene la instalación | WSL usa el hostname del PC en todas | Buscar `~/.nvm/versions/node/*/bin/openclaw` en cada una |
| Dashboard: "No se puede acceder al Gateway" | La URL usaba `wss://` | Cambiar a `ws://127.0.0.1:18789` o usar la URL de `dashboard --no-open` |
| `pairing list` vacío | El código se genera con el primer mensaje al bot | Escribirle al bot primero |
| `Plugin command "/dashboard" conflicts` | Un plugin registra un comando que Telegram ya ocupa | No afecta la conversación; solo ese comando no está disponible |
| Advertencias de `Service config issue` con nvm | El servicio apunta al Node de nvm | Instalar Node a nivel de sistema y `openclaw gateway install --force` |

---

## Limitaciones de este montaje

El Gateway corre dentro de WSL en una laptop. El bot solo responde mientras la laptop esté encendida y WSL activo. No hace falta tener la ventana de Ubuntu abierta, porque el servicio corre en segundo plano, pero WSL se apaga tras un rato sin sesiones.

Para disponibilidad 24/7, el siguiente paso natural es mover el Gateway a un equipo que permanezca encendido: una Raspberry Pi, un mini PC o una VM en un servidor. El procedimiento es el mismo.

---

## Siguientes pasos

- [ ] Programar una tarea recurrente con el cron integrado
- [ ] Explorar skills y plugins en [ClawHub](https://clawhub.ai)
- [ ] Revisar el modelo de permisos antes de ampliar el acceso al sistema
- [ ] Evaluar migración a un host permanente
- [ ] Probar otros canales (WhatsApp, Discord)

---

## Referencias

- [Sitio oficial](https://openclaw.ai)
- [Documentación](https://docs.openclaw.ai)
- [Documentación en español](https://docs.openclaw.ai/es/start/getting-started)
- [Configuración de Telegram](https://docs.openclaw.ai/channels/telegram/setup)
- [Troubleshooting](https://docs.openclaw.ai/troubleshooting)

---

## Nota sobre credenciales

Este repositorio no contiene tokens ni secretos. Los valores sensibles viven en `~/.openclaw/openclaw.json`, fuera del control de versiones. Si se agrega cualquier archivo de configuración al repositorio, revisar antes que no incluya:

- Token del bot de Telegram
- Token del Gateway
- Credenciales OAuth del proveedor de modelo
- IDs de usuario o de grupos de Telegram
