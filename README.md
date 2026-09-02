# ☁️ Oracle A1 Hunter

[![Oracle A1 Hunter](https://github.com/erpereh/oracle-a1-hunter/actions/workflows/oracle-a1.yml/badge.svg)](https://github.com/erpereh/oracle-a1-hunter/actions/workflows/oracle-a1.yml)
![Oracle Cloud](https://img.shields.io/badge/Oracle%20Cloud-A1-F80000?logo=oracle&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-runner-2088FF?logo=githubactions&logoColor=white)
![cron-job.org](https://img.shields.io/badge/cron--job.org-scheduler-E34F26)
![Telegram](https://img.shields.io/badge/Telegram-notifications-26A5E4?logo=telegram&logoColor=white)

Automatiza los reintentos para crear una instancia **Oracle Cloud Ampere A1** cuando Oracle devuelve `Out of host capacity`.

La arquitectura usa **cron-job.org como scheduler**, **GitHub Actions como runner** y **Oracle Resource Manager** para lanzar los `Apply`. El intento se ejecuta cada ~5 minutos aunque tu PC esté apagado. Si la VM se crea, recibes un aviso por Telegram y el workflow de GitHub se desactiva automáticamente.

> [!IMPORTANT]
> Este proyecto **no aumenta la capacidad de Oracle ni garantiza que consigas una VM**. Solo automatiza los reintentos de un Stack válido hasta que exista capacidad. Revisa siempre las condiciones actuales de Oracle Cloud Free Tier / Always Free y los límites de tu cuenta.

---

## ✨ Qué hace

- 🔁 Reintenta el Stack cada **5 minutos** mediante cron-job.org.
- 🧠 Comprueba el último Job antes de lanzar otro `Apply`.
- ✅ Reintenta automáticamente solo ante errores de capacidad.
- ⏱️ Trata `429 TooManyRequests` de OCI como un límite temporal y espera al siguiente ciclo en lugar de marcar el run como fallo.
- 🚫 Usa `--no-retry` al crear el nuevo Job para evitar que el CLI de OCI consuma minutos haciendo reintentos internos cuando Oracle está limitando peticiones.
- 🛑 Si aparece un error diferente, detiene ese intento y te avisa.
- 📲 Envía un informe diario por Telegram si sigue sin haber suerte.
- 🎉 Te avisa inmediatamente cuando Oracle devuelve `SUCCEEDED`.
- 💤 Desactiva el workflow de GitHub cuando consigue la VM.
- 💻 No necesitas dejar tu ordenador encendido.
- 🔀 El retry y el informe diario usan **grupos de concurrencia separados**, por lo que no se bloquean entre sí.

```mermaid
flowchart TD
    A[cron-job.org] -->|cada 5 min| B[GitHub Actions: hunt]
    A -->|14:00 Europe/Madrid| C[GitHub Actions: daily-report]
    B --> D[Oracle Resource Manager]
    D --> E{Último Job}
    E -->|IN_PROGRESS| F[Esperar]
    E -->|FAILED| G{¿Out of capacity?}
    G -->|Sí| H[Nuevo Apply]
    H -->|Aceptado| D
    H -->|429 TooManyRequests| F
    G -->|No| I[Telegram ⚠️]
    E -->|SUCCEEDED| J[Telegram 🎉]
    J --> K[Desactivar workflow]
    C --> L[Telegram ⏳]
```

---

# 🚀 Quick Start

Si ya tienes un Stack de Oracle Resource Manager que intenta crear una `VM.Standard.A1.Flex`:

1. Haz **Fork** de este repositorio.
2. Crea una **API Key** de Oracle Cloud.
3. Añade los **8 GitHub Secrets** indicados más abajo.
4. Configura Telegram.
5. Crea un **Fine-grained Personal Access Token** de GitHub limitado a este repositorio.
6. Crea dos jobs en **cron-job.org**:
   - Retry → cada 5 minutos.
   - Daily Report → cada día a las 14:00 `Europe/Madrid`.
7. Haz una ejecución de prueba de ambos.
8. Déjalo funcionando.

> [!NOTE]
> Este repositorio **no usa `schedule:` de GitHub Actions**. Los cron internos de GitHub pueden sufrir retrasos o saltarse ejecuciones, por eso la programación se delega a cron-job.org y GitHub se usa únicamente como runner.

---

# 📖 Instalación desde cero

## 1. Intentar crear la instancia A1

En Oracle Cloud:

**Compute → Instances → Create instance**

La shape que busca este proyecto es:

```text
VM.Standard.A1.Flex
```

Usa una imagen compatible con **ARM64 / Ampere A1** y configura CPU, RAM, almacenamiento, red y SSH según tus necesidades y los límites de tu cuenta.

Si Oracle tiene capacidad, la instancia se creará y no necesitas este proyecto.

Si recibes algo parecido a:

```text
Out of capacity for shape VM.Standard.A1.Flex
```

o:

```text
500-InternalError, Out of host capacity.
```

continúa con el siguiente paso.

---

## 2. Guardar la configuración como Stack

En la pantalla de revisión pulsa:

**Save as stack**

Puedes llamarlo, por ejemplo:

```text
oracle-a1-auto
```

Cuando Oracle pregunte:

```text
Run apply on the created stack?
```

puedes dejarlo desmarcado.

Después ve a:

**Developer Services → Resource Manager → Stacks → tu Stack**

En **Details**, copia el OCID del Stack:

```text
ocid1.ormstack.oc1.<region>....
```

Ese valor será `OCI_STACK_ID`.

---

## 3. Crear una API Key de OCI

En Oracle Cloud abre tu perfil y ve a:

**My profile / User settings → API Keys → Add API Key**

Puedes usar **Generate API Key Pair**.

Oracle mostrará una configuración parecida a:

```ini
[DEFAULT]
user=ocid1.user.oc1...
fingerprint=aa:bb:cc:dd:...
tenancy=ocid1.tenancy.oc1...
region=eu-madrid-1
key_file=<path to your private keyfile>
```

También descargarás una clave privada `.pem`.

> [!CAUTION]
> La clave privada `.pem` es sensible. **No la subas al repositorio, no la publiques y no la compartas.**

---

## 4. Crear los GitHub Secrets

En tu fork entra en:

**Settings → Secrets and variables → Actions → Repository secrets**

Crea estos 8 Secrets:

| Secret | Valor |
|---|---|
| `OCI_USER_OCID` | Valor `user=...` de Oracle |
| `OCI_TENANCY_OCID` | Valor `tenancy=...` |
| `OCI_FINGERPRINT` | Fingerprint de la API Key |
| `OCI_PRIVATE_KEY` | Contenido completo de la clave privada `.pem` |
| `OCI_REGION` | Región, por ejemplo `eu-madrid-1` |
| `OCI_STACK_ID` | OCID del Stack de Resource Manager |
| `TELEGRAM_BOT_TOKEN` | Token de BotFather |
| `TELEGRAM_CHAT_ID` | ID de tu chat con el bot |

La private key debe pegarse completa:

```text
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

> [!WARNING]
> Si empieza por `-----BEGIN PUBLIC KEY-----`, estás usando la clave equivocada.

---

# 📲 Telegram

## 5. Crear el bot

En Telegram abre **@BotFather** y envía:

```text
/newbot
```

Sigue los pasos y guarda el token. Ese valor va en:

```text
TELEGRAM_BOT_TOKEN
```

No publiques el token.

---

## 6. Obtener `TELEGRAM_CHAT_ID`

Abre el bot, pulsa **Start** y envíale un mensaje, por ejemplo `hola`.

En PowerShell:

```powershell
$TOKEN = "TU_TOKEN"

Invoke-RestMethod "https://api.telegram.org/bot$TOKEN/getUpdates" |
    ConvertTo-Json -Depth 10
```

Busca:

```json
"chat": {
  "id": 123456789,
  "type": "private"
}
```

Ese número es `TELEGRAM_CHAT_ID`.

### Probar Telegram

```powershell
$TOKEN = "TU_TOKEN"
$CHAT_ID = "TU_CHAT_ID"

Invoke-RestMethod `
  -Uri "https://api.telegram.org/bot$TOKEN/sendMessage" `
  -Method Post `
  -Body @{
      chat_id = $CHAT_ID
      text    = "✅ Oracle A1 Hunter conectado correctamente"
  }
```

---

# ⏱️ Scheduler fiable con cron-job.org

## 7. Crear un Fine-grained PAT de GitHub

En GitHub:

**Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**

Configuración recomendada:

```text
Token name: Oracle A1 Hunter Scheduler
Repository access: Only select repositories
Repository: tu fork de oracle-a1-hunter
Repository permissions:
  Actions: Read and write
```

> [!CAUTION]
> El PAT es una credencial. No lo guardes en el repositorio ni lo publiques. cron-job.org lo almacenará como header `Authorization`.

---

## 8. Crear el job de reintento

En **cron-job.org**, crea:

```text
Título: Oracle A1 Hunter - Retry
Horario: cada 5 minutos
Zona horaria: Europe/Madrid
Método: POST
```

URL:

```text
https://api.github.com/repos/TU_USUARIO/oracle-a1-hunter/actions/workflows/oracle-a1.yml/dispatches
```

Headers:

```text
Accept: application/vnd.github+json
Authorization: Bearer TU_GITHUB_PAT
X-GitHub-Api-Version: 2026-03-10
Content-Type: application/json
```

Body:

```json
{
  "ref": "main",
  "inputs": {
    "test_daily_report": false
  }
}
```

Guarda el job y usa **Ejecutar prueba**. Después comprueba que aparece un nuevo run en:

**GitHub → Actions → Oracle A1 Hunter**

---

## 9. Crear el informe diario

Duplica el job anterior y cambia:

```text
Título: Oracle A1 Hunter - Daily Report
Horario: todos los días a las 14:00
Zona horaria: Europe/Madrid
```

URL, método y headers son iguales.

Cambia el body a:

```json
{
  "ref": "main",
  "inputs": {
    "test_daily_report": true
  }
}
```

Una ejecución de prueba debe ejecutar `daily-report` y enviarte inmediatamente el mensaje de Telegram.

La configuración final queda así:

```text
cron-job.org
├── Retry
│   └── cada 5 min
│       └── test_daily_report: false
│           └── GitHub Actions: hunt
│
└── Daily Report
    └── 14:00 Europe/Madrid
        └── test_daily_report: true
            └── GitHub Actions: daily-report
```

---

# ▶️ Ejecución manual

También puedes iniciar el workflow desde GitHub:

**Actions → Oracle A1 Hunter → Run workflow**

Para un retry normal, deja desmarcada la opción del informe diario.

Para probar Telegram, marca:

```text
✅ Enviar ahora una prueba del aviso diario de Telegram
```

Los dos modos usan grupos de concurrencia independientes:

```text
oracle-a1-hunter-retry
oracle-a1-hunter-daily
```

Por tanto, el informe diario no queda bloqueado porque haya un retry en curso.

---

# 🧠 Cómo decide si reintenta

El workflow consulta el último Job de Resource Manager.

Si está en:

```text
ACCEPTED
IN_PROGRESS
CANCELING
```

espera al siguiente retry.

Si está en `FAILED`, descarga los logs y solo vuelve a lanzar Terraform si encuentra:

```text
out of host capacity
```

o:

```text
out of capacity
```

Cuando intenta crear el siguiente Job de Resource Manager, ejecuta el OCI CLI con `--no-retry`. Si Oracle responde:

```text
429 TooManyRequests
```

o:

```text
Too many requests for the tenant
```

el workflow lo considera un **throttle temporal**, termina el run correctamente y espera al siguiente disparo de cron-job.org (~5 minutos). De esta forma un límite temporal de la API no aparece como un fallo real ni mantiene el runner ocupado con reintentos internos del SDK.

Si el fallo es distinto, detiene ese run y envía una alerta por Telegram.

---

# 🎉 Cuando consigue la VM

Cuando el último Job devuelve:

```text
SUCCEEDED
```

Oracle A1 Hunter:

1. Te envía un Telegram de éxito.
2. No lanza otro `Apply`.
3. Desactiva automáticamente `oracle-a1.yml` en GitHub.

El mensaje será parecido a:

```text
🎉 ¡Oracle A1 conseguida!

El Stack de Oracle ha terminado correctamente.
Región: eu-madrid-1
Estado: SUCCEEDED

A1 Hunter va a desactivar el workflow de GitHub.
Puedes desactivar también los dos jobs de cron-job.org.
```

> [!NOTE]
> cron-job.org seguirá intentando llamar al endpoint aunque el workflow de GitHub esté desactivado. Cuando recibas el aviso de éxito, desactiva también los dos jobs de cron-job.org para dejar de hacer peticiones innecesarias.

---

# 🕑 Informe diario

El horario lo controla **cron-job.org**, no GitHub Actions.

Si configuras:

```text
14:00
Europe/Madrid
```

cron-job.org se encarga del cambio entre horario de verano e invierno.

El mensaje incluye el último estado, número de Jobs registrados, región y fecha.

Ejemplo:

```text
⏳ Oracle A1: todavía sin suerte hoy.

Último estado: FAILED
Intentos registrados: 42
Región: eu-madrid-1
Fecha: 02/09/2026 14:00

El bot sigue intentando automáticamente cada ~5 minutos.
Te avisaré en cuanto lo consiga.
```

---

# 🛡️ Seguridad

Nunca guardes credenciales directamente en `.github/workflows/oracle-a1.yml`.

No publiques:

- `OCI_PRIVATE_KEY`
- `TELEGRAM_BOT_TOKEN`
- el Fine-grained PAT de GitHub
- claves privadas SSH
- archivos `.pem` privados

Si expones alguna credencial:

1. Revócala inmediatamente.
2. Genera una nueva.
3. Actualiza GitHub Secrets o los headers de cron-job.org.

Para el PAT del scheduler usa el mínimo alcance posible: **solo este repositorio** y **Actions: Read and write**.

---

# 🧰 Troubleshooting

## `Out of host capacity`

```text
500-InternalError, Out of host capacity.
```

✅ Es el caso esperado. En el siguiente ciclo cron-job.org volverá a disparar el workflow.

### `429 TooManyRequests`

Puedes ver un mensaje parecido a:

```text
code: TooManyRequests
message: Too many requests for the tenant
status: 429
```

✅ También es un caso transitorio. Significa que Oracle Resource Manager está aplicando rate limiting al intento de crear el siguiente Job.

Oracle A1 Hunter usa `--no-retry` para no quedarse varios minutos reintentando dentro del mismo runner. Ese ciclo termina en verde y cron-job.org vuelve a intentarlo aproximadamente 5 minutos después.

No necesitas cambiar tus Secrets ni reiniciar nada por un `429` aislado.

### No recibo ejecuciones cada 5 minutos

Comprueba el historial de **cron-job.org**, no el scheduler de GitHub. Este repositorio no incluye un `schedule:` interno.

El job `Retry` debe estar habilitado con:

```text
*/5 * * * *
```

y `Europe/Madrid` como zona horaria.

### cron-job.org devuelve 401 / 403

Revisa:

```text
Authorization: Bearer TU_GITHUB_PAT
```

El PAT debe tener acceso al fork correcto y permiso:

```text
Actions: Read and write
```

### cron-job.org responde correctamente pero no veo el run

Comprueba que la URL contiene tu usuario y el nombre exacto del workflow:

```text
.../actions/workflows/oracle-a1.yml/dispatches
```

Y que el body usa:

```json
{"ref":"main","inputs":{"test_daily_report":false}}
```

### `The provided key is not a private key`

`OCI_PRIVATE_KEY` debe contener el `.pem` privado completo:

```text
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

No uses la public key.

### `OCI authentication` falla

Comprueba:

- `OCI_USER_OCID`
- `OCI_TENANCY_OCID`
- `OCI_FINGERPRINT`
- `OCI_PRIVATE_KEY`
- `OCI_REGION`

Y que la API Key pública está registrada en tu usuario de Oracle.

### `daily-report` aparece como `Skipped`

Ese run fue lanzado con:

```text
test_daily_report: false
```

Para el informe diario el body debe llevar:

```text
test_daily_report: true
```

### El workflow está `pending`

Los retries y los informes diarios ya usan concurrencias diferentes. Si ves un `pending`, revisa si existen varios runs del **mismo tipo** todavía en cola.

### El workflow está desactivado

Si ya consiguió la VM, es el comportamiento esperado.

Si quieres volver a utilizarlo, entra en **Actions**, habilita otra vez el workflow y reactiva los jobs de cron-job.org.

---

# ❓ FAQ

### ¿Tengo que dejar mi PC encendido?

No. cron-job.org dispara GitHub Actions y GitHub se comunica con Oracle Cloud.

### ¿El repositorio crea toda la infraestructura desde cero?

No. Debes tener primero un **Stack válido en Oracle Resource Manager**. El proyecto automatiza los `Apply` sobre ese Stack.

### ¿Puede crear varias VMs por accidente?

El workflow comprueba el último Job. Cuando detecta `SUCCEEDED`, no lanza otro Apply y desactiva el workflow.

### ¿Conseguiré una A1 seguro?

No. Depende completamente de la capacidad disponible y de las restricciones de tu cuenta/región.

### ¿Puedo usar otra región?

Sí. Cambia `OCI_REGION` y usa un Stack creado en esa región.

### ¿Puedo cambiar CPU y RAM?

Sí. Esa configuración está en tu Stack de Oracle, no en este workflow. Comprueba antes límites y precios aplicables a tu cuenta.

### ¿Por qué no usar directamente el cron de GitHub Actions?

Porque los workflows `schedule` de GitHub no garantizan ejecución exacta y pueden sufrir retrasos. Este proyecto usa un scheduler externo para separar la programación del runner.

### ¿Puedo cambiar cada cuánto reintenta?

Sí. Cambia únicamente el horario de **Oracle A1 Hunter - Retry** en cron-job.org. No necesitas editar el YAML.

---

# 📚 Documentación útil

- [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/)
- [Oracle Resource Manager](https://docs.oracle.com/en-us/iaas/Content/ResourceManager/home.htm)
- [OCI CLI - Resource Manager Jobs](https://docs.oracle.com/en-us/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/resource-manager/job.html)
- [GitHub Actions - workflow_dispatch](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch)
- [GitHub REST API - Workflows](https://docs.github.com/rest/actions/workflows)
- [cron-job.org](https://cron-job.org/)
- [Telegram Bot API](https://core.telegram.org/bots/api)

---

## ⭐ ¿Te ha servido?

Si este proyecto te ha ayudado, puedes dejar una ⭐ al repositorio.

PRs, mejoras y sugerencias son bienvenidas.

---

> Este proyecto es independiente y no está afiliado, patrocinado ni respaldado por Oracle, GitHub, cron-job.org o Telegram.