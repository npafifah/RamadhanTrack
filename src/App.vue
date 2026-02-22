<script setup lang="ts">
import { ref, watch, onMounted, computed } from "vue"
import { Bar } from 'vue-chartjs'
import { Chart as ChartJS, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale } from 'chart.js'
import * as htmlToImage from 'html-to-image'
import axios from 'axios'

ChartJS.register(Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale)

/* ================= STATE & TABS ================= */
const activeTab = ref('dashboard') 
const isDark = ref(true)

const tilawah = ref(0); const sholat = ref(0); const sedekah = ref(0); const qiyamul = ref(0)
const dzikir = ref(0); const waterIntake = ref(0); const reflection = ref("")
const sahurMenu = ref(""); const iftarMenu = ref("")
const ramadanGoals = ref(""); const isFasting = ref(true)
const prayerTimes = ref<any>(null)
const showDuaModal = ref(false)
const activeDuaTab = ref('Ramadhan')

const RAMADAN_START = new Date("2026-02-17")

const DUA_LIBRARY: any = {
  Ramadhan: [
    { title: "Niat Puasa", ar: "نَوَيْتُ صَوْمَ غَدٍ عَنْ أَدَاءِ فَرْضِ شَهْرِ رَمَضَانَ هَذِهِ السَّنَةِ لِلَّهِ تَعَالَى", tr: "Nawaitu shauma ghadin..." },
    { title: "Doa Buka Puasa", ar: "ذَهَبَ الظَّمَأُ وَابْتَلَّتِ الْعُرُوقُ وَثَبَتَ الأَجْرُ إِنْ شَاءَ اللهُ", tr: "Dzahabaz zhama'u..." },
    { title: "Doa Lailatul Qadar", ar: "اللَّهُمَّ إِنَّكَ عَفُوٌّ تُحِبُّ الْعَفْوَ فَاعْفُ عَنِّي", tr: "Ya Allah, Engkau Maha Pengampun..." }
  ],
  Harian: [
    { title: "Doa Orang Tua", ar: "رَبِّ اغْفِرْ لِي وَلِوَالِدَيَّ وَارْحَمْهُمَا كَمَا رَبَّيَانِي صَغِيرًا", tr: "Ya Allah ampuni dosaku dan orang tuaku..." },
    { title: "Sapu Jagad", ar: "رَبَّنَا آتِنَا فِي الدُwُنْيَا حَسَنَةً وَفِي الْآخِرَةِ حَسَنَةً وَقِنَا عَذَابَ النَّارِ", tr: "Ya Tuhan kami, berilah kami kebaikan..." }
  ]
}

const dailyHabits = ref([
  { id: 1, text: "Sholat Rawatib", done: false },
  { id: 2, text: "Sedekah Subuh", done: false },
  { id: 3, text: "Baca Tafsir 1 Ayat", done: false },
  { id: 4, text: "Menjaga Lisan", done: false }
])

const quotes = [
  { ar: "الصِّيَامُ جُنَّةٌ", id: "Puasa adalah perisai. (HR. Bukhari & Muslim)" },
  { ar: "ارْحَمُوا مَنْ فِي الأَرْضِ", id: "Sayangilah makhluk di bumi, niscaya yang di langit menyayangimu." },
  { ar: "خَيْرُ الناسِ أَنْفَعُهُمْ لِلنَّاسِ", id: "Sebaik-baik manusia adalah yang paling bermanfaat bagi orang lain." }
]
const randomQuote = quotes[Math.floor(Math.random() * quotes.length)]

/* ================= COMPUTED ================= */
const dayNumber = computed(() => {
  const diff = Math.floor((new Date().getTime() - RAMADAN_START.getTime()) / (1000 * 60 * 60 * 24)) + 1
  return diff > 0 ? (diff > 30 ? 30 : diff) : 1
})

const overallProgress = computed(() => {
  const p = (val: number) => Math.min(100, Math.round((val / 30) * 100))
  const mainIbadah = (p(tilawah.value) + p(sholat.value) + p(sedekah.value) + p(qiyamul.value)) / 4
  const habitPoints = (dailyHabits.value.filter(h => h.done).length / dailyHabits.value.length) * 100
  return Math.round((mainIbadah * 0.7) + (habitPoints * 0.3))
})

const userBadge = computed(() => {
  const progress = overallProgress.value
  if (progress >= 90) return { label: 'Ramadhan Legend', icon: '👑', color: '#fbbf24' }
  if (progress >= 60) return { label: 'Ramadhan Master', icon: '💎', color: '#60a5fa' }
  if (progress >= 30) return { label: 'Ramadhan Warrior', icon: '⚔️', color: '#10b981' }
  return { label: 'Ramadhan Newbie', icon: '🌱', color: '#94a3b8' }
})

const chartData = computed(() => ({
  labels: ['Quran', 'Sholat', 'Sedekah', 'Malam'],
  datasets: [{
    label: 'Progress %',
    data: [(tilawah.value/30)*100, (sholat.value/30)*100, (sedekah.value/30)*100, (qiyamul.value/30)*100],
    backgroundColor: '#10b981', borderRadius: 8
  }]
}))

/* ================= ACTIONS ================= */
const increment = (type: string) => {
  if (type === 'tilawah') tilawah.value++
  else if (type === 'sholat') sholat.value++
  else if (type === 'sedekah') sedekah.value++
  else if (type === 'qiyamul') qiyamul.value++
}

const fetchPrayers = async () => {
  try {
    const res = await axios.get('https://api.aladhan.com/v1/timingsByCity?city=Jakarta&country=Indonesia&method=2')
    prayerTimes.value = res.data.data.timings
  } catch (e) { console.error("API Error") }
}

const exportAsImage = () => {
  const node = document.getElementById('app-capture')
  if (node) {
    htmlToImage.toPng(node).then((dataUrl) => {
      const link = document.createElement('a')
      link.download = `Ramadhan-Day-${dayNumber.value}.png`
      link.href = dataUrl; link.click()
    })
  }
}

/* ================= STORAGE ================= */
onMounted(() => {
  fetchPrayers()
  const saved = localStorage.getItem("ramadhan-final-fixed")
  if (saved) {
    const data = JSON.parse(saved)
    tilawah.value = data.tilawah || 0; sholat.value = data.sholat || 0
    sedekah.value = data.sedekah || 0; qiyamul.value = data.qiyamul || 0
    dzikir.value = data.dzikir || 0; reflection.value = data.reflection || ""
    waterIntake.value = data.waterIntake || 0; sahurMenu.value = data.sahurMenu || ""
    iftarMenu.value = data.iftarMenu || ""; ramadanGoals.value = data.ramadanGoals || ""
    if (data.dailyHabits) dailyHabits.value = data.dailyHabits
  }
})

watch([tilawah, sholat, sedekah, qiyamul, dzikir, reflection, waterIntake, sahurMenu, iftarMenu, ramadanGoals, dailyHabits], () => {
  localStorage.setItem("ramadhan-final-fixed", JSON.stringify({
    tilawah: tilawah.value, sholat: sholat.value, sedekah: sedekah.value,
    qiyamul: qiyamul.value, dzikir: dzikir.value, reflection: reflection.value,
    waterIntake: waterIntake.value, sahurMenu: sahurMenu.value, iftarMenu: iftarMenu.value, 
    ramadanGoals: ramadanGoals.value, dailyHabits: dailyHabits.value
  }))
}, { deep: true })
</script>

<template>
  <div :class="['min-h-screen pb-24 transition-all duration-500', isDark ? 'bg-slate-950 text-slate-100' : 'bg-stone-50 text-slate-900']">
    
    <div v-if="showDuaModal" class="fixed inset-0 z-[60] flex items-center justify-center p-4 bg-black/80 backdrop-blur-md">
      <div :class="['w-full max-w-2xl rounded-[2.5rem] p-8 max-h-[85vh] overflow-y-auto relative', isDark ? 'bg-slate-900' : 'bg-white shadow-2xl']">
        <button @click="showDuaModal = false" class="absolute top-6 right-8 text-2xl opacity-50">✕</button>
        <h2 class="text-2xl font-black italic text-emerald-500 mb-6 text-left">Dua Library</h2>
        <div class="flex gap-4 mb-8">
          <button v-for="cat in ['Ramadhan', 'Harian']" :key="cat" @click="activeDuaTab = cat"
            :class="['px-6 py-2 rounded-full text-xs font-bold transition', activeDuaTab === cat ? 'bg-emerald-500 text-white' : 'bg-slate-800 text-slate-400']">{{ cat }}</button>
        </div>
        <div class="space-y-6">
          <div v-for="dua in DUA_LIBRARY[activeDuaTab]" :key="dua.title" class="p-6 rounded-[2rem] border border-emerald-500/10 bg-emerald-500/5 text-left">
            <h4 class="text-[10px] font-black uppercase text-emerald-500 mb-2">{{ dua.title }}</h4>
            <p class="text-2xl text-right font-serif leading-loose mb-4">{{ dua.ar }}</p>
            <p class="text-xs opacity-60 italic">{{ dua.tr }}</p>
          </div>
        </div>
      </div>
    </div>

    <header class="p-6 flex justify-between items-center max-w-4xl mx-auto">
      <div class="text-left">
        <h2 class="text-2xl font-black italic text-emerald-500 tracking-tighter">RamadhanTrack</h2>
        <div class="flex items-center gap-2">
          <span class="text-[10px] font-bold opacity-50 uppercase tracking-[0.3em]">Day {{ dayNumber }}</span>
          <span :style="{ color: userBadge.color }" class="text-[10px] font-black uppercase tracking-widest">
            • {{ userBadge.icon }} {{ userBadge.label }}
          </span>
        </div>
      </div>
      <button @click="isDark = !isDark" class="w-12 h-12 rounded-2xl bg-slate-800/50 flex items-center justify-center text-xl">
        {{ isDark ? '☀️' : '🌙' }}
      </button>
    </header>

    <main id="app-capture" class="max-w-4xl mx-auto px-6 space-y-6">
      
      <div v-if="activeTab === 'dashboard'" class="space-y-6 animate-in">
        <div class="text-center p-8 bg-emerald-500/5 border border-emerald-500/10 rounded-[2.5rem]">
          <p class="text-3xl font-serif text-emerald-500 mb-2">{{ randomQuote.ar }}</p>
          <p class="text-[10px] opacity-50 italic">{{ randomQuote.id }}</p>
        </div>

        <div :class="['p-8 rounded-[2.5rem] border overflow-hidden relative text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-xl']">
          <p class="text-[10px] font-black text-emerald-500 uppercase mb-2 tracking-widest">Total Progress</p>
          <div class="flex items-end gap-2 mb-4">
            <span class="text-6xl font-black italic tracking-tighter">{{ overallProgress }}%</span>
            <span class="text-xs opacity-40 pb-2">Completed</span>
          </div>
          <div class="w-full bg-slate-800 h-2 rounded-full overflow-hidden">
            <div class="bg-emerald-500 h-full transition-all duration-1000" :style="{ width: overallProgress + '%' }"></div>
          </div>
        </div>

        <div :class="['p-6 rounded-[2.5rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
          <p class="text-[10px] font-black text-emerald-500 uppercase mb-4 tracking-widest">Daily Kebaikan</p>
          <div class="space-y-3">
            <div v-for="habit in dailyHabits" :key="habit.id" @click="habit.done = !habit.done"
              :class="['p-4 rounded-2xl border transition-all flex justify-between items-center cursor-pointer', habit.done ? 'bg-emerald-500/20 border-emerald-500' : 'bg-slate-800/20 border-slate-700 opacity-50']">
              <span class="text-sm font-bold">{{ habit.text }}</span>
              <span v-if="habit.done">✅</span>
              <span v-else class="w-5 h-5 border-2 border-slate-600 rounded-full"></span>
            </div>
          </div>
        </div>

        <div class="grid grid-cols-2 gap-4">
          <div v-for="n in ['Imsak', 'Maghrib']" :key="n" :class="['p-6 rounded-[2rem] border text-center', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white']">
            <p class="text-[10px] font-bold opacity-40 uppercase mb-1">{{ n }} Jakarta</p>
            <p class="text-2xl font-black text-emerald-500">{{ prayerTimes ? prayerTimes[n] : '--:--' }}</p>
          </div>
        </div>
        
        <button @click="exportAsImage" class="w-full py-4 bg-emerald-500 text-white rounded-[2rem] font-black text-xs uppercase tracking-[0.2em] shadow-lg shadow-emerald-500/20 active:scale-95 transition-transform">
          📸 Share Progress Hari Ini
        </button>
      </div>

      <div v-if="activeTab === 'ibadah'" class="space-y-6 animate-in">
        <div :class="['p-6 rounded-[2.5rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-lg']">
           <div class="h-48"><Bar :data="chartData" :options="{responsive: true, maintainAspectRatio: false}" /></div>
        </div>

        <div class="grid grid-cols-2 gap-4">
          <div v-for="item in [{id:'tilawah', l:'Quran', v: tilawah}, {id:'sholat', l:'Sholat', v: sholat}, {id:'sedekah', l:'Sedekah', v: sedekah}, {id:'qiyamul', l:'Tarawih', v: qiyamul}]" :key="item.id"
            :class="['p-6 rounded-[2.5rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
            <p class="text-[10px] font-black opacity-40 uppercase mb-4">{{ item.l }}</p>
            <div class="flex items-center justify-between">
              <span class="text-4xl font-black italic tracking-tighter">{{ item.v }}</span>
              <button @click="increment(item.id)" class="w-12 h-12 bg-emerald-500 rounded-2xl text-white font-bold active:scale-90 transition-transform">+</button>
            </div>
          </div>
        </div>

        <div :class="['p-8 rounded-[3rem] border bg-emerald-500/5 border-emerald-500/20 text-center']">
          <p class="text-[10px] font-black text-emerald-500 uppercase mb-2 tracking-widest">Tasbih Digital</p>
          <div class="text-7xl font-black mb-6 font-mono tracking-tighter">{{ dzikir }}</div>
          <button @click="dzikir++" class="w-full py-8 bg-emerald-500 rounded-[2.5rem] text-2xl font-black shadow-xl active:scale-95 transition-all shadow-emerald-500/20">TAP</button>
          <button @click="dzikir = 0" class="mt-4 text-[10px] opacity-30 font-bold uppercase">Reset</button>
        </div>

        <div :class="['p-8 rounded-[2.5rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white']">
          <p class="text-[10px] font-black text-blue-500 uppercase mb-4">Minum Air (8 Gelas)</p>
          <div class="grid grid-cols-8 gap-2">
            <button v-for="i in 8" :key="i" @click="waterIntake = i" 
              :class="['h-10 rounded-xl transition-all', waterIntake >= i ? 'bg-blue-500 shadow-lg shadow-blue-500/20' : 'bg-slate-800 opacity-20']"></button>
          </div>
        </div>
      </div>

      <div v-if="activeTab === 'spirituil'" class="space-y-6 animate-in">
        <div :class="['p-8 rounded-[2.5rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-md']">
          <h3 class="text-[10px] font-black text-emerald-500 uppercase mb-4 tracking-widest">Target Ramadhan</h3>
          <input v-model="ramadanGoals" placeholder="Contoh: Khatam Al-Quran 2x" class="bg-transparent w-full text-xl font-medium outline-none border-b border-emerald-500/20 pb-2">
        </div>

        <div :class="['p-8 rounded-[2.5rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-xl']">
          <h3 class="text-[10px] font-black text-emerald-500 uppercase mb-4 tracking-widest italic">Catatan Muhasabah</h3>
          <textarea v-model="reflection" class="w-full bg-transparent resize-none outline-none text-sm leading-relaxed h-40" placeholder="Apa hikmah yang didapat hari ini?"></textarea>
        </div>

        <div class="grid grid-cols-2 gap-4">
          <div :class="['p-6 rounded-[2rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
            <p class="text-[10px] font-black text-orange-400 mb-2 uppercase tracking-widest">Menu Sahur</p>
            <textarea v-model="sahurMenu" rows="2" class="bg-transparent w-full text-xs outline-none resize-none" placeholder="Tulis menu..."></textarea>
          </div>
          <div :class="['p-6 rounded-[2rem] border text-left', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
            <p class="text-[10px] font-black text-emerald-500 mb-2 uppercase tracking-widest">Menu Buka</p>
            <textarea v-model="iftarMenu" rows="2" class="bg-transparent w-full text-xs outline-none resize-none" placeholder="Tulis menu..."></textarea>
          </div>
        </div>

        <button @click="showDuaModal = true" class="w-full p-8 rounded-[3rem] bg-gradient-to-br from-emerald-500 to-teal-600 text-white text-center shadow-xl shadow-emerald-500/20 active:scale-95 transition-transform">
          <h3 class="text-2xl font-black italic mb-1 uppercase">📖 Buka Kumpulan Doa</h3>
        </button>
      </div>

    </main>

    <nav :class="['fixed bottom-6 left-1/2 -translate-x-1/2 w-[90%] max-w-md p-2 rounded-[2.5rem] border flex justify-between z-50 backdrop-blur-xl', isDark ? 'bg-slate-900/80 border-slate-800' : 'bg-white/90 border-slate-200 shadow-2xl']">
      <button v-for="tab in ['dashboard', 'ibadah', 'spirituil']" :key="tab" @click="activeTab = tab"
        :class="['flex-1 py-4 rounded-full text-[10px] font-black uppercase tracking-tighter transition-all', activeTab === tab ? 'bg-emerald-500 text-white shadow-lg shadow-emerald-500/20' : 'opacity-40']">
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