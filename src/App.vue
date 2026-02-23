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

// --- STATE UTAMA (TIDAK ADA YANG DIHAPUS) ---
const tilawah = ref(0); const dzikir = ref(0); const waterIntake = ref(0)
const reflection = ref(""); const sahurMenu = ref(""); const iftarMenu = ref("")
const targetKhatam = ref(30); const mood = ref('🤲')
const dailyPhoto = ref<string | null>(null)

// --- FITUR BARU ---
const userCity = ref("Jakarta") 
const favorites = ref<number[]>([]) 
const prayerTimes = ref<any>(null)
const countdownText = ref("Memuat...")
const showDuaModal = ref(false); const activeDuaTab = ref('Ramadhan')

const MOOD_ENGINE: Record<string, { label: string, ayat: string }> = {
  '😌': { label: 'Peaceful', ayat: '“Hanya dengan mengingat Allah hati menjadi tenteram.” (Ar-Ra’d: 28)' },
  '😔': { label: 'Sad', ayat: '“Janganlah kamu berduka cita, sesungguhnya Allah bersama kita.” (At-Taubah: 40)' },
  '🤲': { label: 'Grateful', ayat: '“Jika kamu bersyukur, niscaya Aku akan menambah (nikmat) kepadamu.” (Ibrahim: 7)' },
  '✨': { label: 'Inspired', ayat: '“Berlomba-lombalah kamu dalam kebaikan.” (Al-Baqarah: 148)' },
  '🌙': { label: 'Sleepy', ayat: 'Istirahatlah yang cukup, Afifah. Tubuhmu punya hak untuk dijaga.' }
}

const DUA_LIBRARY: any = {
  Ramadhan: [
    { title: "Niat Puasa", ar: "نَوَيْتُ صَوْمَ غَدٍ عَنْ أَدَاءِ فَرْضِ شَهْرِ رَمَضَانَ هَذِهِ السَّنَةِ لِلَّهِ تَعَالَى", tr: "Nawaitu shauma ghadin..." },
    { title: "Doa Buka Puasa", ar: "ذَهَبَ الظَّمَأُ وَابْتَلَّتِ الْعُرُوقُ وَثَبَتَ الأَجْرُ إِنْ شَاءَ اللهُ", tr: "Dzahabaz zhama'u..." },
    { title: "Doa Lailatul Qadar", ar: "اللَّهُمَّ إِنَّكَ عَفُوٌّ تُحِبُّ الْعَفْوَ فَاعْفُ عَنِّي", tr: "Allahumma innaka 'afuwwun..." },
    { title: "Doa Sahur", ar: "يَرْحَمُ اللهُ الْمُتَسَحِّرِيْنَ", tr: "Semoga Allah merahmati orang yang sahur." }
  ],
  Harian: [
    { title: "Sapu Jagad", ar: "رَبَّنَا آتِنَا فِي الدُّنْيَا حَسَنَةً وَفِي الْآخِرَةِ حَسَنَةً وَقِنَا عَذَابَ النَّارِ", tr: "Ya Tuhan kami..." }
  ]
}

const DAILY_QUOTES = [
  { id: 1, text: "Puasa adalah perisai dari api neraka.", author: "HR. Bukhari & Muslim" },
  { id: 2, text: "Bau mulut orang berpuasa lebih harum bagi Allah daripada misk.", author: "HR. Muslim" },
  { id: 3, text: "Sedekah paling utama adalah sedekah di bulan Ramadhan.", author: "HR. Tirmidzi" }
]

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

/* ================= 2. SMART LOGIC & CHARTS ================= */
const dayNumber = computed(() => {
  const diff = Math.floor((new Date().getTime() - RAMADAN_START.getTime()) / (1000 * 60 * 60 * 24)) + 1
  return diff > 0 ? (diff > 30 ? 30 : diff) : 1
})

const currentQuote = computed(() => DAILY_QUOTES[dayNumber.value % DAILY_QUOTES.length])
const isMalamGanjil = computed(() => [21, 23, 25, 27, 29].includes(dayNumber.value))
const dailyProgress = computed(() => Math.round((dailyHabits.value.filter(h => h.done).length / dailyHabits.value.length) * 100))

const estimasiPerHari = computed(() => {
  const sisaJuz = targetKhatam.value - tilawah.value
  const sisaHari = 30 - dayNumber.value + 1
  return sisaJuz <= 0 ? 0 : (sisaJuz / (sisaHari > 0 ? sisaHari : 1)).toFixed(1)
})

const statusKhatam = computed(() => {
  if (tilawah.value >= dayNumber.value) return "On Track! Mantap Afifah! ✨"
  return `Tertinggal ${dayNumber.value - tilawah.value} Juz. Kejar yuk! 🏃🏻‍♀️`
})

const chartData = computed(() => ({
  labels: ['S', 'S', 'R', 'K', 'J', 'S', 'M'],
  datasets: [{
    data: [65, 80, 45, 90, 70, 55, dailyProgress.value],
    backgroundColor: '#10b981',
    borderRadius: 8,
    barThickness: 12
  }]
}))

const chartOptions = computed(() => ({
  responsive: true,
  maintainAspectRatio: false,
  plugins: { legend: { display: false } },
  scales: {
    x: { grid: { display: false }, ticks: { color: isDark.value ? '#475569' : '#94a3b8', font: { size: 10, weight: '800' } } },
    y: { display: false, max: 100 }
  }
}))

/* ================= 3. UTILITIES ================= */
const fetchPrayers = async () => {
  try {
    const res = await axios.get(`https://api.aladhan.com/v1/timingsByCity?city=${userCity.value}&country=Indonesia&method=2`)
    prayerTimes.value = res.data.data.timings
  } catch (err) { countdownText.value = "Kota tidak valid" }
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
    } else { countdownText.value = "Selamat Berbuka! 🌙" }
  }, 1000)
}

const totalXP = computed(() => {
  let pts = (tilawah.value * 50) + (Math.floor(dzikir.value / 33) * 5)
  dailyHabits.value.forEach(h => { if(h.done) pts += h.pts })
  return pts
})

const handlePhotoUpload = (e: any) => {
  const file = e.target.files[0]
  if (file) {
    const reader = new FileReader()
    reader.onload = (ev) => { dailyPhoto.value = ev.target?.result as string }
    reader.readAsDataURL(file)
  }
}

/* ================= 4. STORAGE ================= */
onMounted(() => {
  fetchPrayers(); startCountdown()
  const saved = localStorage.getItem("ramadhancore_v_final")
  if (saved) {
    const d = JSON.parse(saved)
    tilawah.value = d.tilawah || 0; dzikir.value = d.dzikir || 0
    reflection.value = d.reflection || ""; waterIntake.value = d.waterIntake || 0
    sahurMenu.value = d.sahurMenu || ""; iftarMenu.value = d.iftarMenu || ""
    mood.value = d.mood || '🤲'; dailyPhoto.value = d.dailyPhoto || null
    targetKhatam.value = d.targetKhatam || 30
    userCity.value = d.userCity || "Jakarta"
    if(d.dailyHabits) dailyHabits.value = d.dailyHabits
  }
})

watch([tilawah, dzikir, reflection, waterIntake, sahurMenu, iftarMenu, mood, dailyPhoto, targetKhatam, dailyHabits, userCity], () => {
  localStorage.setItem("ramadhancore_v_final", JSON.stringify({
    tilawah: tilawah.value, dzikir: dzikir.value, reflection: reflection.value, waterIntake: waterIntake.value,
    sahurMenu: sahurMenu.value, iftarMenu: iftarMenu.value, mood: mood.value, dailyPhoto: dailyPhoto.value,
    targetKhatam: targetKhatam.value, dailyHabits: dailyHabits.value, userCity: userCity.value
  }))
}, { deep: true })
</script>

<template>
  <div :class="['min-h-screen pb-32 transition-all duration-500 text-left', isDark ? 'bg-slate-950 text-slate-100' : 'bg-stone-50 text-slate-900']">
    
    <div v-if="showDuaModal" class="fixed inset-0 z-[60] flex items-center justify-center p-4 bg-black/80 backdrop-blur-md">
      <div :class="['w-full max-w-2xl rounded-[2.5rem] p-8 max-h-[85vh] overflow-y-auto relative', isDark ? 'bg-slate-900' : 'bg-white text-slate-900']">
        <button @click="showDuaModal = false" class="absolute top-6 right-8 text-2xl">✕</button>
        <h2 class="text-2xl font-black italic text-emerald-500 mb-6 uppercase tracking-tighter">Dua Library</h2>
        <div class="flex gap-3 mb-8">
          <button v-for="cat in ['Ramadhan', 'Harian']" :key="cat" @click="activeDuaTab = cat"
            :class="['px-6 py-2 rounded-full text-[10px] font-black uppercase transition', activeDuaTab === cat ? 'bg-emerald-500 text-white' : 'bg-slate-800 text-slate-400']">{{ cat }}</button>
        </div>
        <div class="space-y-4">
          <div v-for="dua in DUA_LIBRARY[activeDuaTab]" :key="dua.title" class="p-6 rounded-[2rem] border border-emerald-500/10 bg-emerald-500/5">
            <h4 class="text-[9px] font-black uppercase text-emerald-500 mb-3 tracking-widest">{{ dua.title }}</h4>
            <p class="text-2xl text-right font-serif leading-loose mb-3">{{ dua.ar }}</p>
            <p class="text-[10px] opacity-50 italic">{{ dua.tr }}</p>
          </div>
        </div>
      </div>
    </div>

    <header class="p-6 flex justify-between items-center max-w-4xl mx-auto">
      <div>
        <h2 class="text-2xl font-black italic text-emerald-500 tracking-tighter uppercase">RamadhanCore</h2>
        <span class="text-[9px] font-black uppercase tracking-[0.2em] opacity-40">Day {{ dayNumber }} • {{ totalXP }} XP</span>
      </div>
      <button @click="isDark = !isDark" class="w-12 h-12 rounded-2xl bg-emerald-500/10 flex items-center justify-center text-xl shadow-inner">
        {{ isDark ? '☀️' : '🌙' }}
      </button>
    </header>

    <main class="max-w-2xl mx-auto px-6 space-y-6">
      <div class="space-y-3">
        <div class="p-8 bg-gradient-to-br from-emerald-500 to-teal-700 rounded-[2.5rem] text-white shadow-xl relative overflow-hidden">
          <p class="text-[10px] font-black opacity-60 mb-1 uppercase tracking-widest">Countdown Berbuka</p>
          <h3 class="text-3xl font-black tracking-tight uppercase">{{ countdownText }}</h3>
          <div class="absolute -right-4 -bottom-4 text-9xl opacity-10 rotate-12">🕌</div>
        </div>

        <div :class="['p-6 rounded-[2rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
          <p class="text-[8px] font-black text-emerald-500 uppercase tracking-[0.2em] mb-2">Quote of the Day</p>
          <p class="text-sm font-bold italic">"{{ currentQuote.text }}"</p>
          <p class="text-[10px] opacity-40 mt-1">— {{ currentQuote.author }}</p>
        </div>

        <div v-if="isMalamGanjil" class="p-4 bg-amber-500/10 border border-amber-500/30 rounded-2xl flex items-center gap-3 animate-pulse">
          <span class="text-xl">✨</span>
          <p class="text-[9px] font-black uppercase text-amber-500 tracking-widest">Potential Lailatul Qadar (Day {{ dayNumber }}) - Perbanyak Doa!</p>
        </div>
      </div>

      <section :class="['p-6 rounded-[2.5rem] border text-center', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
        <div class="flex justify-around mb-6">
          <button v-for="(_, emoji) in MOOD_ENGINE" :key="emoji" @click="mood = String(emoji)"
            :class="['text-3xl transition-all', mood === emoji ? 'scale-125' : 'grayscale opacity-20']">{{ emoji }}</button>
        </div>
        <div class="p-4 rounded-2xl bg-emerald-500/5 border border-emerald-500/10">
          <p class="text-[10px] font-black text-emerald-500 uppercase mb-1">AI Reflection</p>
          <p class="text-xs italic opacity-80 leading-relaxed tracking-tight">{{ MOOD_ENGINE[mood]?.ayat }}</p>
        </div>
      </section>

      <div class="flex bg-slate-900/50 p-1.5 rounded-2xl gap-2 backdrop-blur-md sticky top-4 z-40">
        <button v-for="t in ['dashboard', 'ibadah', 'spirituil']" :key="t" @click="activeTab = t"
          :class="['flex-1 py-3 rounded-xl text-[9px] font-black uppercase tracking-widest transition-all', activeTab === t ? 'bg-emerald-500 text-white shadow-lg' : 'opacity-30']">{{ t }}</button>
      </div>

      <div v-if="activeTab === 'dashboard'" class="space-y-6 animate-in">
        <div class="grid grid-cols-2 gap-4">
          <div :class="['p-6 rounded-[2rem] border flex flex-col items-center', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
            <span class="text-3xl font-black text-emerald-500">{{ dailyProgress }}%</span>
            <span class="text-[8px] font-black uppercase opacity-40 mt-1">Daily Progress</span>
          </div>
          <div :class="['p-6 rounded-[2rem] border flex flex-col items-center', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
            <span class="text-3xl font-black text-emerald-500">{{ tilawah }}</span>
            <span class="text-[8px] font-black uppercase opacity-40 mt-1">Total Juz</span>
          </div>
        </div>

        <div :class="['p-6 rounded-[2.5rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
          <div class="flex justify-between items-center mb-6">
            <h4 class="text-[10px] font-black uppercase text-emerald-500 tracking-widest">Konsistensi Mingguan</h4>
            <span class="text-[8px] opacity-40 font-black italic">AKTIF</span>
          </div>
          <div class="h-32">
            <Bar :data="chartData" :options="chartOptions" />
          </div>
        </div>

        <div :class="['p-6 rounded-[2.5rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
          <h4 class="text-[10px] font-black uppercase text-emerald-500 mb-6 tracking-widest">Target Harian</h4>
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
        <div :class="['p-4 rounded-3xl border flex items-center gap-3', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
          <span class="text-lg">📍</span>
          <input v-model="userCity" @change="fetchPrayers" placeholder="Masukkan Kota..." 
            class="bg-transparent outline-none text-sm font-bold w-full uppercase tracking-widest">
        </div>

        <div v-if="prayerTimes" class="grid grid-cols-2 gap-3">
          <div v-for="n in ['Imsak','Fajr','Dhuhr','Asr','Maghrib','Isha']" :key="n" 
            :class="['p-5 rounded-[2rem] border text-center', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
            <p class="text-[10px] font-bold opacity-30 uppercase mb-1">{{ n === 'Fajr' ? 'Subuh' : n }}</p>
            <p class="text-xl font-black text-emerald-500">{{ prayerTimes[n] }}</p>
          </div>
        </div>

        <div class="p-10 rounded-[3rem] border border-emerald-500/20 bg-emerald-500/5 text-center relative overflow-hidden">
          <p class="text-[10px] font-black text-emerald-500 uppercase mb-2 tracking-widest italic">Digital Tasbih</p>
          <div class="text-8xl font-black mb-6 font-mono tracking-tighter text-emerald-500 drop-shadow-lg">{{ dzikir }}</div>
          <button @click="dzikir++" class="w-full py-12 bg-emerald-500 rounded-[2.5rem] text-3xl font-black shadow-xl active:scale-95 transition-all text-black uppercase">TAP</button>
          <button @click="dzikir = 0" class="mt-4 text-[10px] opacity-20 font-bold uppercase tracking-widest">Reset Counter</button>
        </div>

        <div :class="['p-8 rounded-[2.5rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
          <p class="text-[10px] font-black text-blue-500 uppercase mb-4 tracking-widest">Hydration Reminder (8 Gelas)</p>
          <div class="grid grid-cols-8 gap-2">
            <button v-for="i in 8" :key="i" @click="waterIntake = i" 
              :class="['h-10 rounded-xl transition-all', waterIntake >= i ? 'bg-blue-500 shadow-lg' : 'bg-slate-800/20 opacity-20']"></button>
          </div>
        </div>
      </div>

      <div v-if="activeTab === 'spirituil'" class="space-y-6 animate-in">
        <div :class="['p-6 rounded-[2.5rem] border bg-slate-900 border-slate-800 text-white overflow-hidden relative']">
          <p class="text-[10px] font-black uppercase text-emerald-500 mb-4 tracking-widest">Smart Quran Tracker</p>
          <div class="flex justify-between items-end mb-6">
            <h3 class="text-5xl font-black italic">{{ tilawah }} <span class="text-xs not-italic opacity-30 uppercase">/ {{ targetKhatam }} Juz</span></h3>
            <div class="text-right">
              <p class="text-[9px] font-bold opacity-40 uppercase">Saran Target</p>
              <p class="text-xl font-black text-emerald-400">{{ estimasiPerHari }} <span class="text-[10px]">Juz/Hari</span></p>
            </div>
          </div>
          <div class="w-full h-3 bg-slate-800 rounded-full mb-4 p-0.5">
            <div class="h-full bg-emerald-500 rounded-full transition-all duration-700" :style="{ width: (tilawah/targetKhatam * 100) + '%' }"></div>
          </div>
          <div class="flex justify-between items-center">
            <p class="text-[10px] font-black italic" :class="tilawah < dayNumber ? 'text-red-400' : 'text-emerald-400'">{{ statusKhatam }}</p>
            <div class="flex gap-2">
               <button @click="tilawah > 0 ? tilawah-- : null" class="w-10 h-10 bg-slate-800 rounded-xl font-black text-lg">-</button>
               <button @click="tilawah++" class="w-10 h-10 bg-emerald-500 rounded-xl text-black font-black text-lg">+</button>
            </div>
          </div>
          <div class="mt-6 pt-4 border-t border-slate-800">
            <input type="range" v-model.number="targetKhatam" min="1" max="60" class="w-full h-1.5 bg-slate-800 accent-emerald-500 rounded-full appearance-none">
          </div>
        </div>

        <div :class="['p-6 rounded-[2.5rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
          <p class="text-[10px] font-black text-emerald-500 uppercase mb-4 tracking-widest">Daily Moment & Notes</p>
          <div v-if="!dailyPhoto" class="relative border-2 border-dashed border-slate-700 rounded-3xl h-32 flex flex-col items-center justify-center cursor-pointer mb-4">
            <input type="file" @change="handlePhotoUpload" class="absolute inset-0 opacity-0 cursor-pointer" accept="image/*">
            <span class="text-2xl mb-1">📸</span>
            <span class="text-[9px] opacity-40 font-bold uppercase tracking-widest">Simpan Foto Hari Ini</span>
          </div>
          <div v-else class="relative overflow-hidden rounded-3xl h-48 mb-4 group">
            <img :src="dailyPhoto" class="w-full h-full object-cover">
            <button @click="dailyPhoto = null" class="absolute top-2 right-2 bg-black/50 w-8 h-8 rounded-full text-xs transition-colors">✕</button>
          </div>
          <textarea v-model="reflection" class="bg-transparent w-full h-24 text-sm outline-none resize-none border-t border-slate-800/20 pt-4" placeholder="Apa yang kamu syukuri hari ini, Afifah?"></textarea>
        </div>

        <div class="grid grid-cols-2 gap-4 text-left">
          <div :class="['p-6 rounded-[2rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
            <p class="text-[10px] font-black text-orange-400 mb-2 uppercase tracking-widest">Menu Sahur</p>
            <input v-model="sahurMenu" class="bg-transparent w-full text-xs font-bold outline-none" placeholder="Rencana makan...">
          </div>
          <div :class="['p-6 rounded-[2rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
            <p class="text-[10px] font-black text-emerald-500 mb-2 uppercase tracking-widest">Menu Buka</p>
            <input v-model="iftarMenu" class="bg-transparent w-full text-xs font-bold outline-none" placeholder="Rencana makan...">
          </div>
        </div>

        <button @click="showDuaModal = true" class="w-full p-8 rounded-[2.5rem] bg-gradient-to-br from-emerald-500 to-teal-600 text-white font-black italic shadow-xl active:scale-95 transition-all text-lg uppercase tracking-tight">📖 DUA LIBRARY</button>
      </div>
    </main>

    <nav :class="['fixed bottom-8 left-1/2 -translate-x-1/2 w-[90%] max-w-md p-2 rounded-[2.5rem] border flex justify-between z-50 backdrop-blur-xl transition-all', isDark ? 'bg-slate-900/90 border-slate-800' : 'bg-white/90 border-slate-200 shadow-2xl']">
      <button v-for="tab in ['dashboard', 'ibadah', 'spirituil']" :key="tab" @click="activeTab = tab"
        :class="['flex-1 py-4 rounded-[2rem] text-[10px] font-black uppercase tracking-widest transition-all duration-300', activeTab === tab ? 'bg-emerald-500 text-white' : 'opacity-20']">
        {{ tab }}
      </button>
    </nav>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&family=Amiri&display=swap');

* { 
  font-family: 'Plus Jakarta Sans', sans-serif; 
  -webkit-tap-highlight-color: transparent;
  scrollbar-width: none;
}
*::-webkit-scrollbar { display: none; }
.animate-in { animation: fadeIn 0.4s cubic-bezier(0.16, 1, 0.3, 1); }
@keyframes fadeIn { from { opacity: 0; transform: translateY(15px); } to { opacity: 1; transform: translateY(0); } }
.font-serif { font-family: 'Amiri', serif; }
input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  height: 16px; width: 16px; border-radius: 50%;
  background: #10b981; cursor: pointer;
}
</style>