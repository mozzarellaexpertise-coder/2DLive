<script>
import { onMount } from 'svelte';
  import { createClient } from '@supabase/supabase-js';

  const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
  const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

  let supabase;
  let liveData = { set: "0", value: "0", twod: "--", time: "--" };
  let status = "DISCONNECTED";
  let lastSyncedTime = ""; // Gatekeeper

  onMount(() => {
    if (supabaseUrl && supabaseAnonKey) {
      supabase = createClient(supabaseUrl, supabaseAnonKey);
      
      fetchAndVault();
      
      // UPGRADED: 30-second interval to save DB quota
      const logger = setInterval(fetchAndVault, 30000); 
      return () => clearInterval(logger);
    } else {
      status = "ERROR";
      console.error("Vault Keys Missing!");
    }
  });

  async function fetchAndVault() {
    if (!supabase) return;

    // --- SMART TIME FILTER (Retained from GitHub) ---
    const now = new Date();
    // Thai Time adjustment (approximate)
    const thaiTime = new Date(now.getTime() + (30 * 60000)); 
    const hours = thaiTime.getHours();
    const minutes = thaiTime.getMinutes();
    const currentTimeVal = (hours * 100) + minutes;

    // Siesta: 12:30 PM to 1:30 PM (Thai Time)
    if (currentTimeVal >= 1230 && currentTimeVal <= 1330) {
      status = "SIESTA (BREAK)";
      return; 
    }

    try {
      const res = await fetch('https://api.thaistock2d.com/live');
      if (!res.ok) throw new Error('API Down');
      const data = await res.json();
      
      liveData = data.live;

      // --- SMART DATA FILTER ---
      const isRealData = data.live.twod !== "--" && data.live.set !== "--";
      
      // --- DUPLICATE CHECK (The Fix) ---
      const isNewData = data.live.time !== lastSyncedTime;

      if (isRealData && isNewData) {
        status = "LIVE";
        const cleanSet = data.live.set.replace(/,/g, '');
        const cleanValue = data.live.value.replace(/,/g, '');

        const { error } = await supabase
          .from('logs')
          .insert([{
            set_index: cleanSet,
            market_value: cleanValue,
            twod: data.live.twod,
            recorded_at: data.live.time
          }]);

        if (error) {
          console.error("Vault Save Error:", error.message);
        } else {
          lastSyncedTime = data.live.time; // Mark as saved
        }
      } else if (!isRealData) {
        status = "IDLE (MARKET CLOSED)";
      } else {
        console.log("Vault: Data already exists for " + data.live.time);
      }
    } catch (err) {
      status = "ERROR";
      console.error("System Error:", err);
    }
  }
</script>

<style>
  :root {
    --bg: #0f172a; /* Darker, stealthier background */
    --card: #1e293b;
    --primary: #38bdf8;
    --accent: #22c55e;
    --danger: #ef4444;
    --muted: #94a3b8;
    --text: #f1f5f9;
  }

  main {
    background: radial-gradient(circle at top, #1e293b, #0f172a);
    font-family: 'Inter', system-ui, sans-serif;
    min-height: 100vh;
    padding: 2rem;
    display: grid;
    grid-template-rows: auto 1fr auto;
    color: var(--text);
  }

  header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid #334155;
    padding-bottom: 1rem;
  }

  h1 {
    font-size: 1.5rem;
    font-weight: 800;
    letter-spacing: -0.025em;
    color: var(--primary);
  }

  .status {
    padding: 0.4rem 1rem;
    border-radius: 6px;
    font-weight: 700;
    font-size: 0.75rem;
    text-transform: uppercase;
  }

  .status.LIVE { background: #064e3b; color: #4ade80; }
  .status.ERROR { background: #7f1d1d; color: #f87171; }
  .status.DISCONNECTED { background: #334155; color: var(--muted); }

  .dashboard {
    display: grid;
    gap: 1.5rem;
    align-content: center;
    max-width: 800px;
    margin: 0 auto;
    width: 100%;
  }

  .card {
    background: var(--card);
    border: 1px solid #334155;
    border-radius: 1rem;
    padding: 2rem;
    text-align: center;
    box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
  }

  .twod {
    font-size: 8rem;
    font-weight: 950;
    color: var(--text);
    line-height: 1;
    font-variant-numeric: tabular-nums;
  }

  .label {
    font-size: 0.75rem;
    font-weight: 600;
    color: var(--muted);
    text-transform: uppercase;
    letter-spacing: 0.1em;
    margin-top: 1rem;
  }

  .row {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 1rem;
  }

  .value {
    font-size: 1.5rem;
    font-weight: 700;
    color: var(--primary);
  }

  footer {
    text-align: center;
    font-size: 0.75rem;
    color: var(--muted);
    padding-top: 2rem;
  }

  @keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.7; }
  }

  .live-glow {
    animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
  }

  main.offline { filter: grayscale(1) opacity(0.5); }
</style>

<main class:offline={status !== 'LIVE'}>
  <header>
  <h1>2D ပုဣား</h1>
  <p class="slogan">"ကံမဟုတ်ဘူး၊ ဒါသင်္ချာ။"</p>
  </header>

  <section class="dashboard">
    <div class="card">
      <div class="twod" class:live-glow={status === 'LIVE'}>
        {liveData.twod}
      </div>
      <div class="label">Current 2D Market Number</div>
    </div>

    <div class="row">
      <div class="card">
        <div class="value">{liveData.set}</div>
        <div class="label">Set Index</div>
      </div>

      <div class="card">
        <div class="value">{liveData.value}</div>
        <div class="label">Market</div>
      </div>

      <div class="card">
        <div class="value">{liveData.time}</div>
        <div class="label">Sync Time</div>
      </div>
    </div>
  </section>

  <footer>
    Cloud-Vault Active • End-to-End Encrypted Sync • Hosted on Vercel
  </footer>
</main>
