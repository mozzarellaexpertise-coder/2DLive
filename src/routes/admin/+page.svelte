<script>
  import { createClient } from '@supabase/supabase-js';

  const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
  const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;
  const supabase = createClient(supabaseUrl, supabaseAnonKey);

  let hotRows = "6x, 0x, 5x";
  let patThee = "6, 0";
  let status = "READY TO PUSH";

  async function pushToNational() {
    status = "BROADCASTING...";
    
    // We update the 'broadcast' table that the main page is listening to
    const { error } = await supabase
      .from('broadcast')
      .update({ 
        rows: hotRows, 
        pat_thee: patThee,
        updated_at: new Date() 
      })
      .eq('id', 'live_feed');

    if (error) {
      status = "ERROR: " + error.message;
    } else {
      status = "SUCCESS: SENT TO VERCEL";
      setTimeout(() => status = "READY TO PUSH", 3000);
    }
  }
</script>

<main>
  <div class="card">
    <h1>DR. GEM COMMAND CENTER</h1>
    <p>National Wide Rehearsal: Jan 14</p>

    <div class="input-group">
      <label>3 Hot Rows</label>
      <input bind:value={hotRows} placeholder="e.g. 6x, 0x, 5x" />
    </div>

    <div class="input-group">
      <label>2 Pat-Thee (ပတ်သီး)</label>
      <input bind:value={patThee} placeholder="e.g. 6, 0" />
    </div>

    <button on:click={pushToNational} class={status.includes('SUCCESS') ? 'success' : ''}>
      {status}
    </button>
  </div>
</main>

<style>
  main { min-height: 100vh; background: #0f172a; display: flex; align-items: center; justify-content: center; font-family: sans-serif; }
  .card { background: #1e293b; padding: 2rem; border-radius: 1rem; box-shadow: 0 20px 25px -5px rgba(0,0,0,0.5); width: 100%; max-width: 400px; color: white; }
  .input-group { margin-bottom: 1.5rem; }
  label { display: block; font-size: 0.8rem; color: #94a3b8; margin-bottom: 0.5rem; text-transform: uppercase; }
  input { width: 100%; padding: 0.75rem; border-radius: 0.5rem; border: 1px solid #334155; background: #0f172a; color: white; font-size: 1.1rem; }
  button { width: 100%; padding: 1rem; background: #2563eb; color: white; border: none; border-radius: 0.5rem; font-weight: bold; cursor: pointer; transition: 0.2s; }
  button:hover { background: #1d4ed8; }
  .success { background: #10b981 !important; }
</style>
