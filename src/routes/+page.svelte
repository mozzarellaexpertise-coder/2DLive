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
    /* 🎨 Bright & Modern Palette */
    --bg-gradient: linear-gradient(135deg, #f0f9ff 0%, #e0f2fe 100%);
    --glass: rgba(255, 255, 255, 0.7);
    --glass-border: rgba(255, 255, 255, 0.5);
    --primary: #0ea5e9; /* Sky Blue */
    --accent: #10b981;  /* Emerald Green */
    --text-main: #0f172a;
    --text-muted: #64748b;
    --shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.07);
  }

  main {
    background: var(--bg-gradient);
    min-height: 100vh;
    font-family: 'Inter', sans-serif;
    color: var(--text-main);
    padding: 1.5rem;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  /* 🏛️ Modern Header with Logo */
  header {
    width: 100%;
    max-width: 800px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 2rem;
  }

  .logo-group {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .mascot {
    width: 50px;
    height: 50px;
    border-radius: 12px;
    background: white;
    box-shadow: var(--shadow);
    transition: transform 0.3s ease;
  }

  /* ⚡ Pulsing Mascot when LIVE */
  .pulse-live {
    animation: mascot-pulse 2s infinite ease-in-out;
    border: 2px solid var(--accent);
  }

  @keyframes mascot-pulse {
    0% { transform: scale(1); box-shadow: 0 0 0 0 rgba(16, 185, 129, 0.4); }
    70% { transform: scale(1.05); box-shadow: 0 0 0 10px rgba(16, 185, 129, 0); }
    100% { transform: scale(1); }
  }

  .glass-card {
    background: var(--glass);
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
    border: 1px solid var(--glass-border);
    border-radius: 24px;
    box-shadow: var(--shadow);
    padding: 2rem;
    width: 100%;
    max-width: 500px;
    text-align: center;
    margin-bottom: 1.5rem;
  }

  .big-number {
    font-size: 6rem;
    font-weight: 900;
    background: linear-gradient(to bottom, #0ea5e9, #2563eb);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    letter-spacing: -2px;
  }

  .status-badge {
    padding: 6px 16px;
    border-radius: 99px;
    font-size: 0.75rem;
    font-weight: 800;
    text-transform: uppercase;
  }

  .LIVE { background: #dcfce7; color: #166534; }
  .ERROR { background: #fee2e2; color: #991b1b; }
  .SIESTA { background: #fef9c3; color: #854d0e; }

  .grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 12px;
    width: 100%;
    max-width: 500px;
  }

  .mini-card {
    background: var(--glass);
    padding: 1rem;
    border-radius: 16px;
    border: 1px solid var(--glass-border);
    text-align: center;
  }

  .label { font-size: 0.65rem; color: var(--text-muted); font-weight: 700; text-transform: uppercase; }
  .val { font-size: 1.1rem; font-weight: 700; color: var(--primary); }
</style>

<main>
  <header>
    <div class="logo-group">
      <img 
        src="/logo.png" 
        alt="Ponnar Logo" 
        class="mascot" 
        class:pulse-live={status === 'LIVE'} 
      />
      <div>
        <h1 style="font-size: 1.2rem; margin:0; font-weight:800;">2D ပုဣား</h1>
        <span style="font-size: 0.7rem; color: var(--text-muted);">Operation 6-Series v5.5</span>
      </div>
    </div>
    <div class="status-badge {status}">{status}</div>
  </header>

  <div class="glass-card">
    <div class="label">Current Market Number</div>
    <div class="big-number">{liveData.twod}</div>
    <p style="font-size: 0.8rem; font-style: italic; color: var(--text-muted);">"ကံမဟုတ်ဘူး၊ ဒါသင်္ချာ။"</p>
  </div>

  <div class="grid">
    <div class="mini-card">
      <div class="val">{liveData.set}</div>
      <div class="label">SET</div>
    </div>
    <div class="mini-card">
      <div class="val">{liveData.value}</div>
      <div class="label">Value</div>
    </div>
    <div class="mini-card">
      <div class="val">{liveData.time}</div>
      <div class="label">Time</div>
    </div>
  </div>

  <footer style="margin-top: auto; font-size: 0.7rem; color: var(--text-muted);">
    Vercel Stealth Architecture • Cloud-Vault Active
  </footer>
</main>
