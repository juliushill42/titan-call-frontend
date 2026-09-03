<script>
  import { animate } from 'motion';
  import { onMount } from 'svelte';

  const API = import.meta.env.VITE_TITAN_CALL_API || 'https://api.geturclarity.net';
  const nav = ['Home', 'Calls', 'Devices', 'Network', 'Setup', 'Logs', 'Settings'];
  let page = 'Home';
  let online = false;
  let loading = true;
  let error = '';
  let system = {};
  let calls = [];
  let devices = [];
  let network = {};
  let events = [];
  let lastRefresh = null;
  let menuOpen = false;

  async function api(path) {
    const r = await fetch(`${API}${path}`, { credentials: 'include' });
    if (!r.ok) throw new Error(`${r.status} ${r.statusText}`);
    return r.json();
  }

  async function refresh() {
    loading = true;
    error = '';
    try {
      const [health, sys, callData, deviceData, netData] = await Promise.all([
        api('/health'),
        api('/api/v1/system'),
        api('/api/v1/calls'),
        api('/api/v1/devices'),
        api('/api/v1/network')
      ]);
      online = health?.status === 'ok' || health?.status === 'healthy';
      system = sys || {};
      calls = Array.isArray(callData) ? callData : (callData?.calls || []);
      devices = Array.isArray(deviceData) ? deviceData : (deviceData?.devices || []);
      network = netData || {};
      lastRefresh = new Date();
    } catch (e) {
      online = false;
      error = `Control API not connected: ${e.message}`;
    } finally {
      loading = false;
    }
  }

  function select(next) {
    page = next;
    menuOpen = false;
    requestAnimationFrame(() => {
      const el = document.querySelector('.page');
      if (el) animate(el, { opacity: [0.15, 1], transform: ['translateY(8px)', 'translateY(0px)'] }, { duration: 0.32 });
    });
  }

  function fmtTime(v) {
    if (!v) return '—';
    const d = new Date(v);
    return Number.isNaN(d.getTime()) ? String(v) : d.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
  }

  function age(v) {
    if (!v) return '—';
    const ms = Date.now() - new Date(v).getTime();
    if (!Number.isFinite(ms)) return '—';
    const s = Math.max(0, Math.floor(ms / 1000));
    if (s < 60) return `${s}s ago`;
    if (s < 3600) return `${Math.floor(s/60)}m ago`;
    return `${Math.floor(s/3600)}h ago`;
  }

  function callState(c) {
    return c.state || c.status || 'unknown';
  }

  function deviceName(d) {
    return d.name || d.user || d.device || d.id || 'Unnamed device';
  }

  onMount(() => {
    refresh();
    const timer = setInterval(refresh, 15000);
    let es;
    try {
      es = new EventSource(`${API}/api/v1/events`, { withCredentials: true });
      es.onmessage = (e) => {
        try { events = [JSON.parse(e.data), ...events].slice(0, 30); } catch {}
      };
    } catch {}
    return () => { clearInterval(timer); es?.close(); };
  });
</script>

<svelte:head>
  <title>Titan Call · Communication Control</title>
</svelte:head>

<div class="shell">
  <aside class:open={menuOpen}>
    <div class="brand">
      <div class="brand-mark"><span>:</span></div>
      <div><b>Titan Call</b><small>GetUrClarity.net</small></div>
    </div>

    <nav>
      {#each nav as item}
        <button class:active={page === item} on:click={() => select(item)}>
          <span class="nav-dot"></span>{item}
        </button>
      {/each}
    </nav>

    <div class="sidebar-foot">
      <div class="live-line"><i class:up={online}></i><span>{online ? 'System online' : 'API disconnected'}</span></div>
      <small>{lastRefresh ? `Updated ${fmtTime(lastRefresh)}` : 'Waiting for live data'}</small>
    </div>
  </aside>

  <main>
    <header>
      <button class="menu" on:click={() => menuOpen = !menuOpen}>☰</button>
      <div class="eyebrow">COMMUNICATION CONTROL</div>
      <div class="header-actions">
        <div class="connection"><i class:up={online}></i>{online ? 'LIVE' : 'OFFLINE'}</div>
        <button class="refresh" on:click={refresh} disabled={loading}>{loading ? 'Syncing…' : 'Refresh'}</button>
      </div>
    </header>

    {#if error}
      <div class="warning"><b>Backend not connected.</b><span>{error}</span><code>{API}</code></div>
    {/if}

    <section class="page">
      {#if page === 'Home'}
        <div class="hero">
          <div>
            <div class="kicker">PRIVATE COMMUNICATION, ONE VIEW</div>
            <h1>See every call.<br/><em>Know every connection.</em></h1>
            <p>Calls, phones and private network health from the real Titan Call system. No simulated data.</p>
          </div>
          <div class="orb-wrap" aria-hidden="true">
            <div class="orb"><span></span><span></span><span></span><b>:</b></div>
          </div>
        </div>

        <div class="stats">
          <article><span>Phones online</span><strong>{devices.filter(d => d.status === 'online' || d.online === true).length}</strong><small>{devices.length} known devices</small></article>
          <article><span>Calls now</span><strong>{calls.filter(c => ['active','ringing','connected'].includes(callState(c))).length}</strong><small>{calls.length} current records</small></article>
          <article><span>Private network</span><strong>{network.online === true || network.status === 'online' ? 'Up' : '—'}</strong><small>{network.peers?.length ?? network.peer_count ?? 0} visible peers</small></article>
          <article><span>Control service</span><strong>{online ? 'Live' : '—'}</strong><small>{system.version ? `v${system.version}` : 'version from server'}</small></article>
        </div>

        <div class="grid-two">
          <article class="panel calls-panel">
            <div class="panel-head"><div><span>LIVE CALLS</span><h2>What’s happening now</h2></div><button on:click={() => select('Calls')}>View all</button></div>
            {#if calls.length}
              <div class="call-list">
                {#each calls.slice(0,5) as c}
                  <div class="call-row"><div class="avatar">{(c.from || c.caller || '?').slice(0,1).toUpperCase()}</div><div class="who"><b>{c.from || c.caller || 'Unknown'}</b><small>to {c.to || c.callee || 'Unknown'}</small></div><span class="transport">{c.transport || 'call'}</span><span class="state {callState(c)}">{callState(c)}</span></div>
                {/each}
              </div>
            {:else}
              <div class="empty">No active calls reported.</div>
            {/if}
          </article>

          <article class="panel network-panel">
            <div class="panel-head"><div><span>PRIVATE NETWORK</span><h2>Connection map</h2></div><button on:click={() => select('Network')}>Details</button></div>
            <div class="network-map">
              <div class="core"><b>:</b><small>Titan Call</small></div>
              {#each devices.slice(0,6) as d, i}
                <div class="node n{i+1}"><i class:up={d.status === 'online' || d.online}></i><span>{deviceName(d)}</span></div>
              {/each}
              <svg viewBox="0 0 800 330" preserveAspectRatio="none">
                {#each devices.slice(0,6) as _, i}<line x1="400" y1="165" x2={[110,220,600,690,210,600][i]} y2={[65,275,55,260,70,275][i]} />{/each}
              </svg>
            </div>
          </article>
        </div>

        <article class="panel activity">
          <div class="panel-head"><div><span>RECENT ACTIVITY</span><h2>System events</h2></div></div>
          {#if events.length}
            {#each events.slice(0,8) as e}<div class="event"><i></i><div><b>{e.type || e.event || 'Update'}</b><small>{e.message || e.detail || 'System event received'}</small></div><time>{fmtTime(e.time || e.timestamp)}</time></div>{/each}
          {:else}<div class="empty">Live events will appear here when the API stream is connected.</div>{/if}
        </article>

      {:else if page === 'Calls'}
        <div class="page-title"><div><span>CALLS</span><h1>Call activity</h1><p>Current sessions reported by Titan Call.</p></div></div>
        <article class="panel table-panel">
          <div class="table-wrap"><table><thead><tr><th>From</th><th>To</th><th>State</th><th>Transport</th><th>Started</th></tr></thead><tbody>
          {#each calls as c}<tr><td><b>{c.from || c.caller || '—'}</b></td><td>{c.to || c.callee || '—'}</td><td><span class="state {callState(c)}">{callState(c)}</span></td><td>{c.transport || '—'}</td><td>{fmtTime(c.started_at || c.startedAt)}</td></tr>{/each}
          </tbody></table>{#if !calls.length}<div class="empty">No calls reported.</div>{/if}</div>
        </article>

      {:else if page === 'Devices'}
        <div class="page-title"><div><span>DEVICES</span><h1>Connected phones</h1><p>Only devices reported by the live system are shown.</p></div></div>
        <div class="device-grid">
          {#each devices as d}
            <article class="device-card"><div class="phone-icon">▯</div><div class="device-top"><span class="state-dot" class:up={d.status === 'online' || d.online}></span><small>{d.status || (d.online ? 'online' : 'offline')}</small></div><h3>{deviceName(d)}</h3><p>{d.platform || 'Titan Call device'}</p><dl><div><dt>Address</dt><dd>{d.address || d.mesh_ip || '—'}</dd></div><div><dt>Version</dt><dd>{d.version || '—'}</dd></div><div><dt>Last seen</dt><dd>{age(d.last_seen || d.updated_at)}</dd></div></dl></article>
          {/each}
          {#if !devices.length}<div class="empty panel">No devices reported.</div>{/if}
        </div>

      {:else if page === 'Network'}
        <div class="page-title"><div><span>NETWORK</span><h1>Private connection</h1><p>Actual private-network status from the Titan Call host.</p></div></div>
        <div class="stats"><article><span>Status</span><strong>{network.status || (network.online ? 'Online' : '—')}</strong></article><article><span>Peers</span><strong>{network.peers?.length ?? network.peer_count ?? 0}</strong></article><article><span>Host</span><strong class="small-strong">{network.self?.hostname || network.hostname || '—'}</strong></article><article><span>Address</span><strong class="small-strong">{network.self?.ip || network.ip || '—'}</strong></article></div>
        <article class="panel table-panel"><div class="panel-head"><div><span>PEERS</span><h2>Visible network members</h2></div></div><div class="table-wrap"><table><thead><tr><th>Name</th><th>Address</th><th>Status</th></tr></thead><tbody>{#each network.peers || [] as p}<tr><td><b>{p.hostname || p.name || '—'}</b></td><td>{p.ip || p.address || '—'}</td><td><span class="state {p.online ? 'active' : 'offline'}">{p.online ? 'online' : 'offline'}</span></td></tr>{/each}</tbody></table>{#if !(network.peers || []).length}<div class="empty">No peer data reported.</div>{/if}</div></article>

      {:else if page === 'Setup'}
        <div class="page-title"><div><span>SETUP</span><h1>Add a phone</h1><p>Enrollment is generated by the backend and expires automatically.</p></div></div>
        <article class="panel setup-card"><div class="setup-visual"><div class="qr-frame"><div class="qr-placeholder">QR</div></div></div><div><span class="kicker">ONE-TIME ENROLLMENT</span><h2>Pair a phone securely</h2><p>For security, the dashboard does not manufacture enrollment codes in the browser. Connect the control API to generate the one-time QR.</p><button class="primary" disabled={!online}>Generate enrollment</button></div></article>

      {:else if page === 'Logs'}
        <div class="page-title"><div><span>LOGS</span><h1>System activity</h1><p>Live event stream from Titan Call.</p></div></div>
        <article class="panel logbox">{#each events as e}<div><time>{fmtTime(e.time || e.timestamp)}</time><code>{e.type || e.event || 'event'}</code><span>{e.message || e.detail || JSON.stringify(e)}</span></div>{/each}{#if !events.length}<div class="empty">No events received yet.</div>{/if}</article>

      {:else}
        <div class="page-title"><div><span>SETTINGS</span><h1>System details</h1><p>Runtime information reported by Titan Call.</p></div></div>
        <div class="settings-grid"><article class="panel"><span class="kicker">API</span><h2>Control address</h2><code class="big-code">{API}</code></article><article class="panel"><span class="kicker">VERSION</span><h2>{system.version || 'Not reported'}</h2><p>{system.service || 'Titan Call control service'}</p></article></div>
      {/if}
    </section>
  </main>
</div>
