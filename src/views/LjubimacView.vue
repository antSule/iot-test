<template>
  <div class="page">
    <div v-if="loading" class="loading-wrap">
      <div class="spinner"></div>
      <p>Učitavam ljubimca...</p>
    </div>

    <div v-else-if="!found" class="card not-found">
      <div class="big-emoji">😢</div>
      <h3>Ljubimac nije pronađen</h3>
      <p>ID: <strong>{{ id }}</strong> ne postoji u Home Assistantu.</p>
      <RouterLink to="/" class="btn btn-primary mt">← Natrag</RouterLink>
    </div>

    <div v-else>
      <!-- Header card -->
      <div class="card plushie-card">
        <img src="/images/blushingmedo.png" class="plushie-img" alt="Plushie" />

        <!-- Name display/edit -->
        <div v-if="!editingName" class="name-row">
          <h2 class="plushie-name">{{ name }}</h2>
          <button class="icon-btn" @click="startEditName" title="Preimenuj">✏️</button>
        </div>
        <div v-else class="name-edit-row">
          <input v-model="tempName" class="input-field name-input" maxlength="32" @keyup.enter="saveName" @keyup.escape="editingName=false" />
          <button class="icon-btn green" @click="saveName" :disabled="!tempName.trim()">✅</button>
          <button class="icon-btn" @click="editingName=false">❌</button>
        </div>
        <p class="plushie-id">ID: {{ id }}</p>
        <p v-if="nameStatus" class="status-msg" :class="nameStatus.type">{{ nameStatus.msg }}</p>
      </div>

      <!-- Happiness card -->
      <div class="card happiness-card">
        <h3>Sreća</h3>
        <div class="happiness-bar-wrap">
          <img :src="`/images/healthBar${happiness}.png`" class="happiness-bar" :alt="`Sreća: ${happiness}/10`" />
        </div>
        <p class="happiness-label" :class="happinessClass">
          {{ happinessEmoji }} {{ happiness }}/10 — {{ happinessText }}
        </p>
        <div class="happiness-controls">
          <button class="btn btn-secondary ctrl-btn" @click="adjustHappiness(-1)" :disabled="happiness <= 0">– Manje</button>
          <button class="btn btn-primary ctrl-btn" @click="adjustHappiness(1)" :disabled="happiness >= 10">+ Više</button>
        </div>
      </div>

      <!-- Sound card -->
      <div class="card sound-card">
        <h3>Zvuk 🔊</h3>
        <p>Natjerajte ljubimca da govori putem vašeg kućnog zvučnika!</p>
        <div class="media-player-row">
          <label class="small-label">Media Player</label>
          <input v-model="mediaPlayer" class="input-field" placeholder="media_player.ha_speaker" />
        </div>
        <p v-if="soundStatus" class="status-msg" :class="soundStatus.type">{{ soundStatus.msg }}</p>
      </div>

      <!-- Refresh -->
      <div class="refresh-row">
        <button class="btn btn-secondary small-btn" @click="loadPlushie">🔄 Osvježi</button>
      </div>
    </div>
  </div>
</template>

<script>
import { RouterLink } from 'vue-router'
import {
  getPlushieName, getPlushieHappiness,
  setPlushieName, setPlushieHappiness,
} from '../homeAssistant.js'

export default {
  components: { RouterLink },
  props: ['id'],
  data() {
    return {
      loading: true,
      found: false,
      name: '',
      happiness: 10,
      editingName: false,
      tempName: '',
      nameStatus: null,
      soundStatus: null,
      playingSound: false,
      mediaPlayer: 'media_player.ha_speaker',
    }
  },
  computed: {
    happinessClass() {
      if (this.happiness >= 8) return 'happy'
      if (this.happiness >= 4) return 'ok'
      return 'sad'
    },
    happinessEmoji() {
      if (this.happiness >= 8) return '😄'
      if (this.happiness >= 5) return '🙂'
      if (this.happiness >= 3) return '😐'
      return '😢'
    },
    happinessText() {
      if (this.happiness === 10) return 'Presretan!'
      if (this.happiness >= 8) return 'Sretan'
      if (this.happiness >= 6) return 'Dobro'
      if (this.happiness >= 4) return 'Osrednje'
      if (this.happiness >= 2) return 'Tužan'
      return 'Jako tužan!'
    }
  },
  async mounted() {
    await this.loadPlushie()
  },
  methods: {
    async loadPlushie() {
      this.loading = true
      const [name, happiness] = await Promise.all([
        getPlushieName(),
        getPlushieHappiness(),
      ])
      if (name !== null) {
        this.found = true
        this.name = name
        this.happiness = happiness ?? 10
      } else {
        this.found = false
      }
      this.loading = false
    },
    startEditName() {
      this.tempName = this.name
      this.editingName = true
      this.nameStatus = null
    },
    async saveName() {
      if (!this.tempName.trim()) return
      const ok = await setPlushieName(this.tempName.trim())
      if (ok) {
        this.name = this.tempName.trim()
        this.nameStatus = { type: 'success', msg: '✅ Ime promijenjeno!' }
      } else {
        this.nameStatus = { type: 'error', msg: '❌ Greška pri promjeni imena.' }
      }
      this.editingName = false
      setTimeout(() => { this.nameStatus = null }, 3000)
    },
    async adjustHappiness(delta) {
      const newVal = Math.min(10, Math.max(0, this.happiness + delta))
      const ok = await setPlushieHappiness(newVal)
      if (ok) this.happiness = newVal
    },
  }
}
</script>

<style scoped>
.loading-wrap { text-align: center; padding: 60px 0; color: var(--purple); }
.spinner {
  width: 48px; height: 48px;
  border: 4px solid var(--pink-light);
  border-top-color: var(--pink);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
  margin: 0 auto 16px;
}
@keyframes spin { to { transform: rotate(360deg); } }

.card { background: white; border-radius: 24px; padding: 24px; box-shadow: var(--shadow); margin-bottom: 16px; }
.not-found { text-align: center; }
.big-emoji { font-size: 3rem; }
.mt { margin-top: 12px; }

.plushie-card { text-align: center; }
.plushie-img {
  width: 120px;
  animation: float 3s ease-in-out infinite;
}
@keyframes float {
  0%,100% { transform: translateY(0); }
  50% { transform: translateY(-8px); }
}
.name-row { display: flex; align-items: center; justify-content: center; gap: 10px; margin: 12px 0 4px; }
.plushie-name { font-family: 'Pacifico', cursive; font-size: 1.6rem; color: var(--pink); }
.icon-btn { background: none; border: none; font-size: 1.3rem; cursor: pointer; padding: 4px; transition: transform 0.1s; }
.icon-btn:hover { transform: scale(1.2); }
.name-edit-row { display: flex; align-items: center; gap: 6px; margin: 12px 0 4px; }
.name-input { flex: 1; font-size: 1.1rem; text-align: center; }
.plushie-id { font-size: 0.8rem; color: #d1d5db; }

.happiness-card h3, .sound-card h3 {
  font-size: 1.1rem; font-weight: 800; color: var(--purple-dark); margin-bottom: 12px;
}
.happiness-bar-wrap { display: flex; justify-content: center; margin: 8px 0; }
.happiness-bar { max-width: 100%; height: auto; }
.happiness-label { text-align: center; font-weight: 800; font-size: 1rem; margin: 8px 0; }
.happiness-label.happy { color: #16a34a; }
.happiness-label.ok { color: var(--yellow); }
.happiness-label.sad { color: var(--red); }
.happiness-controls { display: flex; gap: 12px; justify-content: center; margin-top: 12px; }
.ctrl-btn { min-width: 100px; }

.sound-card p { color: #6b7280; font-size: 0.9rem; margin-bottom: 12px; }
.media-player-row { display: flex; flex-direction: column; gap: 4px; }
.small-label { font-size: 0.8rem; font-weight: 700; color: #9ca3af; }
.input-field {
  padding: 10px 16px;
  border: 2px solid var(--pink-light);
  border-radius: 50px;
  font-family: 'Nunito', sans-serif;
  font-size: 0.9rem;
  outline: none;
  width: 100%;
  transition: border-color 0.2s;
}
.input-field:focus { border-color: var(--purple); }

.status-msg { font-size: 0.9rem; font-weight: 700; margin-top: 8px; text-align: center; }
.status-msg.success { color: #16a34a; }
.status-msg.error { color: var(--red); }

.refresh-row { display: flex; justify-content: center; margin-bottom: 24px; }
.small-btn { font-size: 0.85rem; padding: 8px 20px; }
</style>