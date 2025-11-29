<template>
  <div class="sim-root">
    <h1 class="title">Your Retirement Account Over Time</h1>
    <p class="subtitle">
      Use this simulator to see how different risk levels, spending choices and market paths can play out over a full retirement.
    </p>

    <!-- Primary inputs row -->
    <div class="row inputs">
      <div class="input-group">
        <label>Initial Lump Sum</label>
        <input
          type="number"
          v-model.number="form.initialInvestment"
          min="0"
        />
      </div>

      <div class="input-group">
        <label>Monthly Saving</label>
        <input
          type="number"
          v-model.number="form.monthlySaving"
          min="0"
        />
      </div>

      <div class="input-group">
        <label>Retirement Year</label>
        <input
          type="number"
          v-model.number="form.retirementYear"
          :min="startYear"
          :max="2500"
        />
      </div>

      <div class="input-group">
        <label>Initial Risk Allocation</label>
        <select v-model="form.initialRiskProfile">
          <option v-for="opt in riskOptions" :key="opt" :value="opt">
            {{ opt }}
          </option>
        </select>
      </div>

      <div class="input-group">
        <label>Desired Monthly Withdrawal (in today's dollars)</label>
        <input
          type="number"
          v-model.number="form.desiredMonthlyWithdrawal"
          min="0"
        />
      </div>

      <div class="input-group buttons">
        <button @click="startNewSimulation" class="primary">
          Start / Restart
        </button>
      </div>
    </div>

    <!-- Chart -->
    <div class="chart-wrapper">
      <canvas ref="chartCanvas"></canvas>
    </div>

    <!-- Saved runs chips (visual key, not legend) -->
    <div class="saved-runs" v-if="historicalRuns.length">
      <span class="saved-label">Saved runs:</span>
      <span
        v-for="(run, index) in historicalRuns"
        :key="index"
        class="saved-chip"
        :class="{ 'saved-chip-active': activeSavedRunIndex === index }"
        @click="setActiveSavedRun(index)"
      >
        {{ run.risk }} {{ index + 1 }}
      </span>
    </div>

    <!-- High level insights -->
    <div class="insights-panel" v-if="insightHeadline">
      <div class="insight-headline">{{ insightHeadline }}</div>
      <div class="insight-detail">{{ insightDetail }}</div>
      <div class="insight-metrics">
        <span>Worst 5 year market drawdown so far: {{ worstDrawdownText }}</span>
      </div>
    </div>

    <p class="helper-text">
      Run a scenario, then save it and compare it with more conservative or more aggressive choices.
    </p>

    <!-- Speed + risk controls -->
    <div class="row controls">
      <div class="control-group">
        <span class="group-label">Speed</span>
        <button
          @click="setSpeed('Pause')"
          :class="{ active: speedLabel === 'Pause' }"
        >
          Pause
        </button>
        <button
          @click="setSpeed('Normal')"
          :class="{ active: speedLabel === 'Normal' }"
        >
          Play
        </button>
        <button
          @click="setSpeed('Fast')"
          :class="{ active: speedLabel === 'Fast' }"
        >
          Fast
        </button>
        <button
          @click="setSpeed('Faster')"
          :class="{ active: speedLabel === 'Faster' }"
        >
          Very Fast
        </button>
        <button
          @click="setSpeed('Max')"
          :class="{ active: speedLabel === 'Max' }"
        >
          Max
        </button>
        <span class="speed-label">
          {{ friendlySpeedLabel }}
        </span>
      </div>

      <!-- Risk controls -->
      <div class="control-group">
        <span class="group-label">Risk Profile</span>
        <button
          v-for="opt in riskOptions"
          :key="opt"
          @click="changeRisk(opt)"
          :class="{ active: currentRisk === opt }"
        >
          <span
            class="risk-dot"
            :style="{ backgroundColor: riskColor(opt) }"
          ></span>
          {{ opt }}
        </button>
      </div>
    </div>

    <!-- Feeling + withdrawal + runs -->
    <div class="row controls">
      <div class="control-group">
        <span class="group-label">How this feels</span>
        <button @click="logEmotion('Discomfort')">Scary</button>
        <button @click="logEmotion('Excitement')">Exciting</button>
        <span class="microcopy">
          These do not change the numbers - they simply mark where your gut reacts so you can talk about it.
        </span>
      </div>

      <div class="control-group">
        <span class="group-label">Withdrawals</span>
        <button @click="adjustWithdrawal('down')">Reduce spending</button>
        <button @click="adjustWithdrawal('up')">Increase spending</button>
      </div>

      <div class="control-group runs-group">
        <span class="group-label">Runs</span>
        <button @click="resetSimulation">Save run &amp; restart</button>
        <button @click="clearHistory">Clear history</button>
      </div>
    </div>

    <!-- Status bar (simplified) -->
    <div class="status-bar">
      <span>Current risk: {{ currentRisk }}</span>
      <span>Phase: {{ phaseText }}</span>
      <span>Year: {{ yearText }}</span>
      <span>Monthly Withdrawal (In Today's Dollars): {{ withdrawalText }}</span>
      <span class="status-highlight">
        Withdrawal Rate: {{ withdrawalRateText }}
      </span>
      <span>Track To Zero Rate: {{ trackToZeroText }}</span>
    </div>

    <!-- Disclaimer -->
    <div class="disclaimer">
      <p>
        This simulator has been developed by Page Family Financial Planning Limited. It is for general information and education only and is not financial advice. It does not take into account your personal objectives, financial situation or needs.
      </p>
      <p>
        All investment paths shown are based on hypothetical, randomised return assumptions. They are not forecasts, guarantees or representations of actual products. Past performance or simulated performance is not a reliable indicator of future results. Before making any investment or retirement decision, consider seeking personalised advice from a licensed financial adviser.
      </p>
    </div>
  </div>
</template>

<script setup>
import {
  ref,
  reactive,
  onMounted,
  onBeforeUnmount,
  computed,
} from "vue";
import {
  Chart,
  LineController,
  LineElement,
  PointElement,
  LinearScale,
  CategoryScale,
  Tooltip,
  Legend,
  ScatterController,
} from "chart.js";

function roundRect(ctx, x, y, w, h, r) {
  const radius = typeof r === "number" ? { tl: r, tr: r, br: r, bl: r } : r;
  ctx.beginPath();
  ctx.moveTo(x + radius.tl, y);
  ctx.lineTo(x + w - radius.tr, y);
  ctx.quadraticCurveTo(x + w, y, x + w, y + radius.tr);
  ctx.lineTo(x + w, y + h - radius.br);
  ctx.quadraticCurveTo(x + w, y + h, x + w - radius.br, y + h);
  ctx.lineTo(x + radius.bl, y + h);
  ctx.quadraticCurveTo(x, y + h, x, y + h - radius.bl);
  ctx.lineTo(x, y + radius.tl);
  ctx.quadraticCurveTo(x, y, x + radius.tl, y);
  ctx.closePath();
}

// Plugin to draw labels on the last nominal and real points
const lastPointLabelsPlugin = {
  id: "lastPointLabels",
  afterDatasetsDraw(chartInstance) {
    const { ctx } = chartInstance;
    const datasets = chartInstance.data.datasets;
    if (!datasets || datasets.length < 2) return;

    ctx.save();
    ctx.font = "12px Georgia, 'Times New Roman', serif";
    ctx.textBaseline = "middle";

    [0, 1].forEach((dsIndex) => {
      const dataset = datasets[dsIndex];
      if (!dataset || !dataset.data || dataset.data.length === 0) return;

      const meta = chartInstance.getDatasetMeta(dsIndex);
      const lastIndex = dataset.data.length - 1;
      const element = meta.data[lastIndex];
      if (!element) return;

      const x = element.x;
      const y = element.y;
      const value = dataset.data[lastIndex].y;
      const text =
        "$" +
        value.toLocaleString(undefined, {
          minimumFractionDigits: 2,
          maximumFractionDigits: 2,
        });

      const paddingX = 6;
      const boxWidth = ctx.measureText(text).width + paddingX * 2;
      const boxHeight = 20;

      const offsetY = dsIndex === 0 ? -30 : -5;
      const boxX = x + 8;
      const boxY = y + offsetY;

      ctx.fillStyle = "#ffffff";
      ctx.strokeStyle = "#000000";
      ctx.lineWidth = 1;

      roundRect(ctx, boxX, boxY, boxWidth, boxHeight, 4);
      ctx.fill();
      ctx.stroke();

      ctx.fillStyle = "#000000";
      ctx.fillText(text, boxX + paddingX, boxY + boxHeight / 2);
    });

    ctx.restore();
  },
};

// Plugin to draw spending change labels like "4%"
const spendingLabelsPlugin = {
  id: "spendingLabels",
  afterDatasetsDraw(chartInstance) {
    const { ctx } = chartInstance;
    const datasets = chartInstance.data.datasets;
    if (!datasets) return;

    const dsIndex = datasets.findIndex(
      (d) => d && d.label === "Spending Change"
    );
    if (dsIndex === -1) return;

    const dataset = datasets[dsIndex];
    if (!dataset.data || dataset.data.length === 0) return;

    const meta = chartInstance.getDatasetMeta(dsIndex);
    ctx.save();
    ctx.font = "11px Georgia, 'Times New Roman', serif";
    ctx.textBaseline = "middle";

    dataset.data.forEach((point, idx) => {
      const element = meta.data[idx];
      if (!element) return;
      const label = point.label;
      if (!label) return;

      const x = element.x;
      const y = element.y;

      const paddingX = 4;
      const paddingY = 2;
      const textWidth = ctx.measureText(label).width;
      const boxWidth = textWidth + paddingX * 2;
      const boxHeight = 16;

      // stagger labels a bit to avoid collisions
      const direction = idx % 2 === 0 ? -1 : 1;
      const boxX = x + 6;
      const boxY = y + direction * 18;

      ctx.fillStyle = "#ffffff";
      ctx.strokeStyle = "#4b5563";
      ctx.lineWidth = 1;
      roundRect(ctx, boxX, boxY, boxWidth, boxHeight, 4);
      ctx.fill();
      ctx.stroke();

      ctx.fillStyle = "#111827";
      ctx.fillText(label, boxX + paddingX, boxY + boxHeight / 2);
    });

    ctx.restore();
  },
};

// Plugin to draw a palm tree at the retirement point
const retirementPalmPlugin = {
  id: "retirementPalm",
  afterDatasetsDraw(chartInstance) {
    const { ctx } = chartInstance;
    const datasets = chartInstance.data.datasets;
    if (!datasets) return;

    const dsIndex = datasets.findIndex(
      (d) => d && d.label === "Retirement"
    );
    if (dsIndex === -1) return;

    const dataset = datasets[dsIndex];
    if (!dataset.data || dataset.data.length === 0) return;

    const meta = chartInstance.getDatasetMeta(dsIndex);
    ctx.save();
    ctx.font = "16px Georgia, 'Times New Roman', serif";
    ctx.textBaseline = "bottom";

    dataset.data.forEach((point, idx) => {
      const element = meta.data[idx];
      if (!element) return;
      const x = element.x;
      const y = element.y;
      ctx.fillText("🌴", x - 8, y - 6);
    });

    ctx.restore();
  },
};

Chart.register(
  LineController,
  LineElement,
  PointElement,
  LinearScale,
  CategoryScale,
  Tooltip,
  Legend,
  ScatterController,
  lastPointLabelsPlugin,
  spendingLabelsPlugin,
  retirementPalmPlugin
);

// ---- Constants ----

const MONTH_NAMES = [
  "Jan",
  "Feb",
  "Mar",
  "Apr",
  "May",
  "Jun",
  "Jul",
  "Aug",
  "Sep",
  "Oct",
  "Nov",
  "Dec",
];

const RISK_SETTINGS = {
  Cash: {
    mean: 0.01,
    std: 0.006,
  },
  Conservative: {
    mean: 0.029,
    std: 0.0386,
  },
  Moderate: {
    mean: 0.035,
    std: 0.0499,
  },
  Balanced: {
    mean: 0.047,
    std: 0.0741,
  },
  Growth: {
    mean: 0.0578,
    std: 0.099,
  },
  "High Growth": {
    mean: 0.0667,
    std: 0.1237,
  },
};

const PAST_RUN_COLORS = {
  Cash: "#9ca3af",
  Conservative: "#6b7280",
  Moderate: "#22c55e",
  Balanced: "#eab308",
  Growth: "#0ea5e9",
  "High Growth": "#ef4444",
  Unknown: "#e6c300",
};

const riskOptions = Object.keys(RISK_SETTINGS);
const startYear = 2026;
const inflationRate = 0.02;

// RNG
function randn() {
  let u = 0;
  let v = 0;
  while (u === 0) u = Math.random();
  while (v === 0) v = Math.random();
  return Math.sqrt(-2.0 * Math.log(u)) * Math.cos(2.0 * Math.PI * v);
}

function formatCurrency(val) {
  return (
    "$" +
    val.toLocaleString(undefined, {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2,
    })
  );
}

// ---- Simulation engine ----

class InvestmentPortfolioSimulator {
  constructor(
    initialInvestment,
    monthlySaving,
    initialRiskProfile,
    startYear,
    retirementYear,
    monthlyWithdrawal
  ) {
    this.year = startYear;
    this.startYear = startYear;
    this.retirementYear = retirementYear;
    this.deathYear = retirementYear + 25;
    this.monthIndex = 0;

    this.initialInvestment = initialInvestment;
    this.initialMonthlySaving = monthlySaving;
    this.initialRiskProfile = initialRiskProfile;
    this.initialMonthlyWithdrawal = monthlyWithdrawal;

    this.units = initialInvestment / 1.0;
    this.unitValue = 1.0;
    this.cash = 0.0;

    this.monthlySaving = monthlySaving;
    this.baseWithdrawal = monthlyWithdrawal;
    this.inflationRate = inflationRate;

    this.riskProfile = null;
    this.meanReturn = null;
    this.stdDev = null;

    this.withdrawals = 0.0;
    this.history = [];
    this.emotions = [];
    this.riskChanges = [];
    this.spendingChanges = [];
    this.finished = false;

    this.setRiskProfile(initialRiskProfile);
  }

  setRiskProfile(profile) {
    let key = (profile || "").trim();
    if (!RISK_SETTINGS[key]) {
      key = "Balanced";
      console.log("Unknown risk profile, defaulting to Balanced.");
    }

    // Log risk change only once we are underway
    if (
      this.riskProfile &&
      this.riskProfile !== key &&
      this.history.length > 0
    ) {
      this.riskChanges.push({
        MonthIndex: this.monthIndex,
        Year: this.year,
        From: this.riskProfile,
        To: key,
      });
    }

    const settings = RISK_SETTINGS[key];
    this.riskProfile = key;
    this.meanReturn = settings.mean;
    this.stdDev = settings.std;
    console.log(
      `Risk profile set to ${this.riskProfile} (mean ${(this.meanReturn * 100).toFixed(
        2
      )}%, std ${(this.stdDev * 100).toFixed(2)}%).`
    );
  }

  logEmotion(emotion) {
    this.emotions.push({
      Year: this.year,
      MonthIndex: this.monthIndex,
      Emotion: emotion,
    });
    console.log(
      `Emotion logged: ${emotion} at Year ${this.year.toFixed(
        2
      )}, Month ${this.monthIndex}.`
    );
  }

  recordSpendingChange(newRateFraction) {
    if (!this.history.length) return;
    const monthIndex = this.monthIndex;

    const last = this.spendingChanges[this.spendingChanges.length - 1];

    // If the last change was in the last 6 months, overwrite it instead of adding a new label
    if (last && monthIndex - last.MonthIndex < 6) {
      last.MonthIndex = monthIndex;
      last.Year = this.year;
      last.Rate = newRateFraction;
    } else {
      this.spendingChanges.push({
        MonthIndex: monthIndex,
        Year: this.year,
        Rate: newRateFraction,
      });
    }
  }

  simulateMonth() {
    if (this.finished) return true;
    if (this.year >= this.deathYear) {
      this.finished = true;
      console.log("Reached death date. Simulation finished.");
      return true;
    }

    const phase = this.year < this.retirementYear ? "Accumulation" : "Retirement";

    const startPortfolioBalance = this.units * this.unitValue + this.cash;
    const yearStartUnitValue = this.unitValue;

    let contributions = 0.0;
    let withdrawalsThisMonth = 0.0;
    this.withdrawals = 0.0;

    if (phase === "Accumulation") {
      if (this.monthlySaving > 0) {
        const unitsToBuy = this.monthlySaving / this.unitValue;
        this.units += unitsToBuy;
        contributions = this.monthlySaving;
      }
    } else {
      if (this.baseWithdrawal > 0) {
        const yearsSinceStart = Math.max(0, this.year - this.startYear);
        const inflationFactor = Math.pow(1.0 + this.inflationRate, yearsSinceStart);
        const desiredWithdrawal = this.baseWithdrawal * inflationFactor;

        const portfolioBefore = this.units * this.unitValue + this.cash;
        withdrawalsThisMonth = Math.min(desiredWithdrawal, portfolioBefore);

        if (withdrawalsThisMonth > 0) {
          if (this.cash < withdrawalsThisMonth) {
            const amountToRaise = withdrawalsThisMonth - this.cash;
            let unitsToSell = amountToRaise / this.unitValue;
            unitsToSell = Math.min(unitsToSell, this.units);
            this.units -= unitsToSell;
            this.cash += amountToRaise;
          }
          this.cash -= withdrawalsThisMonth;
          this.withdrawals += withdrawalsThisMonth;
        }
      }
    }

    const monthlyReturn =
      this.meanReturn / 12.0 + (this.stdDev / Math.sqrt(12.0)) * randn();
    this.unitValue *= 1.0 + monthlyReturn;
    const yearEndUnitValue = this.unitValue;

    const totalInvestmentValue = this.units * this.unitValue;
    const totalPortfolioValue = totalInvestmentValue + this.cash;

    const yearsSinceStart = Math.max(0, this.year - this.startYear);
    const realPortfolioValue =
      totalPortfolioValue / Math.pow(1.0 + this.inflationRate, yearsSinceStart);

    const growthAmount =
      totalPortfolioValue - startPortfolioBalance - contributions + withdrawalsThisMonth;

    this.history.push({
      MonthIndex: this.monthIndex,
      Year: parseFloat(this.year.toFixed(4)),
      Phase: phase,
      RiskProfile: this.riskProfile,
      "Year Start Unit Value": parseFloat(yearStartUnitValue.toFixed(4)),
      "Total Units Held": parseFloat(this.units.toFixed(4)),
      "Year End Unit Value": parseFloat(yearEndUnitValue.toFixed(4)),
      Contributions: parseFloat(contributions.toFixed(2)),
      Withdrawals: parseFloat(withdrawalsThisMonth.toFixed(2)),
      "Year Start Portfolio Balance": parseFloat(startPortfolioBalance.toFixed(2)),
      "Year End Portfolio Balance": parseFloat(totalPortfolioValue.toFixed(2)),
      "Real End Portfolio Balance": parseFloat(realPortfolioValue.toFixed(2)),
      "Growth Amount": parseFloat(growthAmount.toFixed(2)),
    });

    this.year += 1.0 / 12.0;
    this.monthIndex += 1;

    if (this.year >= this.deathYear) {
      this.finished = true;
      console.log("Reached death date. Simulation finished.");
      return true;
    }

    return false;
  }
}

// ---- Vue state ----

const chartCanvas = ref(null);
let chart = null;
let timerId = null;

// x axis mode for labelling
let axisMode = "yearly"; // "monthly", "yearly", "fiveyear"

const form = reactive({
  initialInvestment: 400000,
  monthlySaving: 0,
  retirementYear: 2026,
  initialRiskProfile: "Cash",
  desiredMonthlyWithdrawal: null,
});

let simulator = null;

// past runs: { history: [...], risk: "Balanced" }
const historicalRuns = ref([]);
const activeSavedRunIndex = ref(null);

// UI/status refs
const currentRisk = ref("Cash");
const phaseText = ref("Accumulation");
const yearText = ref(startYear.toFixed(2));
const withdrawalText = ref("$0.00");
const withdrawalRateText = ref("0.00%");
const trackToZeroText = ref("0.00%");

// Insights
const insightHeadline = ref("");
const insightDetail = ref("");
const worstDrawdownText = ref("Not yet calculated");

// Speed controls
const speedLabel = ref("Pause");
let speedSettingMs = 1000;

const friendlySpeedLabel = computed(() => {
  if (speedLabel.value === "Pause") return "Paused";
  if (speedLabel.value === "Normal") return "Running at normal speed";
  if (speedLabel.value === "Fast") return "Running fast";
  if (speedLabel.value === "Faster") return "Running very fast";
  if (speedLabel.value === "Max") return "Running at max speed";
  return speedLabel.value;
});

function riskColor(risk) {
  return PAST_RUN_COLORS[risk] || PAST_RUN_COLORS.Unknown;
}

// Helper to format x axis ticks based on current axisMode
function formatXAxisTick(monthIndexRaw) {
  const m = Number(monthIndexRaw);
  if (Number.isNaN(m)) return "";
  const yearOffset = Math.floor(m / 12);
  const monthIndex = m % 12;
  const yearValue = startYear + yearOffset;

  if (axisMode === "monthly") {
    const monthName = MONTH_NAMES[monthIndex];
    const shortYear = String(yearValue).slice(-2);
    return `${monthName} ${shortYear}`;
  }

  // yearly or fiveyear - just show year
  return String(yearValue);
}

// ---- Chart helpers ----

function initChart() {
  if (!chartCanvas.value) return;
  const ctx = chartCanvas.value.getContext("2d");

  chart = new Chart(ctx, {
    type: "line",
    data: {
      labels: [],
      datasets: [
        {
          label: "Nominal balance",
          data: [],
          borderColor: "#003366",
          backgroundColor: "rgba(0,51,102,0.08)",
          borderWidth: 2,
          pointRadius: 0,
          tension: 0.1,
        },
        {
          label: "Inflation-adjusted balance",
          data: [],
          borderColor: "#7f1d1d",
          backgroundColor: "rgba(127,29,29,0.06)",
          borderWidth: 2,
          borderDash: [4, 4],
          pointRadius: 0,
          tension: 0.1,
        },
        {
          label: "Discomfort",
          data: [],
          type: "scatter",
          borderColor: "#b91c1c",
          backgroundColor: "#b91c1c",
          pointRadius: 6,
          pointStyle: "triangle",
          showLine: false,
        },
        {
          label: "Excitement",
          data: [],
          type: "scatter",
          borderColor: "#15803d",
          backgroundColor: "#15803d",
          pointRadius: 6,
          pointStyle: "triangle",
          showLine: false,
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      animation: false,
      scales: {
        x: {
          type: "linear",
          ticks: {
            stepSize: 12,
            callback(value) {
              return formatXAxisTick(value);
            },
          },
        },
        y: {
          ticks: {
            callback(value) {
              return (
                "$" +
                value.toLocaleString(undefined, {
                  maximumFractionDigits: 0,
                })
              );
            },
          },
        },
      },
      layout: {
        padding: { top: 10, right: 80, bottom: 28, left: 0 },
      },
      plugins: {
        legend: {
          position: "top",
          labels: {
            filter(legendItem, data) {
              const dataset = data.datasets[legendItem.datasetIndex];

              if (dataset && dataset.skipLegend) return false;
              if (!dataset || !dataset.data || dataset.data.length === 0) {
                return false;
              }

              const label = legendItem.text;
              const idx = legendItem.datasetIndex;
              for (let i = 0; i < idx; i++) {
                const earlier = data.datasets[i];
                if (
                  earlier &&
                  !earlier.skipLegend &&
                  earlier.label === label &&
                  earlier.data &&
                  earlier.data.length > 0
                ) {
                  return false;
                }
              }
              return true;
            },
          },
        },
        tooltip: {
          callbacks: {
            label(context) {
              const val = context.parsed.y;
              return `${context.dataset.label}: $${val.toLocaleString(
                undefined,
                {
                  maximumFractionDigits: 0,
                }
              )}`;
            },
          },
        },
      },
    },
  });
}

function updateChartFromSimulator() {
  if (!chart || !simulator) return;
  const hist = simulator.history;
  if (!hist.length) {
    chart.data.labels = [];
    chart.data.datasets[0].data = [];
    chart.data.datasets[1].data = [];
    chart.update();
    return;
  }

  const months = hist.map((row) => row.MonthIndex);
  const nominalValues = hist.map((row) => row["Year End Portfolio Balance"]);
  const realValues = hist.map((row) => row["Real End Portfolio Balance"]);

  const maxMonth = Math.max(...months);
  const maxVal = Math.max(...nominalValues, ...realValues, 1);

  // Decide axis mode and tick spacing based on time horizon
  if (maxMonth <= 36) {
    axisMode = "monthly";
    chart.options.scales.x.ticks.stepSize = maxMonth <= 18 ? 1 : 3;
  } else if (maxMonth <= 180) {
    axisMode = "yearly";
    chart.options.scales.x.ticks.stepSize = 12;
  } else {
    axisMode = "fiveyear";
    chart.options.scales.x.ticks.stepSize = 60;
  }

  chart.options.scales.x.min = 0;
  chart.options.scales.x.max = maxMonth + 1;
  chart.options.scales.y.min = 0;
  chart.options.scales.y.max = maxVal * 1.2;

  chart.data.labels = months;

  chart.data.datasets[0].data = months.map((m, idx) => ({
    x: m,
    y: nominalValues[idx],
  }));
  chart.data.datasets[1].data = months.map((m, idx) => ({
    x: m,
    y: realValues[idx],
  }));

  // Reset to base datasets (nominal, real, emotions)
  chart.data.datasets = chart.data.datasets.slice(0, 4);

  // Quick lookup for nominal values by month
  const nominalByMonth = {};
  months.forEach((m, idx) => {
    nominalByMonth[m] = nominalValues[idx];
  });

  // Historical runs
  historicalRuns.value.forEach((runObj, runIdx) => {
    const runHist = runObj.history || [];
    if (!runHist.length) return;
    const risk = runObj.risk || "Unknown";
    const color = PAST_RUN_COLORS[risk] || PAST_RUN_COLORS.Unknown;

    const mArr = runHist.map((row) => row.MonthIndex);
    const nomArr = runHist.map((row) => row["Year End Portfolio Balance"]);
    if (!mArr.length) return;

    const points = [];
    for (let i = 0; i < mArr.length; i++) {
      const m = mArr[i];
      if (m % 60 === 0 || i === mArr.length - 1) {
        points.push({ x: m, y: nomArr[i] });
      }
    }

    chart.data.datasets.push({
      label: `${risk} run ${runIdx + 1}`,
      data: points,
      borderColor: color,
      borderWidth: activeSavedRunIndex.value === runIdx ? 2 : 1,
      borderDash: [6, 4],
      pointRadius: 0,
      fill: false,
      tension: 0,
      skipLegend: true,
    });
  });

  // Risk change markers
  if (simulator.riskChanges && simulator.riskChanges.length) {
    const riskPoints = [];
    const riskColors = [];

    simulator.riskChanges.forEach((rc) => {
      const y = nominalByMonth[rc.MonthIndex];
      if (y == null) return;
      riskPoints.push({ x: rc.MonthIndex, y });
      riskColors.push(riskColor(rc.To));
    });

    if (riskPoints.length) {
      chart.data.datasets.push({
        label: "Risk change",
        data: riskPoints,
        type: "scatter",
        pointRadius: 5,
        pointStyle: "circle",
        pointBackgroundColor: riskColors,
        pointBorderColor: riskColors,
        borderWidth: 0,
        showLine: false,
        skipLegend: true,
      });
    }
  }

  // Spending change markers (labels handled by plugin)
  if (simulator.spendingChanges && simulator.spendingChanges.length) {
    const spendPoints = [];

    simulator.spendingChanges.forEach((sc, idx) => {
      const y = nominalByMonth[sc.MonthIndex];
      if (y == null) return;
      const pct = (sc.Rate * 100).toFixed(1) + "%";
      spendPoints.push({
        x: sc.MonthIndex,
        y,
        label: pct,
      });
    });

    if (spendPoints.length) {
      chart.data.datasets.push({
        label: "Spending Change",
        data: spendPoints,
        type: "scatter",
        pointRadius: 0,
        showLine: false,
        skipLegend: true,
      });
    }
  }

  // Retirement marker - palm tree
  const retirementPoints = [];
  if (simulator.year >= simulator.retirementYear) {
    const retirementMonthIndex = Math.round(
      (simulator.retirementYear - simulator.startYear) * 12
    );
    const y = nominalByMonth[retirementMonthIndex];
    if (y != null) {
      retirementPoints.push({
        x: retirementMonthIndex,
        y,
      });
    }
  }

  if (retirementPoints.length) {
    chart.data.datasets.push({
      label: "Retirement",
      data: retirementPoints,
      type: "scatter",
      pointRadius: 0,
      showLine: false,
      skipLegend: true,
    });
  }

  // Emotion markers
  const discomfortPoints = [];
  const excitementPoints = [];

  simulator.emotions.forEach((e) => {
    const mIdx = e.MonthIndex;
    const baseY = nominalByMonth[mIdx];
    if (baseY == null) return;
    const yIcon = baseY > 0 ? baseY * 1.03 : 0.03;
    const point = { x: mIdx, y: yIcon };
    if (e.Emotion === "Discomfort") {
      discomfortPoints.push(point);
    } else {
      excitementPoints.push(point);
    }
  });

  chart.data.datasets[2].data = discomfortPoints;
  chart.data.datasets[3].data = excitementPoints;

  chart.update();
}

// 5 year market drawdown logic (nominal, withdrawals added back)
function computeWorstDrawdown(history) {
  if (!history || history.length === 0) {
    worstDrawdownText.value = "Not yet calculated";
    return;
  }

  const n = history.length;
  const wealthAdj = [];
  let cumWithdrawals = 0;

  for (let i = 0; i < n; i++) {
    const nominal = history[i]["Year End Portfolio Balance"];
    const w = history[i].Withdrawals || 0;

    cumWithdrawals += w;
    wealthAdj.push(nominal + cumWithdrawals);
  }

  const windowMonths = 60; // 5 years
  let worstDD = 0;

  for (let i = 0; i < n; i++) {
    const endIdx = i;
    const startIdx = Math.max(0, endIdx - windowMonths + 1);
    let peak = wealthAdj[startIdx];

    for (let j = startIdx + 1; j <= endIdx; j++) {
      if (wealthAdj[j] > peak) {
        peak = wealthAdj[j];
      }
    }

    if (peak > 0) {
      const dd = (peak - wealthAdj[endIdx]) / peak;
      if (dd > worstDD) {
        worstDD = dd;
      }
    }
  }

  if (worstDD > 0) {
    worstDrawdownText.value = `-${(worstDD * 100).toFixed(
      1
    )}% from a prior 5 year peak (markets only, withdrawals added back)`;
  } else {
    worstDrawdownText.value = "No meaningful drawdown in this period";
  }
}

function appendNzSuperLine(baseText) {
  return (
    baseText +
    " This is on top of NZ Super. As a couple NZ Super should give you around $43,000 per year, and around $28,000 per year as an individual."
  );
}

function updateInsights(lastRow, real, withdrawalRate) {
  if (!simulator) return;
  const phase = lastRow.Phase;
  const annualSpend = simulator.baseWithdrawal * 12;
  const monthlySpend = simulator.baseWithdrawal;

  if (phase === "Retirement") {
    const targetYear = Math.round(simulator.deathYear);
    const wrPct = withdrawalRate * 100;

    const remainingYearsRaw = simulator.deathYear - simulator.year;

    // Work out if the portfolio hit zero at some point
    let depletionYear = null;
    for (let i = 0; i < simulator.history.length; i++) {
      if (
        simulator.history[i]["Year End Portfolio Balance"] <= 0 &&
        depletionYear === null
      ) {
        depletionYear = Math.round(simulator.history[i].Year);
        break;
      }
    }

    // End-of-run summary once we are effectively at the last year
    if (remainingYearsRaw <= 0.75) {
      const finalReal = real;
      const startReal = simulator.initialInvestment;

      if (depletionYear && depletionYear < targetYear - 1) {
        insightHeadline.value =
          "On this path the portfolio runs out before the end of retirement.";
        insightDetail.value = appendNzSuperLine(
          `In this simulation the portfolio is effectively spent by about ${depletionYear}, while the plan runs through to around ${targetYear}. The spending level of roughly ${formatCurrency(
            annualSpend
          )} per year (about ${formatCurrency(
            monthlySpend
          )} per month) has been too heavy for this particular mix of returns. To reduce the chance of this happening in real life, you could explore lowering spending, taking less investment risk, or building in a rule to cut spending for a while after big market falls.`
        );
      } else if (finalReal > startReal * 0.3) {
        insightHeadline.value =
          "You finished this retirement with money left over.";
        insightDetail.value = appendNzSuperLine(
          `On this run the portfolio ends around ${formatCurrency(
            finalReal
          )} in today's dollars by about ${targetYear}. Over retirement you have been drawing roughly ${formatCurrency(
            annualSpend
          )} per year (about ${formatCurrency(
            monthlySpend
          )} per month), and the account has stayed ahead of you most of the way. This is what a 'spend comfortably and still leave a buffer or inheritance' path can look like.`
        );
      } else if (finalReal > startReal * 0.05) {
        insightHeadline.value =
          "You just about used the portfolio up by the end.";
        insightDetail.value = appendNzSuperLine(
          `On this run the portfolio ends around ${formatCurrency(
            finalReal
          )} in today's dollars by about ${targetYear} - not quite empty, but not with a big surplus either. Your spending of around ${formatCurrency(
            annualSpend
          )} per year has roughly matched what this particular market path could support. With slightly weaker markets or slightly higher spending, this path could easily have run out of money earlier.`
        );
      } else {
        insightHeadline.value =
          "This path almost fully used the portfolio by the end.";
        insightDetail.value = appendNzSuperLine(
          `By around ${targetYear} the portfolio is down to a small amount - roughly ${formatCurrency(
            finalReal
          )} in today's dollars - after funding withdrawals of about ${formatCurrency(
            annualSpend
          )} per year. It has just made it through the plan, but with very little buffer for late surprises. In reality you might want to start trimming spending earlier if markets were weak.`
        );
      }

      return; // no forward looking text at the very end
    }

    // Mid-retirement forward-looking commentary (with years remaining)
    const remainingYearsSafe = Math.max(remainingYearsRaw, 0.01);
    const approxYears = Math.round(remainingYearsSafe);
    const evenAnnual = remainingYearsSafe > 0 ? real / remainingYearsSafe : 0;
    const evenRatePct = real > 0 ? (evenAnnual / real) * 100 : 0;

    if (wrPct < evenRatePct * 0.7) {
      insightHeadline.value = "Your spending looks very conservative for this portfolio.";
      insightDetail.value = appendNzSuperLine(
        `You are currently withdrawing about ${formatCurrency(
          annualSpend
        )} per year (about ${formatCurrency(
          monthlySpend
        )} per month) in today's dollars. On this simulated path the portfolio looks able to comfortably support this spending for about ${approxYears} more years in retirement, through to around ${targetYear}. If you simply divided today's balance evenly over those ${approxYears} years with no market growth, it would work out to roughly ${formatCurrency(
          evenAnnual
        )} per year, so you have a solid buffer.`
      );
    } else if (wrPct <= evenRatePct * 1.4) {
      insightHeadline.value = "Your spending sits in a broadly sustainable range.";
      insightDetail.value = appendNzSuperLine(
        `You are withdrawing about ${formatCurrency(
          annualSpend
        )} per year (about ${formatCurrency(
          monthlySpend
        )} per month) in today's dollars. Based on this simulated path, that looks manageable for a portfolio expected to last about ${approxYears} more years in retirement, through to around ${targetYear}, as long as you are willing to trim spending if markets are weak for an extended period. Dividing today's balance evenly over those ${approxYears} years would give about ${formatCurrency(
          evenAnnual
        )} per year, which is in the same ballpark as your current spending.`
      );
    } else {
      insightHeadline.value =
        "Your current spending is on the ambitious side for this portfolio.";
      insightDetail.value = appendNzSuperLine(
        `You are withdrawing about ${formatCurrency(
          annualSpend
        )} per year (about ${formatCurrency(
          monthlySpend
        )} per month) in today's dollars. That is high for a portfolio expected to support you for about ${approxYears} more years in retirement, through to roughly ${targetYear}. If you divided today's balance evenly over those ${approxYears} years with no market growth, it would be around ${formatCurrency(
          evenAnnual
        )} per year, which is below your current spending. To reduce the chance of running out early, you could explore lowering spending or taking less investment risk.`
      );
    }
  } else {
    const yearsToRetire = simulator.retirementYear - simulator.year;
    const roundedYears = Math.max(Math.round(yearsToRetire), 0);
    const realStart = simulator.initialInvestment;

    if (real <= realStart * 0.9) {
      insightHeadline.value = "Early days - short term moves are mostly noise.";
      insightDetail.value = `In today's dollars your portfolio is about ${formatCurrency(
        real
      )}. With around ${roundedYears} years until ${Math.round(
        simulator.retirementYear
      )}, the main drivers of success will be consistent saving and staying invested, rather than reacting to short term performance.`;
    } else {
      insightHeadline.value = "On track - your nest egg is growing in real terms.";
      insightDetail.value = `In today's dollars your portfolio is about ${formatCurrency(
        real
      )}, above where you started. Staying in a risk profile that matches your goals and keeping contributions going over the next ${roundedYears} years will matter more than month by month market moves.`;
    }
  }
}

function updateStatusFromSimulator() {
  if (!simulator || !simulator.history.length) return;
  const lastRow = simulator.history[simulator.history.length - 1];

  currentRisk.value = simulator.riskProfile;
  phaseText.value = lastRow.Phase;
  yearText.value = simulator.year.toFixed(2);

  const real = lastRow["Real End Portfolio Balance"];

  withdrawalText.value = formatCurrency(simulator.baseWithdrawal);

  let withdrawalRate = 0;
  if (real > 0) {
    withdrawalRate = (simulator.baseWithdrawal * 12.0) / real;
  }
  withdrawalRateText.value = `${(withdrawalRate * 100).toFixed(2)}%`;

  const remainingYears = Math.max(simulator.deathYear - simulator.year, 0.01);
  const noReturnRate = 1.0 / remainingYears;
  trackToZeroText.value = `${(noReturnRate * 100).toFixed(2)}%`;

  computeWorstDrawdown(simulator.history);
  updateInsights(lastRow, real, withdrawalRate);
}

// ---- Simulation control ----

function startTimer() {
  stopTimer();
  timerId = setInterval(() => {
    if (!simulator) return;
    const finished = simulator.simulateMonth();
    updateChartFromSimulator();
    updateStatusFromSimulator();
    if (finished) {
      stopTimer();
      speedLabel.value = "Pause";
    }
  }, speedSettingMs);
}

function stopTimer() {
  if (timerId !== null) {
    clearInterval(timerId);
    timerId = null;
  }
}

function startNewSimulation() {
  stopTimer();
  speedLabel.value = "Pause";
  activeSavedRunIndex.value = null;

  const initialInvestment = form.initialInvestment || 0;
  const monthlySaving = form.monthlySaving || 0;
  const retirementYear = form.retirementYear || startYear + 25;
  const initialRiskProfile = form.initialRiskProfile || "Balanced";

  let monthlyWithdrawal;
  if (form.desiredMonthlyWithdrawal && form.desiredMonthlyWithdrawal > 0) {
    monthlyWithdrawal = form.desiredMonthlyWithdrawal;
  } else {
    const startingSafeWithdrawalAnnual = initialInvestment * 0.04;
    monthlyWithdrawal = startingSafeWithdrawalAnnual / 12.0;
  }

  simulator = new InvestmentPortfolioSimulator(
    initialInvestment,
    monthlySaving,
    initialRiskProfile,
    startYear,
    retirementYear,
    monthlyWithdrawal
  );

  // Initial snapshot so the chart is never blank
  simulator.history.push({
    MonthIndex: 0,
    Year: simulator.year,
    Phase: "Accumulation",
    RiskProfile: simulator.riskProfile,
    "Year Start Unit Value": simulator.unitValue,
    "Total Units Held": simulator.units,
    "Year End Unit Value": simulator.unitValue,
    Contributions: 0,
    Withdrawals: 0,
    "Year Start Portfolio Balance": initialInvestment,
    "Year End Portfolio Balance": initialInvestment,
    "Real End Portfolio Balance": initialInvestment,
    "Growth Amount": 0,
  });

  currentRisk.value = simulator.riskProfile;
  phaseText.value = "Accumulation";
  yearText.value = simulator.year.toFixed(2);
  withdrawalText.value = formatCurrency(simulator.baseWithdrawal);
  withdrawalRateText.value = "0.00%";
  trackToZeroText.value = "0.00%";
  insightHeadline.value = "";
  insightDetail.value = "";
  worstDrawdownText.value = "Not yet calculated";

  updateChartFromSimulator();
  updateStatusFromSimulator();
}

function resetSimulation() {
  if (!simulator) return;
  if (simulator.history && simulator.history.length) {
    const histCopy = [...simulator.history];
    const lastRisk =
      histCopy[histCopy.length - 1]?.RiskProfile || simulator.riskProfile;
    historicalRuns.value.push({ history: histCopy, risk: lastRisk });
  }

  // keep current risk profile as the new "initial"
  form.initialRiskProfile = simulator.riskProfile;

  startNewSimulation();
}

function clearHistory() {
  historicalRuns.value = [];
  activeSavedRunIndex.value = null;
  updateChartFromSimulator();
}

function setSpeed(mode) {
  speedLabel.value = mode;

  if (mode === "Pause") {
    stopTimer();
    return;
  }

  if (mode === "Normal") speedSettingMs = 1000;
  else if (mode === "Fast") speedSettingMs = 250;
  else if (mode === "Faster") speedSettingMs = 100;
  else if (mode === "Max") speedSettingMs = 10;

  if (simulator) {
    startTimer();
  }
}

function changeRisk(profile) {
  if (!simulator) return;
  simulator.setRiskProfile(profile);
  currentRisk.value = simulator.riskProfile;
  // keep select in sync so restart uses this risk
  form.initialRiskProfile = simulator.riskProfile;
}

function logEmotion(type) {
  if (!simulator) return;
  simulator.logEmotion(type);
  updateChartFromSimulator();
}

function adjustWithdrawal(direction) {
  if (!simulator || !simulator.history.length) return;

  const lastRow = simulator.history[simulator.history.length - 1];
  const real = lastRow["Real End Portfolio Balance"];
  if (real <= 0) return;

  const currentAnnual = simulator.baseWithdrawal * 12.0;
  let currentRate = currentAnnual / real; // fraction

  // round to nearest 0.5% step
  let roundedRate = Math.round(currentRate * 200) / 200; // 0.005 steps

  let newRate;
  if (direction === "down") {
    newRate = Math.max(0, roundedRate - 0.005);
  } else {
    newRate = roundedRate + 0.005;
  }

  const newAnnual = newRate * real;
  const newMonthly = newAnnual / 12.0;

  simulator.baseWithdrawal = newMonthly;
  simulator.recordSpendingChange(newRate);

  withdrawalText.value = formatCurrency(simulator.baseWithdrawal);
  updateStatusFromSimulator();
}

function setActiveSavedRun(index) {
  activeSavedRunIndex.value = index === activeSavedRunIndex.value ? null : index;
  updateChartFromSimulator();
}

// ---- Lifecycle ----

onMounted(() => {
  initChart();
});

onBeforeUnmount(() => {
  stopTimer();
  if (chart) {
    chart.destroy();
    chart = null;
  }
});
</script>

<style scoped>
.sim-root {
  font-family: Georgia, "Times New Roman", serif;
  padding: 1.5rem;
  color: #111827;
}

.title {
  font-weight: 700;
  font-size: 1.75rem;
  margin-bottom: 0.25rem;
}

.subtitle {
  font-size: 0.95rem;
  color: #4b5563;
  margin-bottom: 0.75rem;
}

.row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
  margin-bottom: 0.75rem;
}

.inputs .input-group {
  display: flex;
  flex-direction: column;
  min-width: 150px;
}

.input-group label {
  font-size: 0.85rem;
  margin-bottom: 0.25rem;
}

.input-group input,
.input-group select {
  padding: 0.35rem 0.5rem;
  border-radius: 4px;
  border: 1px solid #d1d5db;
  font-family: inherit;
  font-size: 0.9rem;
}

.input-group.buttons {
  justify-content: flex-end;
}

button {
  padding: 0.35rem 0.75rem;
  margin-right: 0.25rem;
  border-radius: 4px;
  border: 1px solid #d1d5db;
  background-color: #f9fafb;
  cursor: pointer;
  font-family: inherit;
  font-size: 0.85rem;
}

button:hover {
  background-color: #e5e7eb;
}

button.primary {
  background-color: #003366;
  border-color: #003366;
  color: #fff;
}

button.primary:hover {
  background-color: #02457a;
}

button.active {
  background-color: #003366;
  border-color: #003366;
  color: #ffffff;
}

button.active:hover {
  background-color: #02457a;
}

.chart-wrapper {
  height: 400px;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 0.5rem;
  margin-bottom: 0.4rem;
  background: #ffffff;
}

/* Control sections */

.controls {
  background: #f9fafb;
  border-radius: 6px;
  padding: 0.5rem 0.75rem;
}

.controls .control-group {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 0.25rem;
}

.group-label {
  font-weight: 600;
  margin-right: 0.5rem;
}

.speed-label {
  margin-left: 0.5rem;
  font-size: 0.8rem;
  color: #6b7280;
}

.runs-group {
  margin-left: auto;
}

/* Helper and microcopy */

.helper-text {
  font-size: 0.8rem;
  color: #6b7280;
  margin-top: 0.15rem;
  margin-bottom: 0.5rem;
}

.microcopy {
  font-size: 0.75rem;
  color: #6b7280;
  margin-left: 0.5rem;
}

/* Insights */

.insights-panel {
  padding: 0.75rem 1rem;
  margin-top: 0;
  margin-bottom: 0.4rem;
  border-radius: 6px;
  background: #f5f5f4;
  border: 1px solid #e5e7eb;
}

.insight-headline {
  font-weight: 600;
  margin-bottom: 0.25rem;
}

.insight-detail {
  font-size: 0.85rem;
  color: #4b5563;
  margin-bottom: 0.4rem;
}

.insight-metrics {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
  font-size: 0.8rem;
  color: #4b5563;
}

/* Saved runs chips */

.saved-runs {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 0.25rem;
  margin-bottom: 0.4rem;
  font-size: 0.8rem;
}

.saved-label {
  font-weight: 600;
  margin-right: 0.25rem;
}

.saved-chip {
  padding: 0.15rem 0.5rem;
  border-radius: 999px;
  border: 1px solid #d1d5db;
  background: #ffffff;
  cursor: pointer;
}

.saved-chip-active {
  background: #003366;
  border-color: #003366;
  color: #ffffff;
}

/* Status bar */

.status-bar {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
  margin-top: 0.75rem;
  padding-top: 0.75rem;
  border-top: 1px solid #e5e7eb;
  font-size: 0.85rem;
}

.status-highlight {
  font-weight: 600;
}

/* Risk dots */

.risk-dot {
  display: inline-block;
  width: 0.55rem;
  height: 0.55rem;
  border-radius: 999px;
  margin-right: 0.25rem;
}

/* Disclaimer */

.disclaimer {
  margin-top: 1rem;
  font-size: 0.75rem;
  color: #6b7280;
  line-height: 1.4;
}
</style>
