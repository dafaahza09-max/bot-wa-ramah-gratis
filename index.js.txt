require('dotenv').config()
const { default: makeWASocket, useMultiFileAuthState, DisconnectReason } = require('@whiskeysockets/baileys')
const OpenAI = require('openai')
const qrcode = require('qrcode-terminal')

// PAKAI GROQ GRATIS - Gak perlu bayar
const ai = new OpenAI({
  apiKey: process.env.GROQ_API_KEY,
  baseURL: "https://api.groq.com/openai/v1"
})

async function startBot() {
  const { state, saveCreds } = await useMultiFileAuthState('auth_gratis')
  const sock = makeWASocket({ 
    auth: state,
    browser: ["Bot Ramah Gratis", "Chrome", "1.0"]
  })

  sock.ev.on('creds.update', saveCreds)

  sock.ev.on('connection.update', (u) => {
    const { connection, lastDisconnect, qr } = u
    if(qr) {
      console.log('SCAN QR INI PAKAI WA KAMU:')
      qrcode.generate(qr, {small: true})
    }
    if(connection === 'open') console.log('✅ Bot WA Ramah GRATIS Terhubung! Siap balas chat.')
    if(connection === 'close') {
      const shouldReconnect = lastDisconnect?.error?.output?.statusCode !== DisconnectReason.loggedOut
      console.log('Koneksi putus, mencoba sambung lagi...')
      if(shouldReconnect) startBot()
    }
  })

  sock.ev.on('messages.upsert', async ({ messages }) => {
    const m = messages[0]
    if(!m.message || m.key.fromMe) return
    
    const text = m.message.conversation || m.message.extendedTextMessage?.text || m.message.imageMessage?.caption || ""
    const from = m.key.remoteJid
    if(!text) return
    if(from.endsWith('@g.us')) return // jangan balas grup biar gak spam

    console.log(`📩 ${from}: ${text}`)
    await sock.sendPresenceUpdate('composing', from)

    try {
      const res = await ai.chat.completions.create({
        model: "llama-3.3-70b-versatile", // Model gratis terbaik di Groq
        messages: [
          {
            role: "system",
            content: `Kamu adalah Bot Ramah, asisten WA pribadi milik kak Abdul Mukhti.
Aturan:
1. Super ramah, hangat, santai, pakai bahasa Indonesia sehari-hari
2. Selalu panggil user dengan "kak"
3. Pakai 1-2 emoji saja biar natural, contoh: 👋 😊 🙏 ✨
4. Jawaban SINGKAT maksimal 3 kalimat, jangan bertele-tele
5. Kalau ditanya siapa kamu, jawab: "Aku Bot Ramah asistennya kak Mukhti 😊"
6. Jangan pernah mengaku sebagai ChatGPT atau Meta AI`
          },
          { role: "user", content: text }
        ],
        temperature: 0.7,
        max_tokens: 150
      })
      
      const reply = res.choices[0].message.content
      await sock.sendMessage(from, { text: reply })
      console.log(`✅ Balas: ${reply}`)

    } catch(e) {
      console.error("Error AI:", e.message)
      await sock.sendMessage(from, { text: "Aduh kak lagi agak lemot nih 😅 coba chat lagi ya" })
    }
  })
}

startBot()
