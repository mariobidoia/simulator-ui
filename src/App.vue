<template>
  <div class="min-h-screen bg-slate-900 text-slate-200 p-4 font-sans">

    <!-- Cabeçalho e Configurações de Conexão -->
    <header
      class="bg-slate-800 rounded-lg p-4 mb-4 shadow-lg border border-slate-700 flex flex-wrap gap-4 items-end justify-between">
      <div>
        <h1 class="text-2xl font-bold text-cyan-400 flex items-center gap-2">
          <span>❄️</span> Simulador de Placa Inverter (Gêmeo Digital)
        </h1>
        <div class="mt-4 flex items-center gap-3 bg-slate-900 p-2 rounded border border-slate-700 w-fit">
          <label class="text-sm font-bold text-cyan-400">Selecionar Máquina:</label>
          <select v-model="currentDeviceId" @change="loadDevice"
            class="bg-slate-800 border border-slate-600 rounded px-3 py-1 text-sm font-mono focus:outline-none focus:border-cyan-500">
            <option v-for="id in deviceList" :key="id" :value="id">{{ id }}</option>
          </select>
          <button @click="createNewDevice"
            class="bg-slate-700 hover:bg-slate-600 px-2 py-1 rounded text-xs transition-colors">➕ Nova Placa</button>
        </div>
        <p class="text-slate-400 text-sm mt-1">Status do Envio:
          <span :class="config.lastHttpStatusCode === 200 ? 'text-green-400' : 'text-red-400 font-bold'">
            {{ config.lastDispatchStatusMessage || 'Aguardando...' }}
          </span>
          | Pacotes: {{ config.totalPacketsSent }}
        </p>
      </div>

      <div class="flex gap-4 items-end">
        <div class="flex flex-col">
          <label class="text-xs text-slate-400 mb-1">URL da API Destino</label>
          <input v-model="config.targetApiUrl" @change="saveConfig" type="text"
            class="bg-slate-950 border border-slate-600 rounded px-3 py-1.5 text-sm w-72 focus:outline-none focus:border-cyan-500" />
        </div>
        <div class="flex flex-col">
          <label class="text-xs text-slate-400 mb-1">Intervalo (ms)</label>
          <input v-model.number="config.intervalMilliseconds" @change="saveConfig" type="number" step="500"
            class="bg-slate-950 border border-slate-600 rounded px-3 py-1.5 text-sm w-24 text-center focus:outline-none focus:border-cyan-500" />
        </div>
        <div class="flex items-center gap-2 bg-slate-950 border border-slate-600 rounded px-3 py-1.5 h-[34px]">
          <input v-model="config.applySensorJitter" @change="saveConfig" type="checkbox" id="jitter"
            class="accent-cyan-500" />
          <label for="jitter" class="text-sm cursor-pointer select-none">Ruído Realista</label>
        </div>
        <button @click="toggleTransmission"
          :class="['px-4 py-1.5 rounded text-sm font-bold transition-colors', config.isTransmissionActive ? 'bg-red-600 hover:bg-red-500 text-white' : 'bg-green-600 hover:bg-green-500 text-white']">
          {{ config.isTransmissionActive ? 'Parar Envio' : 'Iniciar Envio' }}
        </button>
      </div>
    </header>

    <div class="grid grid-cols-1 xl:grid-cols-12 gap-4">

      <!-- Lado Esquerdo: Controles e Sliders (8 colunas) -->
      <div class="xl:col-span-8 flex flex-col gap-4">

        <!-- Presets de Falha -->
        <section class="bg-slate-800 rounded-lg p-4 border border-slate-700 shadow-lg">
          <h2 class="text-sm font-bold text-slate-400 uppercase tracking-wider mb-3">Injeção Rápida de Falhas (Presets)
          </h2>
          <div class="flex flex-wrap gap-2">
            <button @click="applyPreset('NORMAL')"
              class="bg-slate-700 hover:bg-cyan-600 px-3 py-2 rounded text-sm transition-colors border border-slate-600">✅
              Normal</button>
            <button @click="applyPreset('GAS_LEAK')"
              class="bg-slate-700 hover:bg-amber-600 px-3 py-2 rounded text-sm transition-colors border border-slate-600">💨
              Vazamento (EC)</button>
            <button @click="applyPreset('CONDENSER_FAN_FAIL')"
              class="bg-slate-700 hover:bg-red-600 px-3 py-2 rounded text-sm transition-colors border border-slate-600">🛑
              Vent. Travado (P4)</button>
            <button @click="applyPreset('LOW_VOLTAGE')"
              class="bg-slate-700 hover:bg-orange-600 px-3 py-2 rounded text-sm transition-colors border border-slate-600">⚡
              Subtensão (P1)</button>
            <button @click="applyPreset('COMM_ERROR')"
              class="bg-slate-700 hover:bg-purple-600 px-3 py-2 rounded text-sm transition-colors border border-slate-600">🔌
              Erro Comunicação (E1)</button>
          </div>
        </section>

        <!-- Grid de Cards de Parâmetros -->
        <div class="grid grid-cols-1 md:grid-cols-2 gap-4" v-if="board">

          <!-- Card Operacional -->
          <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
            <h3 class="text-cyan-400 font-bold mb-3 border-b border-slate-700 pb-2">Estado Operacional</h3>
            <div class="space-y-3">
              <div class="flex justify-between items-center">
                <label class="text-sm">Power</label>
                <input type="checkbox" v-model="board.operationalState.powerState" @change="debouncedSaveState"
                  class="w-5 h-5 accent-cyan-500" />
              </div>
              <div class="flex justify-between items-center">
                <label class="text-sm">Modo</label>
                <select v-model="board.operationalState.operationMode" @change="debouncedSaveState"
                  class="bg-slate-950 border border-slate-600 rounded px-2 py-1 text-sm">
                  <option value="COOL">COOL (Frio)</option>
                  <option value="HEAT">HEAT (Quente)</option>
                  <option value="FAN">FAN (Ventilação)</option>
                  <option value="DRY">DRY (Desumidificar)</option>
                </select>
              </div>
              <div class="flex justify-between items-center">
                <label class="text-sm">Erro (Display)</label>
                <input type="text" v-model="board.diagnostics.errorCode" @input="debouncedSaveState"
                  class="bg-slate-950 border border-slate-600 rounded px-2 py-1 text-sm w-24 text-center font-mono text-red-400 uppercase" />
              </div>
            </div>
          </div>

          <!-- Card Atuadores -->
          <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
            <h3 class="text-cyan-400 font-bold mb-3 border-b border-slate-700 pb-2">Atuadores / Motores</h3>
            <div class="space-y-4">
              <div>
                <div class="flex justify-between text-sm mb-1">
                  <label>Compressor (Hz)</label>
                  <span class="font-mono text-amber-400">{{ board.actuators.compressorFrequencyHz }} Hz</span>
                </div>
                <input type="range" v-model.number="board.actuators.compressorFrequencyHz" @input="debouncedSaveState"
                  min="0" max="120" step="1" class="w-full accent-cyan-500" />
              </div>
              <div>
                <div class="flex justify-between text-sm mb-1">
                  <label>Válvula EEV (Passos)</label>
                  <span class="font-mono text-amber-400">{{ board.refrigerationCycle.eevOpeningSteps }}</span>
                </div>
                <input type="range" v-model.number="board.refrigerationCycle.eevOpeningSteps"
                  @input="debouncedSaveState" min="0" max="480" step="10" class="w-full accent-cyan-500" />
              </div>
            </div>
          </div>

          <!-- Card Temperaturas -->
          <div class="bg-slate-800 rounded-lg p-4 border border-slate-700 md:col-span-2">
            <h3 class="text-cyan-400 font-bold mb-3 border-b border-slate-700 pb-2">Sensores de Temperatura (°C)</h3>
            <div class="grid grid-cols-1 md:grid-cols-2 gap-x-6 gap-y-4">
              <SliderControl v-model="board.temperatures.setpointC" label="Setpoint (Alvo)" min="16" max="30" step="1"
                @update="debouncedSaveState" />
              <SliderControl v-model="board.temperatures.indoorReturnAirC" label="Ar Retorno (Ambiente)" min="0"
                max="50" step="0.5" @update="debouncedSaveState" />
              <SliderControl v-model="board.temperatures.evaporatorCoilC" label="Serpentina Evap. (T2)" min="-10"
                max="40" step="0.5" @update="debouncedSaveState" />
              <SliderControl v-model="board.temperatures.outdoorAmbientC" label="Ar Externo (T4)" min="-10" max="60"
                step="0.5" @update="debouncedSaveState" />
              <SliderControl v-model="board.temperatures.condenserCoilC" label="Serpentina Cond. (T3)" min="-10"
                max="80" step="0.5" @update="debouncedSaveState" />
              <SliderControl v-model="board.temperatures.compressorDischargeC" label="Descarga Comp. (T5)" min="0"
                max="130" step="1" @update="debouncedSaveState" />
              <SliderControl v-model="board.temperatures.ipmModuleC" label="Módulo IPM" min="0" max="110" step="1"
                @update="debouncedSaveState" />
            </div>
          </div>

          <!-- Card Pressões -->
          <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
            <h3 class="text-cyan-400 font-bold mb-3 border-b border-slate-700 pb-2">Pressões (PSI)</h3>
            <div class="space-y-4">
              <SliderControl v-model="board.refrigerationCycle.suctionPressurePsi" label="Sucção (Baixa)" min="0"
                max="250" step="1" @update="debouncedSaveState" />
              <SliderControl v-model="board.refrigerationCycle.dischargePressurePsi" label="Descarga (Alta)" min="0"
                max="650" step="1" @update="debouncedSaveState" />
            </div>
          </div>

          <!-- Card Elétrico -->
          <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
            <h3 class="text-cyan-400 font-bold mb-3 border-b border-slate-700 pb-2">Grandezas Elétricas</h3>
            <div class="space-y-4">
              <SliderControl v-model="board.electrical.acInputVoltageV" label="Tensão AC (Rede)" min="0" max="260"
                step="1" @update="debouncedSaveState" />
              <SliderControl v-model="board.electrical.totalCurrentA" label="Corrente Total (A)" min="0" max="25"
                step="0.1" @update="debouncedSaveState" />
            </div>
          </div>

        </div>
      </div>

      <!-- Lado Direito: Live JSON (4 colunas) -->
      <div
        class="xl:col-span-4 bg-slate-950 rounded-lg border border-slate-700 shadow-lg flex flex-col overflow-hidden max-h-[85vh]">
        <div class="bg-slate-800 p-3 border-b border-slate-700 flex justify-between items-center">
          <h2 class="text-sm font-bold text-slate-400 uppercase tracking-wider">Payload JSON ao Vivo</h2>
          <span class="flex h-3 w-3">
            <span v-if="config.isTransmissionActive"
              class="animate-ping absolute inline-flex h-3 w-3 rounded-full bg-cyan-400 opacity-75"></span>
            <span class="relative inline-flex rounded-full h-3 w-3"
              :class="config.isTransmissionActive ? 'bg-cyan-500' : 'bg-slate-600'"></span>
          </span>
        </div>
        <div class="p-4 overflow-y-auto flex-1 custom-scrollbar">
          <pre class="text-xs text-green-400 font-mono whitespace-pre-wrap">{{ jsonPreview }}</pre>
        </div>
      </div>

    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue';
import SliderControl from './components/SliderControl.vue';

const API_BASE_URL = 'https://localhost:7100/api/simulator';

const deviceList = ref([]);
const currentDeviceId = ref('');
const board = ref(null);
const config = ref({ targetApiUrl: '', intervalMilliseconds: 3000, isTransmissionActive: false, applySensorJitter: false, lastHttpStatusCode: null, lastDispatchStatusMessage: '', totalPacketsSent: 0 });

let pollingInterval = null;
let saveTimeout = null;

const jsonPreview = computed(() => board.value ? JSON.stringify(board.value, null, 2) : 'Carregando...');

// Carrega as máquinas disponíveis e seleciona a primeira
const fetchDevices = async () => {
  const res = await fetch(`${API_BASE_URL}/devices`);
  deviceList.value = await res.json();
  if (deviceList.value.length > 0 && !currentDeviceId.value) {
    currentDeviceId.value = deviceList.value[0];
    await loadDevice();
  }
};

// Ao trocar a placa no dropdown, busca os dados específicos dela
const loadDevice = async () => {
  await fetchConfig();
  await fetchState();
};

const createNewDevice = async () => {
  const newId = prompt("Digite o ID da nova placa (ex: AC-LAB-003):");
  if (newId) {
    await fetch(`${API_BASE_URL}/devices/${newId}`, { method: 'POST' });
    await fetchDevices();
    currentDeviceId.value = newId;
    await loadDevice();
  }
};

const fetchState = async () => {
  if (!currentDeviceId.value) return;
  const res = await fetch(`${API_BASE_URL}/${currentDeviceId.value}/state`);
  if (res.ok) board.value = await res.json();
};

const fetchConfig = async () => {
  if (!currentDeviceId.value) return;
  const res = await fetch(`${API_BASE_URL}/${currentDeviceId.value}/config`);
  if (res.ok) config.value = await res.json();
};

const debouncedSaveState = () => {
  clearTimeout(saveTimeout);
  saveTimeout = setTimeout(async () => {
    await fetch(`${API_BASE_URL}/${currentDeviceId.value}/state`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(board.value)
    });
  }, 300);
};

const applyPreset = async (presetName) => {
  await fetch(`${API_BASE_URL}/${currentDeviceId.value}/preset/${presetName}`, { method: 'POST' });
  await fetchState();
};

const saveConfig = async () => {
  await fetch(`${API_BASE_URL}/${currentDeviceId.value}/config`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(config.value)
  });
};

const toggleTransmission = async () => {
  config.value.isTransmissionActive = !config.value.isTransmissionActive;
  await saveConfig();
};

onMounted(async () => {
  await fetchDevices();
  pollingInterval = setInterval(fetchConfig, 2000);
});

onUnmounted(() => {
  clearInterval(pollingInterval);
  clearTimeout(saveTimeout);
});
</script>


<style>
/* Personalização básica para a scrollbar do JSON */
.custom-scrollbar::-webkit-scrollbar {
  width: 8px;
}

.custom-scrollbar::-webkit-scrollbar-track {
  background: #0f172a;
}

.custom-scrollbar::-webkit-scrollbar-thumb {
  background: #334155;
  border-radius: 4px;
}

.custom-scrollbar::-webkit-scrollbar-thumb:hover {
  background: #475569;
}
</style>