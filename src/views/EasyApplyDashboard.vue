<template>
  <div class="py-4 container-fluid">
    <div class="row mb-4">
      <div class="col-xl-3 col-sm-6 mb-xl-0 mb-4" v-for="card in statsCards" :key="card.title">
        <card
          :title="card.title"
          :value="card.value"
          :percentage="card.percentage"
          :icon-class="card.iconClass"
          :icon-background="card.iconBackground"
          :percentage-color="card.percentageColor"
          direction-reverse
          v-motion-fade
        ></card>
      </div>
    </div>
    <div class="row mb-4">
      <div class="col-12">
        <div class="card">
          <div class="card-body p-3">
            <div class="d-flex justify-content-between align-items-center mb-3">
              <h5 class="font-weight-bolder mb-0">Live Bot Log</h5>
              <div>
                <button class="btn btn-success me-2" @click="startBot" :disabled="botRunning || loadingBot">{{ loadingBot && botRunning ? 'Starting...' : 'Start Bot' }}</button>
                <button class="btn btn-danger me-2" @click="stopBot" :disabled="!botRunning || loadingBot">{{ loadingBot && !botRunning ? 'Stopping...' : 'Stop Bot' }}</button>
                <button class="btn btn-info me-2" @click="exportLogs('csv')">Export CSV</button>
                <button class="btn btn-info" @click="exportLogs('json')">Export JSON</button>
              </div>
            </div>
            <div class="mb-3 row g-2 align-items-center">
              <div class="col-md-3">
                <input v-model="search" class="form-control" placeholder="Search logs..." />
              </div>
              <div class="col-md-3">
                <input type="date" v-model="dateFrom" class="form-control" />
              </div>
              <div class="col-md-3">
                <input type="date" v-model="dateTo" class="form-control" />
              </div>
              <div class="col-md-3">
                <select v-model="jobTitleFilter" class="form-select">
                  <option value="">All Job Titles</option>
                  <option v-for="jt in uniqueJobTitles" :key="jt">{{ jt }}</option>
                </select>
              </div>
              <div class="col-md-3 mt-2 mt-md-0">
                <select v-model="companyFilter" class="form-select">
                  <option value="">All Companies</option>
                  <option v-for="c in uniqueCompanies" :key="c">{{ c }}</option>
                </select>
              </div>
            </div>
            <div class="table-responsive animate__animated animate__fadeIn">
              <table class="table align-items-center mb-0">
                <thead>
                  <tr>
                    <th>Time</th>
                    <th>Job Title</th>
                    <th>Company</th>
                    <th>Status</th>
                    <th>Reason/Error</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="log in filteredLogs" :key="log.timestamp + log.job_title" v-motion-slide-bottom>
                    <td>{{ formatTime(log.timestamp) }}</td>
                    <td>{{ log.job_title }}</td>
                    <td>{{ log.company }}</td>
                    <td>
                      <span :class="statusClass(log.status)">{{ log.status }}</span>
                    </td>
                    <td>{{ log.reason || log.error }}</td>
                  </tr>
                </tbody>
              </table>
            </div>
            <div class="mt-4 animate__animated animate__fadeInUp">
              <apexchart type="donut" height="200" :options="chartOptions" :series="chartSeries" />
            </div>
            <div class="mt-4 animate__animated animate__fadeInUp">
              <apexchart type="bar" height="250" :options="barChartOptions" :series="barChartSeries" />
            </div>
            <div class="mt-4 animate__animated animate__fadeInUp">
              <apexchart type="line" height="250" :options="lineChartOptions" :series="lineChartSeries" />
            </div>
          </div>
        </div>
      </div>
    </div>
    <!-- Config Editor Modal -->
    <div v-if="showConfigEditor" class="modal fade show d-block" tabindex="-1" style="background:rgba(0,0,0,0.5)">
      <div class="modal-dialog">
        <div class="modal-content animate__animated animate__zoomIn">
          <div class="modal-header">
            <h5 class="modal-title">Edit Config</h5>
            <button type="button" class="btn-close" @click="showConfigEditor = false"></button>
          </div>
          <div class="modal-body">
            <textarea v-model="configContent" class="form-control" rows="10"></textarea>
          </div>
          <div class="modal-footer">
            <button class="btn btn-secondary" @click="showConfigEditor = false">Cancel</button>
            <button class="btn btn-primary" @click="saveConfig">Save</button>
          </div>
        </div>
      </div>
    </div>
    <button class="btn btn-outline-secondary mt-4 animate__animated animate__pulse animate__infinite" @click="openConfigEditor">Edit Config</button>
  </div>
</template>
<script>
import Card from "@/examples/Cards/Card.vue";
import axios from "axios";
import ApexChart from "vue3-apexcharts";
import { useMotion } from '@vueuse/motion';
import yaml from 'js-yaml';

const API_URL = "https://easyapply-backend.onrender.com/";

function groupByDate(logs, key = 'timestamp') {
  const grouped = {};
  logs.forEach(l => {
    if (!l[key]) return;
    const d = new Date(l[key]);
    const dateStr = d.toISOString().slice(0, 10);
    if (!grouped[dateStr]) grouped[dateStr] = [];
    grouped[dateStr].push(l);
  });
  return grouped;
}

export default {
  name: "EasyApplyDashboard",
  components: { Card, apexchart: ApexChart },
  data() {
    return {
      statsCards: [],
      logs: [],
      botRunning: false,
      loadingBot: false,
      search: "",
      showConfigEditor: false,
      configContent: "",
      dateFrom: "",
      dateTo: "",
      jobTitleFilter: "",
      companyFilter: "",
      chartOptions: {
        chart: { animations: { enabled: true } },
        labels: ["Success", "Failed", "Timeout", "Skipped"],
        colors: ["#82d616", "#ea0606", "#fbcf33", "#8392AB"],
        legend: { show: true, position: 'bottom' },
        dataLabels: { enabled: true },
        tooltip: { enabled: true },
      },
      chartSeries: [0, 0, 0, 0],
      barChartOptions: {
        chart: { animations: { enabled: true }, id: 'bar' },
        xaxis: { categories: [] },
        legend: { show: false },
        dataLabels: { enabled: false },
        tooltip: { enabled: true },
      },
      barChartSeries: [{ name: 'Applications', data: [] }],
      lineChartOptions: {
        chart: { animations: { enabled: true }, id: 'line' },
        xaxis: { categories: [] },
        legend: { show: false },
        dataLabels: { enabled: false },
        tooltip: { enabled: true },
      },
      lineChartSeries: [{ name: 'Success', data: [] }, { name: 'Failed', data: [] }],
      ws: null,
      toast: null,
    };
  },
  computed: {
    uniqueJobTitles() {
      const set = new Set(this.logs.map(l => l.job_title).filter(Boolean));
      return Array.from(set);
    },
    uniqueCompanies() {
      const set = new Set(this.logs.map(l => l.company).filter(Boolean));
      return Array.from(set);
    },
    filteredLogs() {
      let logs = this.logs;
      if (this.search) {
        const s = this.search.toLowerCase();
        logs = logs.filter(l =>
          (l.job_title || "").toLowerCase().includes(s) ||
          (l.company || "").toLowerCase().includes(s) ||
          (l.status || "").toLowerCase().includes(s) ||
          (l.reason || "").toLowerCase().includes(s) ||
          (l.error || "").toLowerCase().includes(s)
        );
      }
      if (this.dateFrom) {
        logs = logs.filter(l => l.timestamp && new Date(l.timestamp) >= new Date(this.dateFrom));
      }
      if (this.dateTo) {
        logs = logs.filter(l => l.timestamp && new Date(l.timestamp) <= new Date(this.dateTo + 'T23:59:59'));
      }
      if (this.jobTitleFilter) {
        logs = logs.filter(l => l.job_title === this.jobTitleFilter);
      }
      if (this.companyFilter) {
        logs = logs.filter(l => l.company === this.companyFilter);
      }
      return logs;
    },
  },
  watch: {
    filteredLogs: {
      handler() {
        this.updateStatsFromLogs();
        this.updateChartsFromLogs();
      },
      immediate: true,
    },
    logs: {
      handler() {
        this.updateStatsFromLogs();
        this.updateChartsFromLogs();
      },
      immediate: true,
    },
  },
  mounted() {
    this.fetchStatsAndLogs();
    this.connectWebSocket();
  },
  beforeUnmount() {
    if (this.ws) this.ws.close();
  },
  methods: {
    async fetchStatsAndLogs() {
      try {
        const logsRes = await axios.get(`${API_URL}/logs`);
        const logs = logsRes.data.logs.map(l => {
          try { return JSON.parse(l); } catch { return {}; }
        }).filter(l => l.status);
        this.logs = logs;
        this.updateStatsFromLogs();
        this.updateChartsFromLogs();
      } catch (e) { this.showToast('Failed to fetch logs', 'danger'); }
      try {
        const statusRes = await axios.get(`${API_URL}/status`);
        this.botRunning = statusRes.data.running;
      } catch (e) { this.showToast('Failed to fetch bot status', 'danger'); }
    },
    connectWebSocket() {
      if (this.ws) this.ws.close();
      this.ws = new WebSocket(`ws://localhost:8000/logs/stream`);
      this.ws.onmessage = (event) => {
        try {
          const log = JSON.parse(event.data);
          if (log.status) {
            this.logs.unshift(log);
            this.updateStatsFromLogs();
            this.updateChartsFromLogs();
          }
        } catch {}
      };
      this.ws.onclose = () => setTimeout(this.connectWebSocket, 2000);
    },
    updateStatsFromLogs() {
      const stats = { success: 0, failed: 0, timeout: 0, skipped: 0 };
      for (const l of this.filteredLogs) {
        if (l.status && stats[l.status.toLowerCase()] !== undefined) {
          stats[l.status.toLowerCase()]++;
        }
      }
      this.statsCards = [
        { title: "Success", value: stats.success, percentage: "", iconClass: "ni ni-check-bold", iconBackground: "bg-gradient-success" },
        { title: "Failed", value: stats.failed, percentage: "", iconClass: "ni ni-fat-remove", iconBackground: "bg-gradient-danger", percentageColor: "text-danger" },
        { title: "Timeout", value: stats.timeout, percentage: "", iconClass: "ni ni-watch-time", iconBackground: "bg-gradient-warning" },
        { title: "Skipped", value: stats.skipped, percentage: "", iconClass: "ni ni-skip-forward", iconBackground: "bg-gradient-secondary" },
      ];
      this.chartSeries = [stats.success, stats.failed, stats.timeout, stats.skipped];
    },
    updateChartsFromLogs() {
      // Bar/Line: group by date
      const grouped = groupByDate(this.filteredLogs);
      const dates = Object.keys(grouped).sort();
      this.barChartOptions.xaxis.categories = dates;
      this.lineChartOptions.xaxis.categories = dates;
      this.barChartSeries = [{
        name: 'Applications',
        data: dates.map(d => grouped[d].length)
      }];
      this.lineChartSeries = [
        {
          name: 'Success',
          data: dates.map(d => grouped[d].filter(l => l.status && l.status.toLowerCase() === 'success').length)
        },
        {
          name: 'Failed',
          data: dates.map(d => grouped[d].filter(l => l.status && l.status.toLowerCase() === 'failed').length)
        }
      ];
    },
    formatTime(ts) {
      if (!ts) return "";
      return new Date(ts).toLocaleString();
    },
    statusClass(status) {
      switch ((status || "").toLowerCase()) {
        case "success": return "badge bg-gradient-success";
        case "failed": return "badge bg-gradient-danger";
        case "timeout": return "badge bg-gradient-warning";
        case "skipped": return "badge bg-gradient-secondary";
        default: return "badge bg-gradient-light";
      }
    },
    async startBot() {
      this.loadingBot = true;
      try {
        const res = await axios.post(`${API_URL}/start-bot`);
        if (res.data.status === 'started' || res.data.status === 'already_running') {
          this.botRunning = true;
          this.showToast('Bot started!', 'success');
        } else {
          this.showToast('Failed to start bot: ' + (res.data.error || ''), 'danger');
        }
      } catch (e) { this.showToast('Failed to start bot', 'danger'); }
      this.loadingBot = false;
    },
    async stopBot() {
      this.loadingBot = true;
      try {
        const res = await axios.post(`${API_URL}/stop-bot`);
        if (res.data.status === 'stopped') {
          this.botRunning = false;
          this.showToast('Bot stopped!', 'info');
        } else {
          this.showToast('Failed to stop bot', 'danger');
        }
      } catch (e) { this.showToast('Failed to stop bot', 'danger'); }
      this.loadingBot = false;
    },
    exportLogs(type) {
      if (type === 'csv') {
        const csv = this.filteredLogs.map(l => `${l.timestamp},${l.job_title},${l.company},${l.status},${l.reason || l.error}`).join('\n');
        const blob = new Blob([csv], { type: 'text/csv' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = 'logs.csv';
        a.click();
        URL.revokeObjectURL(url);
      } else if (type === 'json') {
        const blob = new Blob([JSON.stringify(this.filteredLogs, null, 2)], { type: 'application/json' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = 'logs.json';
        a.click();
        URL.revokeObjectURL(url);
      }
    },
    async openConfigEditor() {
      try {
        const res = await axios.get(`${API_URL}/config`);
        this.configContent = res.data.content;
        this.showConfigEditor = true;
      } catch (e) { this.showToast('Failed to load config', 'danger'); }
    },
    async saveConfig() {
      try {
        await axios.post(`${API_URL}/config`, { content: this.configContent });
        this.showToast('Config saved!', 'success');
        this.showConfigEditor = false;
      } catch (e) { this.showToast('Failed to save config', 'danger'); }
    },
    showToast(msg, type) {
      if (!this.toast) {
        this.toast = document.createElement('div');
        this.toast.className = `toast align-items-center text-white bg-${type} border-0 position-fixed top-0 end-0 m-4 animate__animated animate__fadeInDown`;
        this.toast.style.zIndex = 9999;
        this.toast.innerHTML = `<div class='d-flex'><div class='toast-body'>${msg}</div><button type='button' class='btn-close btn-close-white me-2 m-auto' data-bs-dismiss='toast'></button></div>`;
        document.body.appendChild(this.toast);
        setTimeout(() => { this.toast.remove(); this.toast = null; }, 3000);
      }
    },
  },
};
</script>
<style scoped>
.badge { font-size: 0.9em; padding: 0.5em 1em; }
</style> 