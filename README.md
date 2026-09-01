# ☁️ Oracle A1 Hunter

[![Oracle A1 Hunter](https://github.com/erpereh/oracle-a1-hunter/actions/workflows/oracle-a1.yml/badge.svg)](https://github.com/erpereh/oracle-a1-hunter/actions/workflows/oracle-a1.yml)
![Oracle Cloud](https://img.shields.io/badge/Oracle%20Cloud-A1-F80000?logo=oracle&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-automated-2088FF?logo=githubactions&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram-notifications-26A5E4?logo=telegram&logoColor=white)

Automatiza los reintentos para crear una instancia **Oracle Cloud Ampere A1** cuando la región devuelve `Out of host capacity`.

El workflow usa **GitHub Actions + Oracle Resource Manager** para volver a lanzar el `Apply` aproximadamente cada 5 minutos, incluso con tu PC apagado. Cuando Oracle consigue crear la máquina, recibes un aviso por Telegram y el workflow se desactiva automáticamente.

> [!IMPORTANT]
> Este proyecto **no aumenta la capacidad disponible de Oracle ni garantiza que consigas una VM**. Solo automatiza los reintentos de un Stack válido hasta que Oracle tenga capacidad. Comprueba siempre las condiciones actuales de Oracle Cloud Free Tier / Always Free y los recursos incluidos en tu cuenta antes de crear nada.

---

## ✨ Qué hace

- 🔁 Reintenta el Stack aproximadamente cada **5 minutos**.
- 🧠 Comprueba el resultado anterior antes de lanzar otro `Apply`.
- ✅ Solo reintenta automáticamente si detecta un error de capacidad.
- 🛑 Si aparece un error diferente, no sigue lanzando intentos a ciegas.
- 📲 Envía un aviso diario por Telegram si todavía no ha habido suerte.
- 🎉 Envía un mensaje inmediatamente cuando el Stack termina en `SUCCEEDED`.
- 💤 Desactiva el workflow automáticamente cuando consigue la VM.
- 💻 No necesitas tener tu ordenador encendido.

```mermaid
flowchart TD
    A[GitHub Actions] -->|cada ~5 min| B[Oracle Resource Manager]
    B --> C{Último Job}
    C -->|IN_PROGRESS| D[Esperar siguiente ciclo]
    C -->|FAILED| E{¿Error de capacidad?}
    E -->|Sí| F[Nuevo Terraform Apply]
    E -->|No| G[Parar + aviso de error]
    F --> B
    C -->|SUCCEEDED| H[Telegram 🎉]
    H --> I[Desactivar workflow]
```

---

## 🚀 Quick Start

Si **ya tienes un Stack de Oracle Resource Manager** que intenta crear una `VM.Standard.A1.Flex`, puedes dejar esto funcionando en pocos minutos:

1. Haz **Fork** de este repositorio.
2. Configura una **API Key** en Oracle Cloud.
3. Añade los 8 Secrets indicados abajo.
4. En GitHub, entra en **Actions → Oracle A1 Hunter → Run workflow**.
5. Ejecuta una vez con `test_daily_report` desmarcado.
6. Déjalo funcionando.

Si todavía no tienes el Stack, sigue la guía completa.

---

# 📖 Instalación desde cero

## 1. Crear la instancia A1 en Oracle Cloud

En la consola de Oracle Cloud:

**Compute → Instances → Create instance**

Configura la instancia que quieras crear. Para este proyecto necesitas que el Stack contenga una instancia con la shape:

```text
VM.Standard.A1.Flex
```

Recomendaciones:

- Usa una imagen compatible con ARM64 / Ampere A1.
- Añade tu clave pública SSH si vas a conectarte por SSH.
- Si necesitas acceso desde Internet, configura una VNIC / subnet con IP pública.
- Selecciona CPU, RAM y almacenamiento **dentro de los límites gratuitos que tenga actualmente tu cuenta**.

Si Oracle tiene capacidad, la instancia se creará y no necesitas este proyecto.

Si aparece un error parecido a:

```text
Out of capacity for shape VM.Standard.A1.Flex
```

O:

```text
500-InternalError, Out of host capacity.
```

continúa con el siguiente paso.

---

## 2. Guardar la configuración como Stack

En la pantalla de revisión de creación de la instancia pulsa:

**Save as stack**

Pon un nombre, por ejemplo:

```text
oracle-a1-auto
```

Cuando Oracle pregunte:

```text
Run apply on the created stack?
```

puedes dejarlo **desmarcado**.

Después ve a:

**Developer Services → Resource Manager → Stacks → tu Stack**

En la pestaña **Details**, copia el **OCID del Stack**.

Será algo parecido a:

```text
ocid1.ormstack.oc1.<region>....
```

Lo necesitaremos más adelante como `OCI_STACK_ID`.

---

## 3. Crear una API Key en Oracle Cloud

En Oracle Cloud abre tu perfil y ve a:

**My profile / User settings → API Keys → Add API Key**

Puedes usar **Generate API Key Pair**.

Oracle te mostrará una configuración parecida a:

```ini
[DEFAULT]
user=ocid1.user.oc1...
fingerprint=aa:bb:cc:dd:...
tenancy=ocid1.tenancy.oc1...
region=eu-madrid-1
key_file=<path to your private keyfile>
```

También descargarás una **clave privada `.pem`**.

> [!CAUTION]
> La clave privada `.pem` es sensible. **No la subas al repositorio, no la pongas en el README y no la compartas.** Solo debe guardarse como GitHub Secret.

---

## 4. Hacer Fork del repositorio

Pulsa **Fork** en GitHub y crea tu propia copia.

Después entra en:

**Settings → Secrets and variables → Actions**

En **Repository secrets**, crea los siguientes Secrets.

| Secret | Valor |
|---|---|
| `OCI_USER_OCID` | El valor `user=...` de la configuración de Oracle |
| `OCI_TENANCY_OCID` | El valor `tenancy=...` |
| `OCI_FINGERPRINT` | El `fingerprint` de la API Key |
| `OCI_PRIVATE_KEY` | El contenido completo de tu clave privada `.pem` |
| `OCI_REGION` | Tu región OCI, por ejemplo `eu-madrid-1` |
| `OCI_STACK_ID` | OCID del Stack de Resource Manager |
| `TELEGRAM_BOT_TOKEN` | Token generado por BotFather |
| `TELEGRAM_CHAT_ID` | ID de tu chat de Telegram |

La clave privada debe pegarse completa, incluyendo las cabeceras:

```text
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

> [!WARNING]
> No confundas `OCI_PRIVATE_KEY` con la clave pública. Si empieza por `-----BEGIN PUBLIC KEY-----`, no es la que necesitas.

---

# 📲 Configurar Telegram

Telegram se usa para:

- Avisarte cuando la A1 se ha creado correctamente.
- Enviarte un resumen diario si sigue sin haber capacidad.
- Avisarte si aparece un error distinto a falta de capacidad.

## 5. Crear el bot

En Telegram abre **@BotFather** y envía:

```text
/newbot
```

Sigue los pasos y guarda el token que te devuelve.

Ejemplo:

```text
1234567890:AAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Ese valor será tu Secret:

```text
TELEGRAM_BOT_TOKEN
```

> [!CAUTION]
> El token permite controlar tu bot. No lo publiques.

---

## 6. Obtener tu `TELEGRAM_CHAT_ID`

Abre tu nuevo bot, pulsa **Start** y envíale un mensaje, por ejemplo:

```text
hola
```

Después, desde PowerShell:

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

Ese número es tu:

```text
TELEGRAM_CHAT_ID
```

### Probar Telegram manualmente

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

Si recibes el mensaje, Telegram está preparado.

---

# ▶️ Arrancar Oracle A1 Hunter

En tu fork entra en:

**Actions → Oracle A1 Hunter → Run workflow**

Para un intento normal:

```text
test_daily_report = false
```

GitHub ejecutará el workflow una vez y después los cron programados seguirán trabajando automáticamente.

En Oracle podrás ver los intentos en:

**Developer Services → Resource Manager → Stacks → tu Stack → Jobs**

Es normal ver varios Jobs como:

```text
APPLY    FAILED
APPLY    FAILED
APPLY    FAILED
...
```

si todos fallan por falta de capacidad.

---

# 🧪 Probar el aviso diario de Telegram

No necesitas esperar a la hora programada.

En:

**Actions → Oracle A1 Hunter → Run workflow**

marca:

```text
✅ Enviar ahora una prueba del aviso diario de Telegram
```

En ese modo:

- `hunt` se omite.
- `daily-report` se ejecuta inmediatamente.
- Recibirás el mismo mensaje que recibirías en el resumen diario.

Ejemplo:

```text
⏳ Oracle A1: todavía sin suerte hoy.

Último estado: FAILED
Intentos registrados: 42
Región: eu-madrid-1
Fecha: 01/09/2026 14:00

El bot sigue intentando automáticamente cada ~5 minutos.
Te avisaré en cuanto lo consiga.
```

---

# ⏱️ Frecuencia de reintentos

El workflow está configurado actualmente con:

```yaml
- cron: "2-57/5 * * * *"
```

Eso programa ejecuciones aproximadamente cada 5 minutos.

> [!NOTE]
> Los workflows programados de GitHub Actions no son un scheduler en tiempo real. Una ejecución puede comenzar algunos minutos más tarde si GitHub tiene carga.

Si quieres reducir la frecuencia, por ejemplo a cada 15 minutos:

```yaml
- cron: "2-57/15 * * * *"
```

---

# 🕑 Aviso diario

Este repositorio está preparado para enviar el resumen diario alrededor de las **14:00 hora de Madrid**.

Como los cron de GitHub utilizan UTC y Madrid cambia entre UTC+1 y UTC+2, hay dos ventanas:

```yaml
- cron: "0 12 * * *"
- cron: "0 13 * * *"
```

El propio workflow comprueba `Europe/Madrid` y **solo envía el mensaje cuando allí son realmente las 14:00**.

Para cambiar la hora, modifica los cron del bloque de `schedule` y la comprobación del job `daily-report`.

---

# 🎉 ¿Qué ocurre cuando lo consigue?

Cuando el último Job de Oracle devuelve:

```text
SUCCEEDED
```

Oracle A1 Hunter:

1. Detecta que el Stack terminó correctamente.
2. Te envía un mensaje por Telegram.
3. No crea otra instancia.
4. Desactiva el workflow automáticamente.

El mensaje será parecido a:

```text
🎉 ¡Oracle A1 conseguida!

El Stack de Oracle ha terminado correctamente.
Región: eu-madrid-1
Estado: SUCCEEDED

A1 Hunter se va a detener automáticamente para no realizar más intentos.
```

---

# 🛡️ Seguridad

Nunca guardes credenciales directamente en `.github/workflows/oracle-a1.yml`.

Usa siempre:

**Settings → Secrets and variables → Actions → Repository secrets**

No publiques:

- `OCI_PRIVATE_KEY`
- `TELEGRAM_BOT_TOKEN`
- claves privadas SSH
- archivos `.pem` privados

Los OCID y fingerprints no son equivalentes a una contraseña, pero tampoco hace falta publicarlos.

Si accidentalmente publicas tu clave privada OCI o el token del bot:

1. Revócalos inmediatamente.
2. Genera credenciales nuevas.
3. Actualiza los Secrets de GitHub.

---

# 🧠 Cómo decide si debe reintentar

El workflow **no relanza Terraform ante cualquier error**.

Si el último Job está en:

```text
ACCEPTED
IN_PROGRESS
CANCELING
```

no hace nada y espera al siguiente ciclo.

Si está en:

```text
FAILED
```

lee los logs de Terraform.

Solo permite un nuevo intento si encuentra:

```text
out of host capacity
```

o:

```text
out of capacity
```

Si detecta otro error, el job falla y envía una alerta por Telegram para que lo revises.

---

# 🧰 Troubleshooting

## `Out of host capacity`

```text
500-InternalError, Out of host capacity.
```

✅ Es el caso esperado. El workflow volverá a intentarlo.

---

## `The provided key is not a private key`

Revisa `OCI_PRIVATE_KEY`.

Debe contener el `.pem` privado completo:

```text
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

No uses la clave pública.

---

## `OCI authentication` falla

Comprueba:

- `OCI_USER_OCID`
- `OCI_TENANCY_OCID`
- `OCI_FINGERPRINT`
- `OCI_PRIVATE_KEY`
- `OCI_REGION`

Y verifica que la API Key pública asociada esté registrada en tu usuario de Oracle Cloud.

---

## `daily-report` aparece como `Skipped`

Es normal si ejecutaste manualmente el workflow **sin** marcar:

```text
test_daily_report
```

Para probar Telegram, ejecuta de nuevo el workflow con esa opción activada.

---

## El workflow ya no se ejecuta

Cuando consigue la VM, Oracle A1 Hunter se desactiva automáticamente.

También debes tener en cuenta que GitHub puede desactivar workflows programados de repositorios públicos después de largos periodos sin actividad. Si necesitas continuar, entra en **Actions** y vuelve a habilitar el workflow.

---

## El Stack falla por algo distinto a capacidad

El workflow no seguirá reintentando automáticamente ese error.

Revisa:

**Oracle Cloud → Resource Manager → Stack → Jobs → Logs**

Corrige el problema y vuelve a ejecutar manualmente Oracle A1 Hunter.

---

# ❓ FAQ

### ¿Tengo que dejar mi PC encendido?

No. Todo se ejecuta en los runners de GitHub Actions y en Oracle Resource Manager.

### ¿El repositorio crea automáticamente toda mi infraestructura?

No. Debes crear o guardar primero un **Stack válido en Oracle Resource Manager**. El repositorio automatiza los `Apply` sobre ese Stack.

### ¿Puede crear varias VMs por accidente?

El workflow comprueba el último Job antes de actuar. Cuando detecta `SUCCEEDED`, avisa y se desactiva.

### ¿Conseguiré una A1 seguro?

No. Depende completamente de la capacidad disponible en tu región y de que tu cuenta pueda crear esos recursos.

### ¿Puedo usar otra región?

Sí. Usa la región correspondiente en `OCI_REGION` y crea el Stack en esa región.

### ¿Puedo usar otra configuración de CPU y RAM?

Sí. La configuración pertenece al Stack de Oracle, no al workflow. Asegúrate de que entra dentro de los límites y precios que correspondan a tu cuenta.

### ¿GitHub Actions cuesta dinero?

Para repositorios públicos, GitHub ofrece runners estándar sin consumo de minutos facturables en las condiciones habituales de GitHub Actions. Las políticas pueden cambiar, así que revisa la documentación vigente de GitHub antes de dejar automatizaciones de larga duración.

---

# 📚 Documentación útil

- [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/)
- [Oracle Resource Manager](https://docs.oracle.com/en-us/iaas/Content/ResourceManager/home.htm)
- [OCI CLI - Resource Manager Jobs](https://docs.oracle.com/en-us/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/resource-manager/job.html)
- [GitHub Actions - Scheduled workflows](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows#schedule)
- [Telegram Bot API](https://core.telegram.org/bots/api)

---

## ⭐ ¿Te ha servido?

Si este proyecto te ha ayudado a automatizar la espera de capacidad de Oracle A1, puedes dejar una ⭐ al repositorio.

PRs, mejoras y sugerencias son bienvenidas.

---

> Este proyecto es independiente y no está afiliado, patrocinado ni respaldado por Oracle, GitHub o Telegram.
