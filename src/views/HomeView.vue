<template>
  <div class="page">
    <div class="hero">
      <img src="/images/medo.png" class="hero-bear" alt="Plushie bear" />
      <h2>Skenirajte QR kod</h2>
      <p>Prinesite kameru QR kodu na vašem mekanom ljubimcu.</p>
    </div>

    <!-- QR Scanner -->
    <div v-if="!scanning && !newPlushie" class="card center-card">
      <button class="btn btn-primary big-btn" @click="startScan">
        📷 Skeniraj QR Kod
      </button>
      <p class="hint">ili unesite ID ručno:</p>
      <div class="manual-row">
        <input v-model="manualId" placeholder="ID ljubimca..." class="input-field" @keyup.enter="loadManual" />
        <button class="btn btn-secondary" @click="loadManual">Učitaj</button>
      </div>
    </div>

    <!-- Scanner active -->
    <div v-if="scanning" class="scanner-wrap">
      <video ref="videoEl" class="scanner-video" autoplay playsinline></video>
      <div class="scanner-overlay">
        <div class="scanner-box"></div>
      </div>
      <button class="btn btn-danger mt" @click="stopScan">Zaustavi</button>
      <p v-if="scanMsg" class="scan-msg">{{ scanMsg }}</p>
    </div>

    <!-- New plushie setup -->
    <div v-if="newPlushie" class="card">
      <div class="new-plushie-icon">🧸</div>
      <h3>Novi ljubimac!</h3>
      <p>ID: <strong>{{ newPlushieId }}</strong></p>
      <p class="sub">Dajte mu ime:</p>
      <input v-model="newName" class="input-field big-input" placeholder="Unesite ime..." maxlength="32" />
      <button class="btn btn-primary mt" :disabled="!newName.trim() || saving" @click="saveNewPlushie">
        {{ saving ? 'Stvaram...' : '✨ Stvori ljubimca' }}
      </button>
      <p v-if="error" class="error-msg">{{ error }}</p>
    </div>
  </div>
</template>

<script>
import { createPlushie, getPlushieName } from '../homeAssistant.js'
import { useRouter } from 'vue-router'

export default {
  setup() {
    const router = useRouter()
    return { router }
  },
  data() {
    return {
      scanning: false,
      manualId: '',
      newPlushie: false,
      newPlushieId: '',
      newName: '',
      saving: false,
      error: '',
      scanMsg: '',
      stream: null,
      scanInterval: null,
    }
  },
  methods: {
    async startScan() {
      this.scanning = true
      this.scanMsg = 'Tražim QR kod...'
      try {
        this.stream = await navigator.mediaDevices.getUserMedia({
          video: { facingMode: 'environment' }
        })
        await this.$nextTick()
        this.$refs.videoEl.srcObject = this.stream

        // Use BarcodeDetector API if available
        if ('BarcodeDetector' in window) {
          const detector = new BarcodeDetector({ formats: ['qr_code'] })
          this.scanInterval = setInterval(async () => {
            try {
              const codes = await detector.detect(this.$refs.videoEl)
              if (codes.length > 0) {
                this.handleScannedValue(codes[0].rawValue)
              }
            } catch {}
          }, 500)
        } else {
          this.scanMsg = 'BarcodeDetector nije podržan. Koristite ručni unos.'
        }
      } catch (e) {
        this.scanMsg = 'Kamera nije dostupna: ' + e.message
      }
    },
    stopScan() {
      clearInterval(this.scanInterval)
      if (this.stream) this.stream.getTracks().forEach(t => t.stop())
      this.scanning = false
      this.scanMsg = ''
    },
    async handleScannedValue(value) {
      this.stopScan()
      // Extract ID from QR — expected format: "plushie:<id>" or just an ID
      const id = value.startsWith('plushie:') ? value.replace('plushie:', '') : value
      await this.loadPlushie(id)
    },
    async loadManual() {
      if (!this.manualId.trim()) return
      await this.loadPlushie(this.manualId.trim())
    },
    async loadPlushie(id) {
      const safeId = id.toLowerCase().replace(/[^a-z0-9_]/g, '_')
      const existing = await getPlushieName()
      if (existing) {
        // Plushie exists — go to its page
        this.router.push(`/ljubimac/${safeId}`)
      } else {
        // New plushie — prompt for name
        this.newPlushieId = safeId
        this.newPlushie = true
      }
    },
    async saveNewPlushie() {
      this.saving = true
      this.error = ''
      try {
        await createPlushie(this.newName.trim())
        this.router.push(`/ljubimac/${this.newPlushieId}`)
      } catch (e) {
        this.error = 'Greška: ' + e.message
      } finally {
        this.saving = false
      }
    },
  },
  beforeUnmount() {
    this.stopScan()
  }
}
</script>

<style scoped>
.hero {
  text-align: center;
  padding: 24px 0 16px;
}
.hero-bear {
  width: 100px;
  animation: float 3s ease-in-out infinite;
}
@keyframes float {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-10px); }
}
.hero h2 {
  font-family: 'Pacifico', cursive;
  font-size: 1.5rem;
  color: var(--pink);
  margin: 12px 0 6px;
}
.hero p { color: #7c3aed; font-size: 0.95rem; }

.card {
  background: white;
  border-radius: 24px;
  padding: 28px 24px;
  box-shadow: var(--shadow);
  margin: 16px 0;
  text-align: center;
}
.center-card { display: flex; flex-direction: column; align-items: center; gap: 12px; }

.big-btn { font-size: 1.1rem; padding: 16px 32px; }

.hint { font-size: 0.85rem; color: #9ca3af; }
.manual-row { display: flex; gap: 8px; width: 100%; }
.input-field {
  flex: 1;
  padding: 10px 16px;
  border: 2px solid var(--pink-light);
  border-radius: 50px;
  font-family: 'Nunito', sans-serif;
  font-size: 0.95rem;
  outline: none;
  transition: border-color 0.2s;
}
.input-field:focus { border-color: var(--purple); }
.big-input { font-size: 1.1rem; padding: 14px 20px; text-align: center; }

.scanner-wrap { position: relative; border-radius: 20px; overflow: hidden; margin: 16px 0; }
.scanner-video { width: 100%; border-radius: 20px; display: block; }
.scanner-overlay { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; }
.scanner-box {
  width: 200px; height: 200px;
  border: 3px solid var(--pink);
  border-radius: 12px;
  box-shadow: 0 0 0 9999px rgba(0,0,0,0.4);
  animation: pulse-border 2s ease-in-out infinite;
}
@keyframes pulse-border {
  0%, 100% { border-color: var(--pink); }
  50% { border-color: var(--purple); }
}
.scan-msg { text-align: center; color: var(--purple); font-weight: 700; font-size: 0.9rem; margin-top: 8px; }
.mt { margin-top: 12px; }
.sub { color: #9ca3af; font-size: 0.9rem; margin: 8px 0; }
.new-plushie-icon { font-size: 3rem; margin-bottom: 8px; }
h3 { font-size: 1.3rem; color: var(--purple-dark); }
.error-msg { color: var(--red); font-size: 0.9rem; margin-top: 8px; }
</style>