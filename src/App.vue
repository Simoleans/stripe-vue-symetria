<script setup>
import { ref, computed, onMounted } from 'vue'

const API_BASE = import.meta.env.VITE_API_BASE || 'https://TU-API.com'

// === planes visibles en el front (solo informativo, el backend valida los price_id) ===
const PLANES = {
  'elite-max-mensual': { title: 'Elite Max (Mensual)', priceText: 'USD 79 / mes' },
  'elite-max-anual':   { title: 'Elite Max (Anual)',   priceText: 'USD 799 / año' },
  'elite-prueba':      { title: 'Elite Prueba',        priceText: 'USD 15 / mes' },
  'standar-plus':      { title: 'Standar Plus',        priceText: 'USD 60 / mes' },
  'elite-plus-mxn':    { title: 'Elite Plus (MXN)',    priceText: 'MXN 4,000 / mes' },
  'standar-mxn':       { title: 'Standar (MXN)',       priceText: 'MXN 3,224.85 / mes' },
  'medical-beauty':    { title: 'Medical Beauty Marketing', priceText: '—' },
}

// === estados ===
const qs = new URLSearchParams(location.search)
const planSlug = ref(qs.get('plan') || null)
const sid = qs.get('sid') // si vienes del success

const planInfo = computed(() => (planSlug.value ? PLANES[planSlug.value] : null))
const isSuccessView = computed(() => !!sid)

const loading = ref(false)
const error = ref(null)
const paid = ref(null)
const amount = ref(null)
const currency = ref('USD')

// === iniciar checkout ===
async function pagar() {
  if (!planSlug.value || !PLANES[planSlug.value]) {
    error.value = 'Plan inválido'
    return
  }

  loading.value = true
  error.value = null

  try {
    const utm = {
      utm_source: qs.get('utm_source') || '',
      utm_medium: qs.get('utm_medium') || '',
      utm_campaign: qs.get('utm_campaign') || '',
      fbclid: qs.get('fbclid') || '',
    }

    const r = await fetch(`${API_BASE}/api/stripe/checkout`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        productId: planSlug.value,
        quantity: 1,
        metadata: { source: 'learnworlds' },
        utm,
      }),
    })
    if (!r.ok) throw new Error(await r.text())
    const { url } = await r.json()
    window.location.href = url
  } catch (e) {
    error.value = e.message || 'No se pudo iniciar el checkout'
  } finally {
    loading.value = false
  }
}

// === en success page, verificar sesión y disparar pixel ===
onMounted(async () => {
  if (!sid) return
  try {
    const r = await fetch(`${API_BASE}/api/stripe/session?sid=${encodeURIComponent(sid)}`)
    if (!r.ok) throw new Error(await r.text())
    const s = await r.json()
    paid.value = s.paid === true
    amount.value = s.amount_total
    currency.value = (s.currency || 'usd').toUpperCase()

    if (paid.value && window.fbq) {
      window.fbq('track', 'Purchase', {
        value: (amount.value || 0) / 100,
        currency: currency.value,
      })
    }
  } catch (e) {
    error.value = e.message
    paid.value = false
  }
})
</script>

<template>
  <div class="max-w-3xl mx-auto px-6 py-12 text-center">
    <!-- Vista de éxito -->
    <section v-if="isSuccessView">
      <h1 class="text-3xl font-bold mb-4">¡Gracias por tu compra!</h1>

      <p v-if="paid === true" class="text-green-700">Pago confirmado ✅</p>
      <p v-else-if="paid === false" class="text-yellow-700">Pago aún no confirmado…</p>

      <p v-if="amount" class="mt-2 text-gray-700">
        Monto: {{ (amount/100).toFixed(2) }} {{ currency }}
      </p>

      <p v-if="error" class="text-red-600 mt-4">{{ error }}</p>
    </section>

    <!-- Vista de checkout -->
    <section v-else>
      <h1 class="text-3xl font-bold mb-6 bg-red-500">Finalizar suscripción</h1>

      <div v-if="planInfo" class="border rounded-xl p-6 bg-white">
        <h2 class="text-xl font-semibold">{{ planInfo.title }}</h2>
        <p class="text-gray-600 mb-4">{{ planInfo.priceText }}</p>

        <button
          class="bg-blue-600 hover:bg-blue-700 text-white px-6 py-3 rounded-lg"
          :disabled="loading"
          @click="pagar"
        >
          {{ loading ? 'Redirigiendo…' : 'Pagar con Stripe' }}
        </button>

        <p v-if="error" class="text-red-600 mt-4">{{ error }}</p>
      </div>

      <div v-else class="text-red-600">
        ⚠️ No se encontró un plan válido en la URL
      </div>
    </section>
  </div>
</template>
