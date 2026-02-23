<script setup lang="ts">
import { ref, watch, onMounted, computed } from "vue"
import { Bar } from 'vue-chartjs'
import { Chart as ChartJS, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale } from 'chart.js'
import axios from 'axios'

ChartJS.register(Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale)

/* ================= 1. STATE & CORE DATA ================= */
const activeTab = ref('dashboard') 
const isDark = ref(true)
const RAMADAN_START = new Date("2026-02-17")

// Stats & Main Data
const tilawah = ref(0); const dzikir = ref(0); const waterIntake = ref(0)
const reflection = ref(""); const sahurMenu = ref(""); const iftarMenu = ref("")
const targetKhatam = ref(30); const mood = ref('🤲')
const dailyPhoto = ref<string | null>(null)

// Prayer & Interface
const prayerTimes = ref<any>(null)
const countdownText = ref("Memuat...")
const showDuaModal = ref(false); const activeDuaTab = ref('Ramadhan')

const MOOD_ENGINE: any = {
  '😌': { label: 'Peaceful', ayat: '“Hanya dengan mengingat Allah hati menjadi tenteram.” (Ar-Ra’d: 28)' },
  '😔': { label: 'Sad', ayat: '“Janganlah kamu berduka cita, sesungguhnya Allah bersama kita.” (At-Taubah: 40)' },
  '🤲': { label: 'Grateful', ayat: '“Jika kamu bersyukur, niscaya Aku akan menambah (nikmat) kepadamu.” (Ibrahim: 7)' },
  '✨': { label: 'Inspired', ayat: '“Berlomba-lombalah kamu dalam kebaikan.” (Al-Baqarah: 148)' },
  '🌙': { label: 'Sleepy', ayat: 'Istirahatlah yang cukup, Afifah. Tubuhmu punya hak untuk dijaga.' }
}

const DUA_LIBRARY: any = {
  Ramadhan: [
    { title: "Niat Puasa", ar: "نَوَيْتُ صَوْمَ غَدٍ عَنْ أَدَاءِ فَرْضِ شَهْرِ رَمَضَانَ هَذِهِ السَّنَةِ لِلَّهِ تَعَالَى", tr: "Nawaitu shauma ghadin..." },
    { title: "Doa Buka Puasa", ar: "ذَهَبَ الظَّمَأُ وَابْتَلَّتِ الْعُرُوقُ وَثَبَتَ الأَجْرُ إِنْ شَاءَ اللهُ", tr: "Dzahabaz zhama'u..." }
  ],
  Harian: [
    { title: "Sapu Jagad", ar: "رَبَّنَا آتِنَا فِي الدُّنْيَا حَسَنَةً وَفِي الْآخِرَةِ حَسَنَةً وَقِنَا عَذَابَ النَّارِ", tr: "Ya Tuhan kami..." }
  ]
}

const dailyHabits = ref([
  { id: 1, text: "Sholat 5 Waktu", done: false, pts: 20 },
  { id: 2, text: "Puasa Hari Ini", done: true, pts: 50 },
  { id: 3, text: "Tarawih & Witir", done: false, pts: 15 },
  { id: 4, text: "Tahajud", done: false, pts: 20 },
  { id: 5, text: "Sholat Rawatib", done: false, pts: 10 },
  { id: 6, text: "Sedekah Subuh", done: false, pts: 15 },
  { id: 7, text: "Dzikir Pagi/Petang", done: false, pts: 10 },
  { id: 8, text: "Baca Tafsir 1 Ayat", done: false, pts: 10 },
  { id: 9, text: "Menjaga Lisan", done: false, pts: 5 }
])

/* ================= 2. LOGIC (PRAYER & XP) ================= */
const fetchPrayers = async () => {
  try {
    const res = await axios.get('https://api.aladhan.com/v1/timingsByCity?city=Jakarta&country=Indonesia&method=2')
    prayerTimes.value = res.data.data.timings
  } catch (err) { countdownText.value = "Offline" }
}

const startCountdown = () => {
  setInterval(() => {
    if (!prayerTimes.value) return
    const now = new Date()
    const [h, m] = prayerTimes.value.Maghrib.split(':')
    const maghrib = new Date(); maghrib.setHours(parseInt(h), parseInt(m), 0)
    const diff = maghrib.getTime() - now.getTime()
    if (diff > 0) {
      const hh = Math.floor(diff / 3600000); const mm = Math.floor((diff % 3600000) / 60000); const ss = Math.floor((diff % 60000) / 1000)
      countdownText.value = `${hh}j ${mm}m ${ss}s lagi buka!`
    } else { countdownText.value = "Waktunya Berbuka! 🌙" }
  }, 1000)
}

const dayNumber = computed(() => {
  const diff = Math.floor((new Date().getTime() - RAMADAN_START.getTime()) / (1000 * 60 * 60 * 24)) + 1
  return diff > 0 ? (diff > 30 ? 30 : diff) : 1
})

const totalXP = computed(() => {
  let pts = (tilawah.value * 50) + (Math.floor(dzikir.value / 33) * 5)
  dailyHabits.value.forEach(h => { if(h.done) pts += h.pts })
  return pts
})

const dailyProgress = computed(() => Math.round((dailyHabits.value.filter(h => h.done).length / dailyHabits.value.length) * 100))

const handlePhotoUpload = (e: any) => {
  const file = e.target.files[0]
  if (file) {
    const reader = new FileReader()
    reader.onload = (ev) => { dailyPhoto.value = ev.target?.result as string }
    reader.readAsDataURL(file)
  }
}

/* ================= 3. STORAGE ================= */
onMounted(() => {
  fetchPrayers(); startCountdown()
  const saved = localStorage.getItem("ramadhancore_v_ultimate")
  if (saved) {
    const d = JSON.parse(saved)
    tilawah.value = d.tilawah || 0; dzikir.value = d.dzikir || 0
    reflection.value = d.reflection || ""; waterIntake.value = d.waterIntake || 0
    sahurMenu.value = d.sahurMenu || ""; iftarMenu.value = d.iftarMenu || ""
    mood.value = d.mood || '🤲'; dailyPhoto.value = d.dailyPhoto || null
    targetKhatam.value = d.targetKhatam || 30
    if(d.dailyHabits) dailyHabits.value = d.dailyHabits
  }
})

watch([tilawah, dzikir, reflection, waterIntake, sahurMenu, iftarMenu, mood, dailyPhoto, targetKhatam, dailyHabits], () => {
  localStorage.setItem("ramadhancore_v_ultimate", JSON.stringify({
    tilawah: tilawah.value, dzikir: dzikir.value, reflection: reflection.value, waterIntake: waterIntake.value,
    sahurMenu: sahurMenu.value, iftarMenu: iftarMenu.value, mood: mood.value, dailyPhoto: dailyPhoto.value,
    targetKhatam: targetKhatam.value, dailyHabits: dailyHabits.value
  }))
}, { deep: true })
</script>

<template>
  <div :class="['min-h-screen pb-32 transition-all duration-500 text-left', isDark ? 'bg-slate-950 text-slate-100' : 'bg-stone-50 text-slate-900']">
    
    <div v-if="showDuaModal" class="fixed inset-0 z-[60] flex items-center justify-center p-4 bg-black/80 backdrop-blur-md">
      <div :class="['w-full max-w-2xl rounded-[2.5rem] p-8 max-h-[85vh] overflow-y-auto relative', isDark ? 'bg-slate-900' : 'bg-white text-slate-900']">
        <button @click="showDuaModal = false" class="absolute top-6 right-8 text-2xl">✕</button>
        <h2 class="text-2xl font-black italic text-emerald-500 mb-6 uppercase">Dua Library</h2>
        <div class="flex gap-4 mb-8">
          <button v-for="cat in ['Ramadhan', 'Harian']" :key="cat" @click="activeDuaTab = cat"
            :class="['px-6 py-2 rounded-full text-xs font-bold transition', activeDuaTab === cat ? 'bg-emerald-500 text-white' : 'bg-slate-800 text-slate-400']">{{ cat }}</button>
        </div>
        <div class="space-y-6">
          <div v-for="dua in DUA_LIBRARY[activeDuaTab]" :key="dua.title" class="p-6 rounded-[2rem] border border-emerald-500/10 bg-emerald-500/5 text-left">
            <h4 class="text-[10px] font-black uppercase text-emerald-500 mb-2">{{ dua.title }}</h4>
            <p class="text-2xl text-right font-serif leading-loose mb-2">{{ dua.ar }}</p>
            <p class="text-[10px] opacity-60 italic">{{ dua.tr }}</p>
          </div>
        </div>
      </div>
    </div>

    <header class="p-6 flex justify-between items-center max-w-4xl mx-auto">
      <div>
        <h2 class="text-2xl font-black italic text-emerald-500 tracking-tighter uppercase">RamadhanCore</h2>
        <span class="text-[9px] font-black uppercase tracking-[0.2em] opacity-50">Day {{ dayNumber }} • {{ totalXP }} XP</span>
      </div>
      <button @click="isDark = !isDark" class="w-12 h-12 rounded-2xl bg-emerald-500/10 flex items-center justify-center text-xl shadow-inner">
        {{ isDark ? '☀️' : '🌙' }}
      </button>
    </header>

    <main class="max-w-2xl mx-auto px-6 space-y-6">
      <div class="p-8 bg-gradient-to-br from-emerald-500 to-teal-700 rounded-[2.5rem] text-white shadow-xl relative overflow-hidden">
        <p class="text-[10px] font-black opacity-60 mb-1 uppercase tracking-widest">Waktu Maghrib Jakarta</p>
        <h3 class="text-3xl font-black tracking-tight uppercase">{{ countdownText }}</h3>
        <div class="absolute -right-4 -bottom-4 text-9xl opacity-10 rotate-12">🕌</div>
      </div>

      <section :class="['p-6 rounded-[2.5rem] border text-center', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
        <div class="flex justify-around mb-6">
          <button v-for="(v, k) in MOOD_ENGINE" :key="k" @click="mood = k"
            :class="['text-3xl transition-all', mood === k ? 'scale-125' : 'grayscale opacity-20']">{{ k }}</button>
        </div>
        <div class="p-4 rounded-2xl bg-emerald-500/5 border border-emerald-500/10">
          <p class="text-[10px] font-black text-emerald-500 uppercase mb-1">AI Reflection</p>
          <p class="text-xs italic opacity-80 leading-relaxed">{{ MOOD_ENGINE[mood].ayat }}</p>
        </div>
      </section>

      <div class="flex bg-slate-900/50 p-1.5 rounded-2xl gap-2 backdrop-blur-md">
        <button v-for="t in ['dashboard', 'ibadah', 'spirituil']" :key="t" @click="activeTab = t"
          :class="['flex-1 py-3 rounded-xl text-[9px] font-black uppercase tracking-widest transition-all', activeTab === t ? 'bg-emerald-500 text-white shadow-lg' : 'opacity-30']">{{ t }}</button>
      </div>

      <div v-if="activeTab === 'dashboard'" class="space-y-6 animate-in">
        <section :class="['p-4 rounded-[2rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
          <div class="grid grid-cols-10 gap-1.5">
            <div v-for="d in 30" :key="d" :class="['aspect-square rounded-lg flex items-center justify-center text-[8px] font-bold border', 
              d < dayNumber ? 'bg-emerald-500 border-emerald-500 text-white' : d === dayNumber ? 'border-emerald-500 text-emerald-500 animate-pulse' : 'border-slate-800 opacity-20']">{{ d }}</div>
          </div>
        </section>

        <div :class="['p-6 rounded-[2.5rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
          <h4 class="text-[10px] font-black uppercase text-emerald-500 mb-6 tracking-widest">Misi Harian ({{ dailyProgress }}%)</h4>
          <div class="space-y-3">
            <div v-for="habit in dailyHabits" :key="habit.id" @click="habit.done = !habit.done"
              :class="['p-4 rounded-2xl border transition-all flex justify-between items-center cursor-pointer', habit.done ? 'bg-emerald-500/10 border-emerald-500' : 'bg-slate-800/20 border-transparent opacity-40']">
              <span class="text-sm font-bold">{{ habit.text }}</span>
              <div :class="['w-5 h-5 rounded-full border-2 flex items-center justify-center', habit.done ? 'bg-emerald-500 border-emerald-500' : 'border-slate-700']">
                <span v-if="habit.done" class="text-[8px] text-white">✓</span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div v-if="activeTab === 'ibadah'" class="space-y-6 animate-in">
        <div v-if="prayerTimes" class="grid grid-cols-2 gap-3">
          <div v-for="n in ['Imsak','Fajr','Dhuhr','Asr','Maghrib','Isha']" :key="n" 
            :class="['p-5 rounded-[2rem] border text-center', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white border-slate-100']">
            <p class="text-[10px] font-bold opacity-40 uppercase mb-1">{{ n === 'Fajr' ? 'Subuh' : n }}</p>
            <p class="text-xl font-black text-emerald-500">{{ prayerTimes[n] }}</p>
          </div>
        </div>

        <div class="p-8 rounded-[3rem] border border-emerald-500/20 bg-emerald-500/5 text-center">
          <p class="text-[10px] font-black text-emerald-500 uppercase mb-2 tracking-widest italic">Tasbih Digital</p>
          <div class="text-7xl font-black mb-6 font-mono tracking-tighter text-emerald-500">{{ dzikir }}</div>
          <button @click="dzikir++" class="w-full py-10 bg-emerald-500 rounded-[2.5rem] text-3xl font-black shadow-xl active:scale-95 transition-all">TAP</button>
          <button @click="dzikir = 0" class="mt-4 text-[10px] opacity-30 font-bold uppercase">Reset</button>
        </div>

        <div :class="['p-8 rounded-[2.5rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
          <p class="text-[10px] font-black text-blue-500 uppercase mb-4 tracking-widest">Hydration (8 Gelas)</p>
          <div class="grid grid-cols-8 gap-2">
            <button v-for="i in 8" :key="i" @click="waterIntake = i" 
              :class="['h-10 rounded-xl transition-all', waterIntake >= i ? 'bg-blue-500 shadow-lg' : 'bg-slate-800 opacity-20']"></button>
          </div>
        </div>
      </div>

      <div v-if="activeTab === 'spirituil'" class="space-y-6 animate-in">
        <div :class="['p-6 rounded-[2.5rem] border bg-slate-900 border-slate-800 text-white overflow-hidden relative']">
          <p class="text-[10px] font-black uppercase text-emerald-500 mb-2">Progres Khatam</p>
          <div class="flex justify-between items-center mb-4 relative z-10">
            <h3 class="text-4xl font-black italic">{{ tilawah }} <span class="text-xs not-italic opacity-30">/ {{ targetKhatam }} JUZ</span></h3>
            <button @click="tilawah++" class="w-12 h-12 bg-emerald-500 rounded-2xl text-black font-black">+</button>
          </div>
          <input type="range" v-model="targetKhatam" min="1" max="60" class="w-full h-1.5 bg-slate-800 accent-emerald-500 rounded-full appearance-none">
        </div>

        <div :class="['p-6 rounded-[2.5rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
          <p class="text-[10px] font-black text-emerald-500 uppercase mb-4 tracking-widest">Momen & Refleksi</p>
          <div v-if="!dailyPhoto" class="relative border-2 border-dashed border-slate-700 rounded-3xl h-32 flex flex-col items-center justify-center cursor-pointer mb-4">
            <input type="file" @change="handlePhotoUpload" class="absolute inset-0 opacity-0 cursor-pointer" accept="image/*">
            <span class="text-2xl mb-1">📸</span>
            <span class="text-[9px] opacity-40 font-bold uppercase">Upload Moment</span>
          </div>
          <div v-else class="relative overflow-hidden rounded-3xl h-48 mb-4">
            <img :src="dailyPhoto" class="w-full h-full object-cover">
            <button @click="dailyPhoto = null" class="absolute top-2 right-2 bg-black/50 w-8 h-8 rounded-full text-xs">✕</button>
          </div>
          <textarea v-model="reflection" class="bg-transparent w-full h-24 text-sm outline-none resize-none border-t border-slate-800 pt-4" placeholder="Tulis catatan kecil untuk Allah..."></textarea>
        </div>

        <div class="grid grid-cols-2 gap-4 text-left">
          <div :class="['p-6 rounded-[2rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
            <p class="text-[10px] font-black text-orange-400 mb-2 uppercase tracking-widest">Menu Sahur</p>
            <input v-model="sahurMenu" class="bg-transparent w-full text-xs outline-none" placeholder="...">
          </div>
          <div :class="['p-6 rounded-[2rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
            <p class="text-[10px] font-black text-emerald-500 mb-2 uppercase tracking-widest">Menu Buka</p>
            <input v-model="iftarMenu" class="bg-transparent w-full text-xs outline-none" placeholder="...">
          </div>
        </div>

        <button @click="showDuaModal = true" class="w-full p-8 rounded-[3rem] bg-gradient-to-br from-emerald-500 to-teal-600 text-white font-black italic shadow-xl active:scale-95 transition-all">📖 LIBRARY DOA</button>
      </div>
    </main>

    <nav :class="['fixed bottom-8 left-1/2 -translate-x-1/2 w-[90%] max-w-md p-2 rounded-[2rem] border flex justify-between z-50 backdrop-blur-xl transition-all', isDark ? 'bg-slate-900/90 border-slate-800' : 'bg-white/90 border-slate-200 shadow-2xl']">
      <button v-for="tab in ['dashboard', 'ibadah', 'spirituil']" :key="tab" @click="activeTab = tab"
        :class="['flex-1 py-4 rounded-[1.5rem] text-[9px] font-black uppercase tracking-tighter transition-all', activeTab === tab ? 'bg-emerald-500 text-white' : 'opacity-20']">
        {{ tab }}
      </button>
    </nav>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&family=Amiri&display=swap');
* { font-family: 'Plus Jakarta Sans', sans-serif; -webkit-tap-highlight-color: transparent; }
.animate-in { animation: fadeIn 0.4s ease-out; }
@keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
.font-serif { font-family: 'Amiri', serif; }
</style>