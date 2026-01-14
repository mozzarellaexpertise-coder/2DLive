<script>
  import { onMount } from 'svelte'
  import { createClient } from '@supabase/supabase-js'

  const supabaseUrl = import.meta.env.VITE_SUPABASE_URL
  const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY
  const supabase = createClient(supabaseUrl, supabaseAnonKey)

  let liveData = { set: "0", value: "0", twod: "--", time: "--" }
  let broadcast = { rows: "SCANNING...", pat_thee: "--" }
  let status = "DISCONNECTED"
  let lastSyncedTime = ""

  /* ===== GRAPH STATE ===== */
  let graphPoints = []
  const MAX_POINTS = 30

  onMount(() => {
    if (!supabaseUrl || !supabaseAnonKey) {
      status = "ERROR"
      return
    }

    status = "CONNECTING..."
    fetchAndVault()
    setupRealtime()

    const timer = setInterval(fetchAndVault, 30000)
    return () => clearInterval(timer)
  })

  function setupRealtime() {
    supabase
      .channel('public:broadcast')
      .on(
        'postgres_changes',
        { event: 'UPDATE', schema: 'public', table: 'broadcast' },
        payload => broadcast = payload.new
      )
      .subscribe()

    supabase
      .from('broadcast')
      .select('*')
      .eq('id', 'live_feed')
      .single()
      .then(({ data }) => data && (broadcast = data))
  }

  async function fetchAndVault() {
    const now = new Date()
    const thaiTime = new Date(now.getTime() + 30 * 60000)
    const timeVal = thaiTime.getHours() * 100 + thaiTime.getMinutes()

    if (timeVal >= 1230 && timeVal <= 1330) {
      status = "SIESTA"
      return
    }

    try {
      const res = await fetch('https://api.thaistock2d.com/live')
      const data = await res.json()
      liveData = data.live
      status = data.live.twod !== "--" ? "LIVE" : "IDLE"

      if (
        status === "LIVE" &&
        data.live.time !== lastSyncedTime &&
        data.live.twod !== "--"
      ) {
        const numeric = Number(data.live.twod)

        if (!isNaN(numeric)) {
          graphPoints = [...graphPoints, numeric].slice(-MAX_POINTS)
        }

        await supabase.from('logs').insert([{
          set_index: data.live.set.replace(/,/g, ''),
          market_value: data.live.value.replace(/,/g, ''),
          twod: data.live.twod,
          recorded_at: data.live.time
        }])

        lastSyncedTime = data.live.time
      }

    } catch (err) {
      console.error("Vault Error:", err)
      status = "ERROR"
    }
  }

  /* ===== SVG GRAPH BUILDER ===== */
  function buildPath(points, width = 300, height = 120) {
    if (points.length < 2) return ''

    const min = Math.min(...points)
    const max = Math.max(...points)
    const range = max - min || 1

    return points.map((p, i) => {
      const x = (i / (points.length - 1)) * width
      const y = height - ((p - min) / range) * height
      return `${i === 0 ? 'M' : 'L'} ${x} ${y}`
    }).join(' ')
  }
</script>

<main>

  <!-- HEADER -->
  <header class="guru-header">
    <div>
      <strong>2D Guru Live</strong>
      <div class="subtitle">National Market Scan</div>
    </div>
    <div class="status-pill {status}">{status}</div>
  </header>

  <!-- MAIN NUMBER -->
  <section class="guru-focus">
    <div class="label">Today’s Signal</div>
    <div class="guru-number">{liveData.twod}</div>

    <!-- 🔥 LIVE GRAPH -->
    <section class="guru-graph">
      <svg viewBox="0 0 300 120" preserveAspectRatio="none">
        <path
          d="{buildPath(graphPoints)}"
          fill="none"
          stroke="#ff9933"
          stroke-width="3"
          stroke-linecap="round"
          stroke-linejoin="round"
        />
      </svg>
    </section>

    <em class="myanmar-text">ကံမဟုတ်ဘူး၊ ဒါသင်္ချာ။</em>
  </section>

  <!-- PREDICTION + PONNNAR -->
  <section class="prediction-wrap">
    <div class="guru-panel half">
      <div class="panel-title">Guru Guidance</div>

      <div class="panel-row">
        <label>ထိပ်စီး</label>
        <div class="panel-value">{broadcast.rows}</div>
      </div>

      <div class="panel-row">
        <label>ပတ်သီး</label>
        <div class="panel-value">{broadcast.pat_thee}</div>
      </div>
    </div>

    <div class="ponnar">
      <img src="/pointing.png" alt="Ponnnar Sir" />
    </div>
  </section>

  <!-- META -->
  <section class="meta-grid">
    <div>{liveData.set}<span>SET</span></div>
    <div>{liveData.value}<span>VALUE</span></div>
    <div>{liveData.time}<span>TIME</span></div>
  </section>

  <footer>Guru Engine • Supabase Realtime • v7.0</footer>
</main>

<style>
  :root {
    --bg: #1b1b1b;
    --card: #2a1f14;
    --saffron: #ff9933;
    --gold: #d4af37;
    --sand: #f3e8d1;
    --muted: #9ca3af;
  }

  main {
    min-height: 100svh;
    background: radial-gradient(circle, #2a1f14, #1b1b1b);
    padding: 1rem;
    color: var(--sand);
    font-family: system-ui, sans-serif;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .guru-header {
    width: 100%;
    max-width: 420px;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .subtitle { font-size: 0.7rem; color: var(--muted); }

  .status-pill {
    padding: 4px 12px;
    border-radius: 999px;
    font-size: 0.65rem;
    background: #333;
  }

  .status-pill.LIVE { background: var(--saffron); color: #000; }

  .guru-focus {
    width: 100%;
    max-width: 420px;
    text-align: center;
    padding: 1.5rem;
    border-radius: 24px;
    border: 1px solid var(--gold);
    margin-top: 1rem;
  }

  .guru-number {
    font-size: 4.5rem;
    font-weight: 900;
    color: var(--saffron);
  }

  .guru-graph {
    height: 120px;
    margin-top: 1rem;
    padding: 0.5rem;
    background: rgba(0,0,0,0.35);
    border-radius: 16px;
    border: 1px solid rgba(255,153,51,0.35);
  }

  .guru-graph svg { width: 100%; height: 100%; }

  .prediction-wrap {
    width: 100%;
    max-width: 420px;
    display: flex;
    gap: 0.5rem;
    margin-top: 1rem;
  }

  .ponnar img {
    width: 100%;
    max-height: 180px;
    object-fit: contain;
  }

  footer {
    margin-top: auto;
    font-size: 0.6rem;
    color: var(--muted);
  }
</style>
