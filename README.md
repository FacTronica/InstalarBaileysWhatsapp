# 📲 Instalación de Baileys (WhatsApp) en Debian 12

Guía paso a paso para instalar y dejar funcionando **Baileys** (WhatsApp Web API no oficial) en **Debian 12**.

---

## 1️⃣ Actualizar el sistema

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 2️⃣ Instalar dependencias básicas

```bash
sudo apt install -y curl git build-essential ca-certificates
```

---

## 3️⃣ Crear usuario dedicado (recomendado)

```bash
sudo adduser whatsapp
sudo su - whatsapp
```

---

## 4️⃣ Instalar NVM (Node Version Manager)

```bash
curl -o install.sh https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh
bash install.sh
rm install.sh
```

Cargar NVM:

```bash
source ~/.bashrc
```

Verificar:

```bash
nvm --version
```

---

## 5️⃣ Instalar Node.js (LTS recomendado)

```bash
nvm install 20
nvm use 20
nvm alias default 20
```

Verificar:

```bash
node -v
npm -v
```

---

## 6️⃣ Crear proyecto Baileys

```bash
mkdir baileys-whatsapp
cd baileys-whatsapp
npm init -y
```

---

## 7️⃣ Instalar Baileys

```bash
npm install @whiskeysockets/baileys
```

Dependencias recomendadas:

```bash
npm install pino
npm install qrcode-terminal
npm install express
```


## 8️⃣ Editar archivo `package.json`
Se debe agregar   "type": "module",

```js
{
  "name": "whapi",
  "version": "1.0.0",
  "type": "module",
  "main": "whapi.js",
  "scripts": {
    "start": "node whapi.js"
  }
}
```



---

## 8️⃣ Crear archivo principal `whapi.js`

```bash
nano whapi.js
```

Contenido básico:

```js
import makeWASocket, {
  useMultiFileAuthState,
  DisconnectReason,
} from "@whiskeysockets/baileys";

import P from "pino";
import qrcode from "qrcode-terminal";
import express from "express";

/* ================= CONFIG ================= */
const PORT = 9000;
const API_KEY = "a8hf4bc7j7fs3g89";
const AUTH_DIR = "auth";
/* ========================================= */

let sock = null;
let whatsappReady = false;
let reconnecting = false;
let keepAliveInterval = null;

/* ================= WHATSAPP ================= */
async function startWhatsApp() {
  if (sock) {
    console.log("ℹ️ WhatsApp ya inicializado, no se recrea socket");
    return;
  }

  console.log("🚀 Iniciando WhatsApp...");

  const { state, saveCreds } = await useMultiFileAuthState(AUTH_DIR);

  sock = makeWASocket({
    auth: state,
    logger: P({ level: "silent" }),
    markOnlineOnConnect: false,
  });

  sock.ev.on("creds.update", saveCreds);

  sock.ev.on("connection.update", (update) => {
    const { connection, lastDisconnect, qr } = update;

    if (qr) {
      console.log("📱 Escanea QR (solo si es primera vez):");
      qrcode.generate(qr, { small: true });
    }

    if (connection === "open") {
      console.log("✅ WhatsApp conectado correctamente");
      whatsappReady = true;
      reconnecting = false;
      iniciarKeepAlive();
    }

    if (connection === "close") {
      whatsappReady = false;
      detenerKeepAlive();

      const reason = lastDisconnect?.error?.output?.statusCode;
      console.log("❌ Conexión cerrada. Código:", reason);

      if (reason === DisconnectReason.loggedOut) {
        console.log("🚫 Sesión cerrada definitivamente (logout)");
        sock = null;
        return;
      }

      if (!reconnecting) {
        reconnecting = true;
        console.log("🔄 Reconectando WhatsApp en 10 segundos...");

        setTimeout(() => {
          sock = null;
          startWhatsApp();
        }, 10000);
      }
    }
  });
}

/* ================= KEEP ALIVE ================= */
function iniciarKeepAlive() {
  if (keepAliveInterval) return;

  keepAliveInterval = setInterval(() => {
    if (sock?.user) {
      sock.sendPresenceUpdate("available").catch(() => {});
    }
  }, 60000);
}

function detenerKeepAlive() {
  if (keepAliveInterval) {
    clearInterval(keepAliveInterval);
    keepAliveInterval = null;
  }
}

/* ================= API HTTP ================= */
const app = express();
app.use(express.json());

app.post("/send", async (req, res) => {
  const { apikey, numero, grupo, mensaje, urladjunto } = req.body;

  /* ===== SEGURIDAD ===== */
  if (apikey !== API_KEY) {
    return res.status(401).json({ ok: false, error: "API KEY inválida" });
  }

  if (!mensaje) {
    return res.status(400).json({ ok: false, error: "mensaje obligatorio" });
  }

  if (!numero && !grupo) {
    return res.status(400).json({
      ok: false,
      error: "Debe indicar numero o grupo",
    });
  }

  if (numero && grupo) {
    return res.status(400).json({
      ok: false,
      error: "Indique solo numero o grupo",
    });
  }

  /* ===== ESTADO REAL ===== */
  if (!whatsappReady || !sock?.user) {
    return res.status(503).json({
      ok: false,
      error: "WhatsApp no conectado",
    });
  }

  try {
    let jid;
    let destino;

    /* ===== CONTACTO ===== */
    if (numero) {
      jid = numero.includes("@s.whatsapp.net")
        ? numero
        : `${numero.replace(/\D/g, "")}@s.whatsapp.net`;
      destino = "contacto";
    }

    /* ===== GRUPO ===== */
    if (grupo) {
      destino = "grupo";

      if (grupo.includes("@g.us")) {
        jid = grupo;
      } else {
        const groups = await sock.groupFetchAllParticipating();
        for (const [gid, info] of Object.entries(groups)) {
          if (info.subject === grupo) {
            jid = gid;
            break;
          }
        }
        if (!jid) {
          return res
            .status(404)
            .json({ ok: false, error: "Grupo no encontrado" });
        }
      }
    }

    /* ===== ENVÍO ===== */
    if (urladjunto) {
      await sock.sendMessage(jid, {
        document: { url: urladjunto },
        mimetype: "application/pdf",
        fileName: "archivo.pdf",
        caption: mensaje,
      });
    } else {
      await sock.sendMessage(jid, { text: mensaje });
    }

    console.log(`✅ Mensaje enviado a ${destino}: ${jid}`);
    res.json({ ok: true, destino, jid });
  } catch (err) {
    console.error("❌ Error enviando mensaje:", err);
    res.status(500).json({ ok: false, error: "Error enviando mensaje" });
  }
});

/* ================= STATUS ================= */
app.get("/status", (req, res) => {
  res.json({
    conectado: whatsappReady,
    numero: sock?.user?.id || null,
    reconectando: reconnecting,
  });
});

/* ================= START ================= */
app.listen(PORT, () => {
  console.log(`🚀 API escuchando en http://localhost:${PORT}`);
});

startWhatsApp();
```

Guardar y salir.

---

## 9️⃣ Ejecutar Baileys

```bash
node whapi.js
```

📸 Escanea el **QR** desde WhatsApp:
- WhatsApp → Dispositivos vinculados → Vincular dispositivo

---

## 🔁 10️⃣ Mantener activo con PM2 (opcional)

Instalar PM2:

```bash
npm install -g pm2
```

Ejecutar:

```bash
pm2 start whapi.js --name whapi
pm2 save
pm2 startup
```

---

## 🔥 11️⃣ Puertos necesarios (AWS / VPS)

Baileys **NO requiere puertos adicionales**.
Solo necesita **salida a internet (HTTPS 443)**.

---

## 📁 Estructura final

```
baileys-whatsapp/
├── app.js
├── auth/
├── node_modules/
└── package.json
```

## Ejemplo Php para enviar Whatsapp

```php
<?php
ini_set('display_errors', 1);
error_reporting(E_WARNING | E_ERROR);
date_default_timezone_set("Chile/Continental");

$urlapi = "http://su_ip:9000/send";

$token = "sutoken";

# a quien va el mensaje
$fono = "56912345678";
$grupo = "";
$mensaje = "Hola!, soy www.Whapi.cl";

$urladjunto = "";
$tipo_adjunto = ""; //"PDF";

$fono_cc_1 = "";
$fono_cc_2 = "";
$fono_cc_3 = "";

function enviarWhatsApp($urlapi, $token, $numero, $grupo, $mensaje, $urladjunto)
{


    $data = array(
        "apikey"  => $token,
        "numero"  => $numero,
        "mensaje" => $mensaje,
        "grupo" => $grupo,
        "urladjunto" => $urladjunto
    );

    $payload = json_encode($data);

    $ch = curl_init($urlapi);

    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, array(
        "Content-Type: application/json",
        "Content-Length: " . strlen($payload)
    ));
    curl_setopt($ch, CURLOPT_POSTFIELDS, $payload);
    curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, 10);
    curl_setopt($ch, CURLOPT_TIMEOUT, 30);

    $response = curl_exec($ch);

    if ($response === false) {
        $error = curl_error($ch);
        curl_close($ch);
        return array(
            "ok" => false,
            "error" => $error
        );
    }

    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    return array(
        "ok" => ($httpCode === 200),
        "http_code" => $httpCode,
        "response" => json_decode($response, true)
    );
}


enviarWhatsApp($urlapi, $token, $fono,  $grupo, $mensaje, $urladjunto);
?>
```


---

## ⚠️ Recomendaciones importantes

- ❌ No abuses de envíos masivos
- ⏱️ Agrega delays entre mensajes
- 🔐 Respalda la carpeta `auth/`
- 📵 Un número = una sesión estable

---

## ✅ Estado final

✔ Debian 12  
✔ Node.js 20  
✔ Baileys funcionando  
✔ WhatsApp vinculado  

---

💡 **Tip pro**: usa números nuevos o poco usados para evitar bloqueos.

---

Hecho con ❤️ para Debian 12
