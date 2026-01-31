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
npm install pino qrcode-terminal
```

---

## 8️⃣ Crear archivo principal `app.js`

```bash
nano app.js
```

Contenido básico:

```js
const { default: makeWASocket, useMultiFileAuthState, DisconnectReason } = require('@whiskeysockets/baileys')
const Pino = require('pino')

async function start() {
    const { state, saveCreds } = await useMultiFileAuthState('auth')

    const sock = makeWASocket({
        auth: state,
        logger: Pino({ level: 'silent' })
    })

    sock.ev.on('creds.update', saveCreds)

    sock.ev.on('connection.update', (update) => {
        const { connection, lastDisconnect } = update
        if (connection === 'close') {
            const reason = lastDisconnect?.error?.output?.statusCode
            console.log('Conexión cerrada. Código:', reason)
            start()
        }
        if (connection === 'open') {
            console.log('✅ WhatsApp conectado correctamente')
        }
    })
}

start()
```

Guardar y salir.

---

## 9️⃣ Ejecutar Baileys

```bash
node app.js
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
pm2 start app.js --name whatsapp-baileys
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
