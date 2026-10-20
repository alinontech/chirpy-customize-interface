<script setup>
  import { onBeforeUnmount, ref } from 'vue'
  import { vueEmbedComponent } from '@knowlearning/agents/vue.js'

  const copy = x => JSON.parse(JSON.stringify(x))

  // Same pass rule as the KL player. Candli closes once per page load, so after
  // a Replay we only re-report a passing run; partial checkpoint saves are ignored.
  const PASS_RATIO = 0.85

  const { id } = defineProps({ id: String })

  const candliLink = ref(null)
  let closedOnce = false
  let reportedPass = false
  let resendTimer = null
  let unmounted = false

  Agent
    .state(id)
    .then(async ({ game, configuration }) => {
      const { variables } = await Agent.environment()
      variables.configuration = configuration
      variables.configurationId = id
      const urlParams = new URLSearchParams(window.location.search)
      const langParam = urlParams.get('lang')

      const lang = langParam || variables.LANGUAGES?.[0]?.split('-')[0] || 'en'
      candliLink.value = `https://pila.cand.li/pila-play.html?game=${game}&lang=${lang}`
    })

  async function handleClose(info) {
    await Agent.synced()
    const latestCompetencies = await Agent.state(`pila/latest_competencies/${id}`)
    const competencies = copy(latestCompetencies)
    if (Agent.embedded) Agent.close({ competencies, success: info?.success })
    reportedPass = reportedPass || passes(competencies)
    closedOnce = true
    await clearOutLatestCompetencies()
  }

  function handleMutate({ scope }) {
    if (
      !closedOnce
      || !Agent.embedded
      || reportedPass
      || !String(scope).endsWith(`pila/latest_competencies/${id}`)
    ) return
    clearTimeout(resendTimer)
    resendTimer = setTimeout(resendIfPassed, 1500)
  }

  async function resendIfPassed() {
    await Agent.synced()
    if (unmounted) return
    const scores = copy(await Agent.state(`pila/latest_competencies/${id}`))
    if (!reportedPass && passes(scores)) {
      reportedPass = true
      Agent.close({ competencies: scores, success: true })
    }
  }

  function passes(competencies) {
    let earned = 0
    let total = 0
    Object.keys(competencies || {}).forEach(key => {
      if (key === 'general:attempts') return
      const score = competencies[key]?.[0]
      const max = competencies[key]?.[1]
      if (typeof score !== 'number' || typeof max !== 'number') return
      earned += score
      total += max
    })
    return total > 0 && earned / total > PASS_RATIO
  }

  onBeforeUnmount(() => {
    unmounted = true
    clearTimeout(resendTimer)
  })

  async function clearOutLatestCompetencies() {
    const competencies = await Agent.state(`pila/latest_competencies/${id}`)
    Object.keys(competencies).forEach(key => delete competencies[key])
  }
</script>

<template>
  <div
    style="
      width: 100vw;
      height: 100vh;
      overflow: hidden;
    "
  >
    <vueEmbedComponent
      v-if="candliLink"
      :id="candliLink"
      @close="handleClose"
      @mutate="handleMutate"
    />
  </div>
</template>
