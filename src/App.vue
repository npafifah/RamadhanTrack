<script setup lang="ts">
import { ref, watch, onMounted, computed } from "vue"

/* ================= CONSTANTS & DATA ================= */
const DAYS = 30
const JUZ = 30
const RAMADAN_START = new Date("2026-02-17")
const RAMADAN_END = new Date("2026-03-18")

const DUAS = [
  "Allahumma laka sumtu wa bika amantu...", // Day 1
  "Allahumma qowwini fihi 'ala iqomati amrik...", // Day 2
  // ... bisa dilanjutkan sampai 30
]

/* ================= STATE ================= */
const isDark = ref(true)
const tilawah = ref(0)
const sholat = ref(0)
const sedekah = ref(0)
const qiyamul = ref(0)
const dzikir = ref(0)
const reflection = ref("")
const waterIntake = ref(0)
const currentMood = ref("😊")
const sahurMenu = ref("")
const iftarMenu = ref("")
const ramadanGoals = ref("")
const isFasting = ref(true)

const habits = ref([
  { text: "Sholat Rawatib", done: false },
  { text: "Baca Tafsir 1 Ayat", done: false },
  { text: "No Social Media (Detox)", done: false },
  { text: "Maafkan Orang Lain", done: false }
])

const stats: Record<string, any> = { tilawah, sholat, sedekah, qiyamul }

/* ================= COMPUTED ================= */
const dayNumber = computed(() => {
  const diff = Math.floor((new Date().getTime() - RAMADAN_START.getTime()) / (1000 * 60 * 60 * 24)) + 1
  return diff > 0 ? (diff > 30 ? 30 : diff) : 0
})

const overallProgress = computed(() => {
  const p = (val: number, max: number) => Math.min(100, Math.round((val / max) * 100))
  return Math.round((p(tilawah.value, JUZ) + p(sholat.value, DAYS) + p(sedekah.value, DAYS) + p(qiyamul.value, DAYS)) / 4)
})

const isLastTenDays = computed(() => dayNumber.value > 20)

/* ================= STORAGE ================= */
onMounted(() => {
  const saved = localStorage.getItem("ramadhan-ultimate-data")
  if (saved) {
    const data = JSON.parse(saved)
    Object.assign(this, data) // Simple way to restore if state matches
    tilawah.value = data.tilawah || 0
    sholat.value = data.sholat || 0
    sedekah.value = data.sedekah || 0
    qiyamul.value = data.qiyamul || 0
    dzikir.value = data.dzikir || 0
    reflection.value = data.reflection || ""
    waterIntake.value = data.waterIntake || 0
    currentMood.value = data.currentMood || "😊"
    sahurMenu.value = data.sahurMenu || ""
    iftarMenu.value = data.iftarMenu || ""
    ramadanGoals.value = data.ramadanGoals || ""
    if(data.habits) habits.value = data.habits
  }
})

watch([tilawah, sholat, sedekah, qiyamul, dzikir, reflection, waterIntake, currentMood, sahurMenu, iftarMenu, ramadanGoals, habits, isFasting], () => {
  localStorage.setItem("ramadhan-ultimate-data", JSON.stringify({
    tilawah: tilawah.value, sholat: sholat.value, sedekah: sedekah.value,
    qiyamul: qiyamul.value, dzikir: dzikir.value, reflection: reflection.value,
    waterIntake: waterIntake.value, currentMood: currentMood.value,
    sahurMenu: sahurMenu.value, iftarMenu: iftarMenu.value,
    ramadanGoals: ramadanGoals.value, habits: habits.value, isFasting: isFasting.value
  }))
}, { deep: true })

/* ================= ACTIONS ================= */
const inc = (id: string, max: number) => { if (stats[id].value < max) stats[id].value++ }
const dec = (id: string) => { if (stats[id].value > 0) stats[id].value-- }
</script>

<template>
  <div :class="['min-h-screen p-4 md:p-8 transition-all duration-500', isDark ? 'bg-slate-950 text-slate-100' : 'bg-stone-50 text-slate-900']">
    
    <main class="max-w-7xl mx-auto grid grid-cols-1 lg:grid-cols-12 gap-6">
      
      <div class="lg:col-span-3 space-y-6">
        <div :class="['p-6 rounded-[2.5rem] border relative overflow-hidden', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-xl border-slate-100']">
          <button @click="isDark = !isDark" class="absolute top-4 right-4 text-xl">{{ isDark ? '☀️' : '🌙' }}</button>
          <div class="w-20 h-20 rounded-full bg-emerald-500/20 flex items-center justify-center text-3xl mb-4">🌙</div>
          <h2 class="text-xl font-black italic">Ramadhan 2026</h2>
          <p class="text-xs opacity-50">Day {{ dayNumber }} of 30</p>
          
          <div class="mt-6 space-y-2">
            <div class="flex justify-between text-xs font-bold uppercase tracking-widest text-emerald-500">
              <span>Spirit Level</span>
              <span>{{ overallProgress }}%</span>
            </div>
            <div class="w-full bg-slate-800 h-1.5 rounded-full overflow-hidden">
              <div class="bg-emerald-500 h-full transition-all duration-1000" :style="{ width: overallProgress + '%' }"></div>
            </div>
          </div>
        </div>

        <div :class="['p-6 rounded-[2rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-lg border-slate-100']">
          <div class="flex justify-between items-center mb-6">
            <span class="text-sm font-bold">Fasting?</span>
            <button @click="isFasting = !isFasting" :class="['px-4 py-1 rounded-full text-[10px] font-black', isFasting ? 'bg-emerald-500' : 'bg-rose-500']">
              {{ isFasting ? 'YES' : 'NO' }}
            </button>
          </div>
          <p class="text-[10px] font-bold opacity-40 uppercase mb-3">Today's Mood</p>
          <div class="flex justify-between bg-slate-800/30 p-2 rounded-2xl">
            <button v-for="m in ['😊','😇','😴','😐','😫']" :key="m" @click="currentMood = m" 
              :class="['w-10 h-10 rounded-xl transition', currentMood === m ? 'bg-emerald-500 text-xl' : 'opacity-40 hover:opacity-100']">{{ m }}</button>
          </div>
        </div>

        <div :class="['p-6 rounded-[2rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-lg border-slate-100']">
          <h3 class="text-sm font-bold mb-4 flex items-center gap-2">💧 Water Intake <span class="text-xs opacity-40">(Goal: 8)</span></h3>
          <div class="grid grid-cols-4 gap-2">
            <button v-for="i in 8" :key="i" @click="waterIntake = i" 
              :class="['h-8 rounded-lg transition-all', waterIntake >= i ? 'bg-blue-500 shadow-lg shadow-blue-500/30' : 'bg-slate-800 opacity-20']"></button>
          </div>
        </div>
      </div>

      <div class="lg:col-span-6 space-y-6">
        <div :class="['p-6 rounded-[2rem] border relative overflow-hidden group', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white border-slate-100 shadow-lg']">
           <div class="absolute top-0 right-0 p-8 text-6xl opacity-5 pointer-events-none">✨</div>
           <h3 class="font-bold text-emerald-500 mb-2 uppercase text-xs tracking-widest">Ramadhan Goals</h3>
           <input v-model="ramadanGoals" placeholder="What do you want to achieve this month?" class="bg-transparent w-full text-xl font-medium outline-none border-none p-0 placeholder:opacity-20">
        </div>

        <div class="grid grid-cols-2 gap-4">
          <div v-for="(item, id) in { tilawah: {t:'Quran', i:'📖', u:'Juz'}, sholat: {t:'Prayers', i:'🕌', u:'Days'}, sedekah: {t:'Charity', i:'🤲', u:'Days'}, qiyamul: {t:'Night', i:'🌙', u:'Night'} }" :key="id"
            :class="['p-6 rounded-[2rem] border transition-all', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white border-slate-100 shadow-sm']">
            <div class="text-2xl mb-2">{{ item.i }}</div>
            <p class="text-[10px] font-bold opacity-40 uppercase tracking-widest mb-1">{{ item.t }}</p>
            <div class="flex items-end gap-2">
              <span class="text-3xl font-black leading-none">{{ stats[id].value }}</span>
              <span class="text-[10px] opacity-30 pb-1">/ {{ item.u }}</span>
            </div>
            <div class="flex gap-2 mt-4">
              <button @click="dec(id as string)" class="flex-1 py-2 bg-slate-800 rounded-xl hover:bg-slate-700">-</button>
              <button @click="inc(id as string, 30)" class="flex-1 py-2 bg-emerald-500 rounded-xl hover:bg-emerald-400 text-white">+</button>
            </div>
          </div>
        </div>

        <div :class="['p-6 rounded-[2rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white border-slate-100 shadow-lg']">
          <h3 class="font-bold mb-4 text-emerald-500 flex items-center gap-2">✅ Sunnah Habits</h3>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-3">
            <div v-for="(h, i) in habits" :key="i" @click="h.done = !h.done" 
              :class="['p-4 rounded-2xl cursor-pointer border transition-all flex items-center gap-3', h.done ? 'bg-emerald-500/10 border-emerald-500/30 opacity-60' : 'bg-slate-800/40 border-transparent']">
              <div :class="['w-4 h-4 rounded-full border-2', h.done ? 'bg-emerald-500 border-emerald-500' : 'border-slate-500']"></div>
              <span :class="['text-xs font-bold', h.done ? 'line-through' : '']">{{ h.text }}</span>
            </div>
          </div>
        </div>

        <div class="grid grid-cols-2 gap-4">
          <div :class="['p-6 rounded-[2rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
            <p class="text-[10px] font-bold text-orange-400 mb-2 uppercase tracking-tighter">Sahur Plan</p>
            <textarea v-model="sahurMenu" rows="2" class="bg-transparent w-full text-xs outline-none resize-none" placeholder="Healthy meal..."></textarea>
          </div>
          <div :class="['p-6 rounded-[2rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white shadow-sm']">
            <p class="text-[10px] font-bold text-emerald-500 mb-2 uppercase tracking-tighter">Iftar Plan</p>
            <textarea v-model="iftarMenu" rows="2" class="bg-transparent w-full text-xs outline-none resize-none" placeholder="Dates and water..."></textarea>
          </div>
        </div>
      </div>

      <div class="lg:col-span-3 space-y-6">
        <div :class="['p-6 rounded-[2rem] border text-center transition-all', isLastTenDays ? 'border-emerald-500 ring-2 ring-emerald-500/20 shadow-2xl shadow-emerald-500/20' : 'border-slate-800 bg-slate-900']">
          <p class="text-[10px] font-bold text-emerald-500 uppercase mb-4">{{ isLastTenDays ? '🔥 LAILATUL QADR HUNT' : 'Dhikr Counter' }}</p>
          <div class="text-6xl font-black mb-6">{{ dzikir }}</div>
          <button @click="dzikir++" class="w-full py-6 bg-emerald-500 rounded-[1.5rem] font-black text-xl hover:scale-95 active:scale-90 transition shadow-lg">TAP</button>
          <button @click="dzikir = 0" class="mt-4 text-[10px] opacity-30 hover:opacity-100 font-bold uppercase">Reset</button>
        </div>

        <div :class="['p-6 rounded-[2rem] border', isDark ? 'bg-slate-900 border-slate-800' : 'bg-white border-slate-100 shadow-lg']">
          <h3 class="font-bold mb-4 flex items-center gap-2 italic">📝 My Reflection</h3>
          <textarea v-model="reflection" class="w-full bg-transparent resize-none outline-none text-xs leading-loose" rows="8" placeholder="What made you grateful today?"></textarea>
        </div>
      </div>

    </main>

    <footer class="mt-16 text-center">
      <div class="text-[10px] font-bold opacity-20 uppercase tracking-[1em] mb-4">May Allah Accept Our Deeds</div>
      <div class="flex justify-center gap-2 opacity-10">
        <div class="w-2 h-2 rounded-full bg-emerald-500"></div>
        <div class="w-2 h-2 rounded-full bg-emerald-500"></div>
        <div class="w-2 h-2 rounded-full bg-emerald-500"></div>
      </div>
    </footer>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;800&display=swap');

* {
  font-family: 'Plus Jakarta Sans', sans-serif;
}

textarea::placeholder, input::placeholder {
  color: currentColor;
  opacity: 0.3;
}
</style>