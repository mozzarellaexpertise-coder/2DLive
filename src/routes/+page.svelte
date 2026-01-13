<script>
  import { onMount } from 'svelte';
  import { createClient } from '@supabase/supabase-js';

  const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
  const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

  let supabase;
  let liveData = { set: "0", value: "0", twod: "--", time: "--" };
  let status = "DISCONNECTED";
  let lastSyncedTime = "";

  onMount(() => {
    if (supabaseUrl && supabaseAnonKey) {
      supabase = createClient(supabaseUrl, supabaseAnonKey);
      fetchAndVault();
      const logger = setInterval(fetchAndVault, 30000);
      return () => clearInterval(logger);
    } else {
      status = "ERROR";
      console.error("Vault Keys Missing!");
    }
  });

  async function fetchAndVault() {
    if (!supabase) return;

    const now = new Date();
    const thaiTime = new Date(now.getTime() + (30 * 60000));
    const currentTimeVal = thaiTime.getHours() * 100 + thaiTime.getMinutes();

    if (currentTimeVal >= 1230 && currentTimeVal <= 1330) {
      status = "SIESTA";
      return;
    }

    try {
      const res = await fetch('https://api.thaistock2d.com/live');
      if (!res.ok) throw new Error('API Down');
      const data = await res.json();

      liveData = data.live;

      const isRealData = data.live.twod !== "--";
      const isNewData = data.live.time !== lastSyncedTime;

      if (isRealData && isNewData) {
        status = "LIVE";

        const { error } = await supabase.from('logs').insert([{
          set_index: data.live.set.replace(/,/g, ''),
          market_value: data.live.value.replace(/,/g, ''),
          twod: data.live.twod,
          recorded_at: data.live.time
        }]);

        if (!error) lastSyncedTime = data.live.time;
      } else if (!isRealData) {
        status = "IDLE";
      }
    } catch (err) {
      status = "ERROR";
      console.error(err);
    }
  }
</script>

<style>
  :root {
    --bg: linear-gradient(135deg, #f0f9ff, #e0f2fe);
    --glass: rgba(255,255,255,.72);
    --border: rgba(255,255,255,.5);
    --primary: #0ea5e9;
    --accent: #10b981;
    --text: #0f172a;
    --muted: #64748b;
    --shadow: 0 10px 30px rgba(0,0,0,.08);
  }

  main {
    min-height: 100svh;
    background: var(--bg);
    padding: env(safe-area-inset-top) 1rem env(safe-area-inset-bottom);
    display: flex;
    flex-direction: column;
    align-items: center;
    font-family: Inter, system-ui, sans-serif;
    color: var(--text);
  }

  header {
    width: 100%;
    max-width: 900px;
    display: flex;
    gap: 1rem;
    justify-content: space-between;
    align-items: center;
    flex-wrap: wrap;
    margin: 1rem 0 1.5rem;
  }

  .logo-group {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .mascot {
    width: 44px;
    height: 44px;
    border-radius: 12px;
    background: white;
    box-shadow: var(--shadow);
  }

  .pulse-live {
    animation: pulse 2s infinite;
    border: 2px solid var(--accent);
  }

  @keyframes pulse {
    0% { box-shadow: 0 0 0 0 rgba(16,185,129,.4); }
    70% { box-shadow: 0 0 0 14px rgba(16,185,129,0); }
    100% { box-shadow: 0 0 0 0; }
  }

  .status-badge {
    padding: 6px 14px;
    border-radius: 99px;
    font-size: .7rem;
    font-weight: 800;
  }

  .LIVE { background:#dcfce7; color:#166534 }
  .ERROR { background:#fee2e2; color:#991b1b }
  .SIESTA, .IDLE { background:#fef9c3; color:#854d0e }

  .glass {
    background: var(--glass);
    border: 1px solid var(--border);
    border-radius: 22px;
    box-shadow: var(--shadow);
  }

  .hero {
    width: 100%;
    max-width: 520px;
    padding: clamp(1.2rem, 4vw, 2rem);
    text-align: center;
    margin-bottom: 1.2rem;
  }

  .big-number {
    font-size: clamp(3.5rem, 14vw, 6rem);
    font-weight: 900;
    background: linear-gradient(#0ea5e9, #2563eb);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    line-height: 1;
  }

  .grid {
    width: 100%;
    max-width: 520px;
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(120px,1fr));
    gap: .8rem;
  }

  .mini {
    padding: .9rem;
    text-align: center;
  }

  .label {
    font-size: .65rem;
    text-transform: uppercase;
    font-weight: 700;
    color: var(--muted);
  }

  .val {
    font-size: 1.1rem;
    font-weight: 800;
    color: var(--primary);
  }

  footer {
    margin-top: auto;
    font-size: .7rem;
    color: var(--muted);
    padding: 1rem 0;
  }
</style>

<main>
  <header>
    <div class="logo-group">
      <img src="/logo.png" class="mascot" class:pulse-live={status === 'LIVE'} />
      <div>
        <strong>2D ပုဣား</strong><br />
        <small style="color:var(--muted)">Operation 6-Series v5.5</small>
      </div>
    </div>
    <div class="status-badge {status}">{status}</div>
  </header>

  <section class="glass hero">
    <div class="label">Current Market Number</div>
    <div class="big-number">{liveData.twod}</div>
    <em style="font-size:.8rem;color:var(--muted)">ကံမဟုတ်ဘူး၊ ဒါသင်္ချာ။</em>
  </section>

  <section class="grid">
    <div class="glass mini">
      <div class="val">{liveData.set}</div>
      <div class="label">SET</div>
    </div>
    <div class="glass mini">
      <div class="val">{liveData.value}</div>
      <div class="label">Value</div>
    </div>
    <div class="glass mini">
      <div class="val">{liveData.time}</div>
      <div class="label">Time</div>
    </div>
  </section>

  <footer>
    Vercel Stealth Architecture • Cloud-Vault Active
  </footer>
</main>
