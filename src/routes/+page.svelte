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

      if (status === "LIVE" && data.live.time !== lastSyncedTime) {
        await supabase.from('logs').insert([{
          set_index: data.live.set.replace(/,/g, ''),
          market_value: data.live.value.replace(/,/g, ''),
          twod: data.live.twod,
          recorded_at: data.live.time
        }])
        lastSyncedTime = data.live.time
      }
    } catch {
      status = "ERROR"
    }
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
    <em class="myanmar-text">ကံမဟုတ်ဘူး၊ ဒါသင်္ချာ။</em>
  </section>

  <!-- GURU BROADCAST -->
  <section class="guru-panel">
    <div class="panel-title">Guru Guidance</div>

    <div class="panel-row">
      <label>HOT ROWS</label>
      <div class="panel-value">{broadcast.rows}</div>
    </div>

    <div class="panel-row">
      <label>PAT-THEE</label>
      <div class="panel-value">{broadcast.pat_thee}</div>
    </div>
  </section>

  <!-- MARKET META -->
  <section class="meta-grid">
    <div>{liveData.set}<span>SET</span></div>
    <div>{liveData.value}<span>VALUE</span></div>
    <div>{liveData.time}<span>TIME</span></div>
  </section>

  <footer>Guru Engine • Supabase Realtime • v6.0</footer>
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

  /* HEADER */
  .guru-header {
    width: 100%;
    max-width: 420px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 1rem;
  }

  .subtitle {
    font-size: 0.7rem;
    color: var(--muted);
  }

  .status-pill {
    padding: 4px 12px;
    border-radius: 999px;
    font-size: 0.65rem;
    background: #333;
  }

  .status-pill.LIVE {
    background: var(--saffron);
    color: #000;
  }

  /* MAIN FOCUS */
  .guru-focus {
    width: 100%;
    max-width: 420px;
    text-align: center;
    padding: 1.5rem;
    border-radius: 24px;
    border: 1px solid var(--gold);
    margin-bottom: 1rem;
  }

  .guru-number {
    font-size: 4.5rem;
    font-weight: 900;
    color: var(--saffron);
    letter-spacing: 2px;
  }

  /* PANEL */
  .guru-panel {
    width: 100%;
    max-width: 420px;
    background: var(--card);
    border-radius: 18px;
    padding: 1rem;
    border: 1px solid rgba(212,175,55,0.4);
  }

  .panel-title {
    font-size: 0.7rem;
    letter-spacing: 2px;
    color: var(--gold);
    margin-bottom: 0.5rem;
  }

  .panel-row {
    margin-bottom: 0.75rem;
  }

  .panel-value {
    font-size: 1.8rem;
    font-weight: 800;
  }

  /* META GRID */
  .meta-grid {
    width: 100%;
    max-width: 420px;
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 0.5rem;
    margin-top: 1rem;
  }

  .meta-grid div {
    background: #1f2933;
    padding: 0.75rem;
    border-radius: 14px;
    text-align: center;
    font-weight: 700;
  }

  .meta-grid span {
    display: block;
    font-size: 0.6rem;
    color: var(--muted);
  }

  footer {
    margin-top: auto;
    font-size: 0.6rem;
    color: var(--muted);
  }
</style>
