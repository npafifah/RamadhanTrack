<script setup lang="ts">
import { ref, watch, onMounted, computed } from "vue"

const isDark = ref(true)

onMounted(() => {
  const savedTheme = localStorage.getItem("theme")
  if (savedTheme) {
    isDark.value = savedTheme === "dark"
  }
})

watch(isDark, (val) => {
  localStorage.setItem("theme", val ? "dark" : "light")
})
const tilawah = ref(0)
const tilawahTarget = 30

const sholat = ref(0)
const sholatTarget = 30

const sedekah = ref(0)
const qiyamul = ref(0)
const tilawahPercent = computed(() =>
  Math.round((tilawah.value / tilawahTarget) * 100)
)

const sholatPercent = computed(() =>
  Math.round((sholat.value / sholatTarget) * 100)
)

function getStatus(percent: number) {
  if (percent >= 80) return "🔥 Konsisten"
  if (percent >= 50) return "⚡ Lumayan"
  return "🚀 Semangat!"
}

onMounted(() => {
  const savedData = localStorage.getItem("ramadhan-data")
  if (savedData) {
    const parsed = JSON.parse(savedData)
    tilawah.value = parsed.tilawah
    sholat.value = parsed.sholat
    sedekah.value = parsed.sedekah
    qiyamul.value = parsed.qiyamul
  }
})

watch(
  [tilawah, sholat, sedekah, qiyamul],
  () => {
    localStorage.setItem(
      "ramadhan-data",
      JSON.stringify({
        tilawah: tilawah.value,
        sholat: sholat.value,
        sedekah: sedekah.value,
        qiyamul: qiyamul.value,
      })
    )
  },
  { deep: true }
)

function increaseTilawah() {
  if (tilawah.value < tilawahTarget) tilawah.value++
}
function decreaseTilawah() {
  if (tilawah.value > 0) tilawah.value--
}

function increaseSholat() {
  if (sholat.value < sholatTarget) sholat.value++
}
function decreaseSholat() {
  if (sholat.value > 0) sholat.value--
}
</script>

<template>
  <div
  :class="[
    'min-h-screen p-8 transition-colors duration-500',
    isDark ? 'bg-slate-900 text-white' : 'bg-gray-100 text-slate-900'
  ]"
>
    
    <header class="mb-10">
      <h1 class="text-4xl font-bold text-emerald-400">
        RamadhanTrack 🌙
      </h1>
      <p class="text-slate-400 mt-2">
        Smart Ibadah Productivity Dashboard
      </p>
      <button
  @click="isDark = !isDark"
  class="absolute top-8 right-8 px-4 py-2 rounded-lg bg-emerald-500 hover:bg-emerald-400 transition"
>
  {{ isDark ? '☀ Light Mode' : '🌙 Dark Mode' }}
</button>
    </header>

    <section class="grid md:grid-cols-2 gap-6">

      <div class="bg-slate-800 p-6 rounded-2xl shadow-lg">
  <h2 class="text-lg font-semibold text-emerald-400">
    📖 Tilawah
  </h2>

  <p class="text-3xl font-bold mt-4">
    {{ tilawah }} / {{ tilawahTarget }} Juz
  </p>

  <!-- Persentase -->
  <p class="mt-2 text-emerald-400 font-semibold">
    {{ tilawahPercent }}% • {{ getStatus(tilawahPercent) }}
  </p>

  <!-- Progress bar -->
  <div class="w-full bg-slate-700 h-3 rounded-full mt-4">
    <div
      class="bg-emerald-400 h-3 rounded-full transition-all duration-500"
      :style="{ width: tilawahPercent + '%' }"
    ></div>
  </div>

  <!-- Buttons -->
  <div class="flex gap-3 mt-6">
    <button
      @click="decreaseTilawah"
      class="bg-slate-700 px-4 py-2 rounded-lg hover:bg-slate-600 transition"
    >
      ➖
    </button>
    <button
      @click="increaseTilawah"
      class="bg-emerald-500 px-4 py-2 rounded-lg hover:bg-emerald-400 transition"
    >
      ➕
    </button>
  </div>
</div>

      <!-- Sholat -->
      <div class="bg-slate-800 p-6 rounded-2xl shadow-lg">
  <h2 class="text-lg font-semibold text-emerald-400">
    🕌 Sholat Tepat Waktu
  </h2>

  <p class="text-3xl font-bold mt-4">
    {{ sholat }} / {{ sholatTarget }} Hari
  </p>

  <p class="mt-2 text-emerald-400 font-semibold">
    {{ sholatPercent }}% • {{ getStatus(sholatPercent) }}
  </p>

  <div class="w-full bg-slate-700 h-3 rounded-full mt-4">
    <div
      class="bg-emerald-400 h-3 rounded-full transition-all duration-500"
      :style="{ width: sholatPercent + '%' }"
    ></div>
  </div>

  <div class="flex gap-3 mt-6">
    <button
      @click="decreaseSholat"
      class="bg-slate-700 px-4 py-2 rounded-lg hover:bg-slate-600 transition"
    >
      ➖
    </button>
    <button
      @click="increaseSholat"
      class="bg-emerald-500 px-4 py-2 rounded-lg hover:bg-emerald-400 transition"
    >
      ➕
    </button>
  </div>
</div>

      <!-- Sedekah -->
      <div class="bg-slate-800 p-6 rounded-2xl shadow-lg">
        <h2 class="text-lg font-semibold text-emerald-400">
          🤲 Sedekah
        </h2>
        <p class="text-3xl font-bold mt-4">
          {{ sedekah }} Hari
        </p>
      </div>

      <!-- Qiyamul Lail -->
      <div class="bg-slate-800 p-6 rounded-2xl shadow-lg">
        <h2 class="text-lg font-semibold text-emerald-400">
          🌙 Qiyamul Lail
        </h2>
        <p class="text-3xl font-bold mt-4">
          {{ qiyamul }} Malam
        </p>
      </div>

    </section>
  </div>
</template>