/* =====================================================
   SCRIPT.JS - Versão totalmente refatorada e modularizada
   ===================================================== */

/*****************************************
 * 1. ESTADO GLOBAL E CONFIGURAÇÕES
 *****************************************/
const state = {
  units: [], // unidades carregadas dinamicamente
  config: {
    incc: 0.0045,
    ipca: 0.005,
    entradaPct: 30,
    entradaMonths: 6,
    payAtoPct: 0,
    payInterPct: 0,
    payInterMonth: 6,
    payChavesPct: 0,
    preLaunchMonths: 2,
    launchMonths: 3,
    discountPre: 10,
    discountLaunch: 5,
  },
  chart: null,
};

/*****************************************
 * 2. HELPERS
 *****************************************/
const $ = (sel) => document.querySelector(sel);
const formatCurrency = (v) => v.toLocaleString("pt-BR", { style: "currency", currency: "BRL" });

const parseNum = (v, fallback = 0) => {
  const n = Number(v);
  return isNaN(n) ? fallback : n;
};

/*****************************************
 * 3. CARREGAMENTO DE UNIDADES (exemplo)
 *****************************************/
function loadUnits() {
  // Em um sistema real viria do backend
  state.units = [
    { id: 1, descricao: "Apto 101", valor: 300000, categoria: "Apto", status: "Disponível" },
    { id: 2, descricao: "Apto 102", valor: 320000, categoria: "Apto", status: "Disponível" },
    { id: 3, descricao: "Cobertura 201", valor: 500000, categoria: "Cobertura", status: "Disponível" },
  ];
}

/*****************************************
 * 4. CÁLCULOS FINANCEIROS
 *****************************************/
function applyDiscount(unitValue, catIdx) {
  const { preLaunchMonths, launchMonths, discountPre, discountLaunch } = state.config;

  if (catIdx < preLaunchMonths) {
    return unitValue * (1 - discountPre / 100);
  }

  if (catIdx < preLaunchMonths + launchMonths) {
    return unitValue * (1 - discountLaunch / 100);
  }

  return unitValue; // valor cheio no pós-lançamento
}

function updateValueByINCC(initialValue, monthIdx) {
  const { incc } = state.config;
  return initialValue * Math.pow(1 + incc, monthIdx);
}

function updateValueByIPCA(initialValue, monthIdx) {
  const { ipca } = state.config;
  return initialValue * Math.pow(1 + ipca, monthIdx);
}

function calcCashflowForUnit(unit, catIdx) {
  const cfg = state.config;
  const cashFlow = new Array(24).fill(0); // fluxo de 24 meses

  // valor com desconto por fase
  const basePrice = applyDiscount(unit.valor, catIdx);

  const entradaTotal = (cfg.entradaPct / 100) * basePrice;
  const payAto = (cfg.payAtoPct / 100) * basePrice;
  const payInter = (cfg.payInterPct / 100) * basePrice;
  const payChaves = (cfg.payChavesPct / 100) * basePrice;

  const entradaMensal = entradaTotal / cfg.entradaMonths;

  // pagamento no ato
  cashFlow[0] += payAto;

  // entrada mensal
  for (let m = 1; m <= cfg.entradaMonths; m++) {
    cashFlow[m] += entradaMensal;
  }

  // pagamento intermediário
  if (cfg.payInterPct > 0) {
    cashFlow[cfg.payInterMonth] += updateValueByINCC(payInter, cfg.payInterMonth);
  }

  // pagamento na entrega (final)
  const monthChaves = 23;
  cashFlow[monthChaves] += updateValueByIPCA(payChaves, monthChaves);

  return cashFlow;
}

// Cálculo TIR aproximada (iterativo simples)
function calcTIR(cashflows) {
  let rate = 0.01;
  const maxIter = 200;
  const tolerance = 1e-6;

  for (let i = 0; i < maxIter; i++) {
    const npv = cashflows.reduce((acc, v, idx) => acc + v / Math.pow(1 + rate, idx), 0);
    if (Math.abs(npv) < tolerance) break;
    rate += npv > 0 ? 0.01 : -0.01;
    rate = Math.max(rate, -0.99);
  }
  return rate;
}

/*****************************************
 * 5. RENDERIZAÇÃO DA TABELA
 *****************************************/
function renderTable() {
  const table = $("#salesTable");
  table.innerHTML = "";

  const header = `
    <tr>
      <th>ID</th>
      <th>Unidade</th>
      <th>Categoria</th>
      <th>Status</th>
      <th>Valor Base</th>
      <th>Valor Fase</th>
    </tr>`;
  table.insertAdjacentHTML("beforeend", header);

  state.units.forEach((u, idx) => {
    const valorFase = applyDiscount(u.valor, idx);

    const row = `
      <tr>
        <td>${u.id}</td>
        <td>${u.descricao}</td>
        <td>${u.categoria}</td>
        <td>${u.status}</td>
        <td>${formatCurrency(u.valor)}</td>
        <td>${formatCurrency(valorFase)}</td>
      </tr>`;

    table.insertAdjacentHTML("beforeend", row);
  });
}

/*****************************************
 * 6. CÁLCULO GERAL E RESUMO
 *****************************************/
function runSimulation() {
  const allFlows = new Array(24).fill(0);

  state.units.forEach((u, idx) => {
    const flow = calcCashflowForUnit(u, idx);
    flow.forEach((v, m) => {
      allFlows[m] += v;
    });
  });

  const receitaTotal = allFlows.reduce((a, b) => a + b, 0);
  $("#receitaTotal").textContent = formatCurrency(receitaTotal);

  const tir = calcTIR(allFlows);
  $("#tir").textContent = (tir * 100).toFixed(2) + "%";

  $("#payback").textContent = calcPayback(allFlows) + " meses";
  $("#roi").textContent = calcROI(allFlows) + "%";

  renderChart(allFlows);
}

function calcPayback(flows) {
  let acc = 0;
  for (let i = 0; i < flows.length; i++) {
    acc += flows[i];
    if (acc >= 0) return i;
  }
  return flows.length;
}

function calcROI(flows) {
  const total = flows.reduce((a, b) => a + b, 0);
  const invested = -Math.min(...flows);
  if (invested <= 0) return 0;
  return ((total / invested) * 100).toFixed(2);
}

/*****************************************
 * 7. GRÁFICO
 *****************************************/
function renderChart(flows) {
  const ctx = $("#chartCanvas").getContext("2d");
  if (state.chart) state.chart.destroy();

  state.chart = new Chart(ctx, {
    type: "line",
    data: {
      labels: flows.map((_, i) => `M${i + 1}`),
      datasets: [
        {
          label: "Fluxo Mensal",
          data: flows,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
    },
  });
}

/*****************************************
 * 8. INTERAÇÕES E EVENTOS
 *****************************************/
function bindEvents() {
  $("#saveCfgBtn").addEventListener("click", () => {
    Object.assign(state.config, {
      incc: parseNum($("#incc").value),
      ipca: parseNum($("#ipca").value),
      entradaPct: parseNum($("#entradaPct").value),
      entradaMonths: parseNum($("#entradaMonths").value),
      payAtoPct: parseNum($("#payAtoPct").value),
      payInterPct: parseNum($("#payInterPct").value),
      payInterMonth: parseNum($("#payInterMonth").value),
      payChavesPct: parseNum($("#payChavesPct").value),
      preLaunchMonths: parseNum($("#preLaunchMonths").value),
      launchMonths: parseNum($("#launchMonths").value),
      discountPre: parseNum($("#discountPre").value),
      discountLaunch: parseNum($("#discountLaunch").value),
    });

    renderTable();
    runSimulation();
  });
}

/*****************************************
 * 9. INICIALIZAÇÃO
 *****************************************/
function init() {
  loadUnits();
  bindEvents();
  renderTable();
  runSimulation();
}

init();
