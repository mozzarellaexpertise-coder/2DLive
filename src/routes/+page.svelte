<script>
  import { onMount } from 'svelte';
  import { createClient } from '@supabase/supabase-js';

  const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
  const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;
  const supabase = createClient(supabaseUrl, supabaseAnonKey);

  let liveData = { set: "0", value: "0", twod: "--", time: "--" };
  let broadcast = { rows: "SCANNING...", pat_thee: "--" };
  let status = "DISCONNECTED";
  let lastSyncedTime = "";

  onMount(() => {
    if (supabaseUrl && supabaseAnonKey) {
      status = "CONNECTING...";
      fetchAndVault();
      setupRealtime();
      const logger = setInterval(fetchAndVault, 30000);
      return () => clearInterval(logger);
    } else {
      status = "ERROR";
    }
  });

  // REALTIME SUBSCRIBER: Listens for your SQL God Mode updates
  function setupRealtime() {
    const channel = supabase
      .channel('public:broadcast')
      .on('postgres_changes', { event: 'UPDATE', schema: 'public', table: 'broadcast' }, payload => {
        broadcast = payload.new;
      })
      .subscribe();

    // Initial fetch of the prediction
    supabase.from('broadcast').select('*').eq('id', 'live_feed').single().then(({ data }) => {
      if (data) broadcast = data;
    });
  }

  async function fetchAndVault() {
    const now = new Date();
    const thaiTime = new Date(now.getTime() + (30 * 60000));
    const currentTimeVal = thaiTime.getHours() * 100 + thaiTime.getMinutes();

    if (currentTimeVal >= 1230 && currentTimeVal <= 1330) {
      status = "SIESTA";
      return;
    }

    try {
      const res = await fetch('https://api.thaistock2d.com/live');
      const data = await res.json();
      liveData = data.live;
      status = data.live.twod !== "--" ? "LIVE" : "IDLE";

      if (status === "LIVE" && data.live.time !== lastSyncedTime) {
        await supabase.from('logs').insert([{
          set_index: data.live.set.replace(/,/g, ''),
          market_value: data.live.value.replace(/,/g, ''),
          twod: data.live.twod,
          recorded_at: data.live.time
        }]);
        lastSyncedTime = data.live.time;
      }
    } catch (err) {
      status = "ERROR";
    }
  }
</script>

<main>
  <header>
    <div class="logo-group">
      <div class="status-dot {status}"></div>
      <div>
        <strong>2D ပုဣား LIVE</strong><br />
        <small>National Wide • v5.5</small>
      </div>
    </div>
    <div class="status-badge {status}">{status}</div>
  </header>

  <section class="broadcast-container">
    <div class="mascot-anchor">
      <img src="/pointing.png" alt="Ponnar" class="pointing-img" />
    </div>

    <div class="tactical-display">
      <div class="display-header">
        <span class="live-dot"></span> 
        DR. GEM NEURAL SCAN: JAN 14
      </div>
      <div class="display-body">
        <div class="row-box">
          <label>HOT ROWS</label>
          <div class="value">{broadcast.rows}</div>
        </div>
        <div class="pat-thee-box">
          <label>ပတ်သီး (PAT-THEE)</label>
          <div class="value">{broadcast.pat_thee}</div>
        </div>
        <div class="status-ticker">
          STREAK: <span class="highlight">100% DEFENSE ACTIVE</span>
        </div>
      </div>
    </div>
  </section>

  <section class="glass hero">
    <div class="label">Current Market Number</div>
    <div class="big-number">{liveData.twod}</div>
    <em class="myanmar-text">ကံမဟုတ်ဘူး၊ ဒါသင်္ချာ။</em>
  </section>

  <section class="grid">
    <div class="glass mini"><div class="val">{liveData.set}</div><div class="label">SET</div></div>
    <div class="glass mini"><div class="val">{liveData.value}</div><div class="label">Value</div></div>
    <div class="glass mini"><div class="val">{liveData.time}</div><div class="label">Time</div></div>
  </section>

  <footer>Vercel Stealth Architecture • Cloud-Vault Active</footer>
</main>

<style>
  :root {
    --bg: #0f172a;
    --primary: #38bdf8;
    --gold: #fbbf24;
  }

  main {
    min-height: 100svh;
    background: radial-gradient(circle at top, #1e293b, #0f172a);
    padding: 1rem;
    display: flex;
    flex-direction: column;
    align-items: center;
    color: white;
    font-family: 'Inter', sans-serif;
  }

  header { width: 100%; max-width: 600px; display: flex; justify-content: space-between; margin-bottom: 2rem; }
  
  .status-dot { width: 10px; height: 10px; border-radius: 50%; background: #64748b; }
  .status-dot.LIVE { background: #10b981; box-shadow: 0 0 10px #10b981; }

  .broadcast-container {
    display: flex;
    align-items: center;
    gap: 1rem;
    width: 100%;
    max-width: 800px;
    margin-bottom: 2rem;
  }

  .mascot-anchor { flex: 1; }
  .pointing-img { width: 100%; filter: drop-shadow(0 0 20px rgba(56, 189, 248, 0.2)); }

  .tactical-display {
    flex: 1.5;
    background: rgba(30, 41, 59, 0.7);
    border: 2px solid var(--gold);
    border-radius: 16px;
    backdrop-filter: blur(10px);
    overflow: hidden;
  }

  .display-header { background: var(--gold); color: #000; padding: 4px 12px; font-size: 0.7rem; font-weight: 900; }
  .display-body { padding: 1.2rem; }
  
  label { font-size: 0.6rem; color: #94a3b8; letter-spacing: 1px; }
  .value { font-size: 2rem; font-weight: 900; color: #fff; margin-bottom: 0.5rem; }
  
  .glass { background: rgba(255,255,255,0.05); border: 1px solid rgba(255,255,255,0.1); border-radius: 20px; }
  .hero { width: 100%; max-width: 500px; text-align: center; padding: 2rem; margin-bottom: 1rem; }
  .big-number { font-size: 5rem; font-weight: 900; color: var(--primary); }
  
  .grid { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 0.5rem; width: 100%; max-width: 500px; }
  .mini { padding: 1rem; text-align: center; }
  
  @media (max-width: 600px) {
    .broadcast-container { flex-direction: column-reverse; }
    .big-number { font-size: 3.5rem; }
  }
</style>
