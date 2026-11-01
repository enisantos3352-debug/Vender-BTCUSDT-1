
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Binance Real-Time Short Strategy - GitHub Pages</title>
  <style>
    :root {
      --bg: #0b0e11;
      --card: #1e2329;
      --text: #eaecef;
      --green: #0ecb81;
      --red: #f6465d;
      --accent: #f0b90b;
      --subtext: #848e9c;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
      background-color: var(--bg);
      color: var(--text);
      margin: 0;
      padding: 20px;
      display: flex;
      justify-content: center;
    }

    .container {
      max-width: 700px;
      width: 100%;
      display: flex;
      flex-direction: column;
      gap: 16px;
    }

    .card {
      background-color: var(--card);
      border-radius: 8px;
      padding: 16px;
      border: 1px solid #2b313a;
    }

    .header-ticker {
      display: flex;
      justify-content: space-between;
      align-items: center;
      flex-wrap: wrap;
      gap: 12px;
    }

    .price-large {
      font-size: 2rem;
      font-weight: bold;
    }

    .status-badge {
      padding: 6px 12px;
      border-radius: 4px;
      font-weight: bold;
      font-size: 0.85rem;
    }

    .status-stopped { background: rgba(132, 142, 156, 0.2); color: var(--subtext); }
    .status-short { background: rgba(246, 70, 93, 0.2); color: var(--red); }
    .status-done { background: rgba(14, 203, 129, 0.2); color: var(--green); }

    .btn-start {
      background-color: var(--green);
      color: #000;
      border: none;
      padding: 12px 24px;
      font-size: 1rem;
      font-weight: bold;
      border-radius: 6px;
      cursor: pointer;
      transition: opacity 0.2s;
    }

    .btn-start:hover {
      opacity: 0.9;
    }

    .btn-start:disabled {
      background-color: var(--subtext);
      cursor: not-allowed;
    }

    .targets-list {
      display: flex;
      flex-direction: column;
      gap: 8px;
      margin-top: 12px;
    }

    .target-item {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 10px 14px;
      background: #14181d;
      border-radius: 6px;
      border-left: 4px solid var(--subtext);
    }

    .target-item.hit {
      border-left-color: var(--green);
      background: rgba(14, 203, 129, 0.08);
    }

    .target-item.active {
      border-left-color: var(--accent);
      background: rgba(240, 185, 11, 0.08);
    }

    .log-box {
      background: #14181d;
      border-radius: 6px;
      padding: 12px;
      height: 180px;
      overflow-y: auto;
      font-family: monospace;
      font-size: 0.82rem;
      color: #b7bdc6;
    }

    .log-entry { margin-bottom: 4px; }
    .log-success { color: var(--green); }
    .log-warn { color: var(--accent); }
    .log-danger { color: var(--red); }
  </style>
</head>
<body>

<div class="container">
  <!-- Preço em Tempo Real & Controle de Start -->
  <div class="card header-ticker">
    <div>
      <div style="color: var(--subtext); font-size: 0.9rem;">BTC/USDT Binance Live</div>
      <div class="price-large" id="live-price">Conectando...</div>
    </div>
    <div style="display: flex; align-items: center; gap: 12px;">
      <span id="position-status" class="status-badge status-stopped">AGUARDANDO START</span>
      <button id="btn-action" class="btn-start" onclick="toggleStrategy()">START</button>
    </div>
  </div>

  <!-- Lista Sequencial de Alvos -->
  <div class="card">
    <h3 style="margin-top: 0; color: var(--accent);">Sequência de Execução de Alvos</h3>
    <div class="targets-list" id="targets-container"></div>
  </div>

  <!-- Log em Tempo Real -->
  <div class="card">
    <h3 style="margin-top: 0; font-size: 1rem; color: var(--subtext);">Log de Operação</h3>
    <div class="log-box" id="log-box"></div>
  </div>
</div>

<script>
  // Sequência exata dos alvos conforme os prints fornecidos
  const TARGETS = [
    { id: 'central', name: '1. Ponto Central (Média Máx/Mín)', price: 77938.615, hit: false },
    { id: 'maxima', name: '2. Máxima do Período', price: 78039.83, hit: false },
    { id: 'minima', name: '3. Mínima do Período', price: 77837.40, hit: false },
    { id: 'ultimo', name: '4. Último Fechamento Fixo', price: 77442.00, hit: false },
    { id: 'pivo', name: '5. Preço Pivô', price: 77601.00, hit: false },
    { id: 's1', name: '6. Suporte 1', price: 75712.00, hit: false },
    { id: 's2', name: '7. Suporte 2', price: 75299.00, hit: false },
    { id: 's3', name: '8. Suporte 3', price: 74887.00, hit: false }
  ];

  let currentTargetIndex = 0;
  let isRunning = false;
  let currentPrice = 0;

  function log(msg, type = '') {
    const logBox = document.getElementById('log-box');
    const time = new Date().toLocaleTimeString();
    const entry = document.createElement('div');
    entry.className = `log-entry ${type}`;
    entry.innerText = `[${time}] ${msg}`;
    logBox.appendChild(entry);
    logBox.scrollTop = logBox.scrollHeight;
  }

  function renderTargets() {
    const container = document.getElementById('targets-container');
    container.innerHTML = '';
    TARGETS.forEach((target, index) => {
      const div = document.createElement('div');
      let statusClass = 'pending';
      let statusText = 'Pendente';

      if (target.hit) {
        statusClass = 'hit';
        statusText = 'FECHADO ✓';
      } else if (isRunning && index === currentTargetIndex) {
        statusClass = 'active';
        statusText = 'EM BUSCA...';
      }

      div.className = `target-item ${statusClass}`;
      div.innerHTML = `
        <div>
          <strong>${target.name}</strong><br>
          <small style="color: var(--subtext)">Alvo: $${target.price.toLocaleString('en-US', {minimumFractionDigits: 2})}</small>
        </div>
        <span style="font-weight: bold;">${statusText}</span>
      `;
      container.appendChild(div);
    });
  }

  function toggleStrategy() {
    const btn = document.getElementById('btn-action');
    const status = document.getElementById('position-status');

    if (!isRunning) {
      if (currentPrice === 0) {
        log('Aguardando conexão com o feed da Binance...', 'log-warn');
        return;
      }
      isRunning = true;
      btn.innerText = 'PAUSAR';
      btn.style.backgroundColor = '#f6465d';
      btn.style.color = '#fff';
      status.innerText = 'POSIÇÃO: VENDIDO (SHORT)';
      status.className = 'status-badge status-short';
      log(`START APLICADO! Ordem Vendida (Short) aberta ao preço de $${currentPrice.toFixed(2)}`, 'log-danger');
      renderTargets();
    } else {
      isRunning = false;
      btn.innerText = 'START';
      btn.style.backgroundColor = '#0ecb81';
      btn.style.color = '#000';
      status.innerText = 'OPERACAO PAUSADA';
      status.className = 'status-badge status-stopped';
      log('Operação pausada manualmente pelo utilizador.', 'log-warn');
      renderTargets();
    }
  }

  function initWebSocket() {
    const ws = new WebSocket('wss://stream.binance.com:9443/ws/btcusdt@ticker');

    ws.onopen = () => {
      log('WebSocket Binance conectado em tempo real.', 'log-success');
    };

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      currentPrice = parseFloat(data.c);

      document.getElementById('live-price').innerText = `$${currentPrice.toLocaleString('en-US', {minimumFractionDigits: 2})}`;

      if (isRunning && currentTargetIndex < TARGETS.length) {
        const target = TARGETS[currentTargetIndex];

        // Regra de toque / cruzamento do preço em tempo real
        const isHit = Math.abs(currentPrice - target.price) <= (target.price * 0.0005) ||
                      (currentPrice <= target.price && target.price <= 78000) ||
                      (currentPrice >= target.price && target.id === 'maxima');

        if (isHit) {
          target.hit = true;
          log(`ALVO EXECUTADO: ${target.name} em $${currentPrice.toFixed(2)}`, 'log-success');
          currentTargetIndex++;
          renderTargets();

          if (currentTargetIndex === TARGETS.length) {
            isRunning = false;
            log('TODOS OS ALVOS E SUPORTES FORAM ATINGIDOS COM SUCESSO!', 'log-success');
            document.getElementById('position-status').innerText = 'TODOS ALVOS CONCLUÍDOS';
            document.getElementById('position-status').className = 'status-badge status-done';
            document.getElementById('btn-action').disabled = true;
          }
        }
      }
    };

    ws.onerror = () => {
      log('Erro de conexão no WebSocket da Binance.', 'log-danger');
    };
  }

  window.onload = () => {
    renderTargets();
    initWebSocket();
  };
</script>

</body>
</html>
