<template>
  <div :class="isDarkMode ? 'dark-mode' : 'light-mode'">
    <div class="sidebar">
      <ul>
        <li @click="goHome" :class="{ active: currentPage === 'home' }">{{ $t('home') }}</li>
        <li v-for="device in devices" 
            :key="device.id" 
            @click="selectDevice(device)"
            :class="{ active: selectedDevice?.id === device.id }">
          {{ device.name }}
        </li>
      </ul>
    </div>
    <div class="main-content">
      <div class="control-bar">
        <button @click="connectSerial" :disabled="isConnected" class="connect-btn">
          {{ isConnected ? `${$t('connected')} (${currentPort})` : $t('connectSerial') }}
        </button>
        <button @click="disconnectSerial" :disabled="!isConnected" class="disconnect-btn">
          {{ $t('disconnectSerial') }}
        </button>
        <select v-model="selectedLanguage" @change="changeLanguage" class="language-select">
          <option value="zh">{{ $t('chinese') }}</option>
          <option value="en">{{ $t('english') }}</option>
        </select>
        <button @click="toggleDarkMode" class="theme-btn">
          {{ isDarkMode ? $t('lightMode') : $t('darkMode') }}
        </button>
      </div>

      <div v-if="currentPage === 'home'" class="home-page">
        <h1>{{ $t('deviceManagement') }}</h1>
        <button @click="showAddDeviceModal = true" class="add-device-btn">
          {{ $t('addDevice') }}
        </button>
        <ul class="device-list">
          <li v-for="device in devices" :key="device.id" class="device-item">
            {{ device.name }}
            <button @click="deleteDevice(device.id)" class="delete-btn">
              {{ $t('deleteDevice') }}
            </button>
          </li>
        </ul>
      </div>

      <div v-if="selectedDevice && currentPage === 'device'" class="device-control">
        <div class="control-header">
          <h2>{{ selectedDevice.name }}</h2>
          <div class="connection-status">
            {{ isConnected ? `COM: ${currentPort}` : $t('notConnected') }}
          </div>
        </div>

        <div class="control-panel">
          <div class="control-section">
            <div class="parameter-display">
              <div class="param-box">
                <span class="param-label">{{ $t('brightness') }}</span>
                <span class="param-value">{{ selectedDevice.brightness }}</span>
              </div>
              <div class="param-box">
                <span class="param-label">{{ $t('colorTemperature') }}</span>
                <span class="param-value">{{ selectedDevice.temperature }}K</span>
              </div>
            </div>

            <div class="control-inputs">
              <div class="input-group">
                <label for="brightness">{{ $t('brightness') }}:</label>
                <input type="range" id="brightness" min="0" max="100" 
                  v-model="selectedDevice.brightness" 
                  class="brightness-slider"
                  ref="brightnessRef">
              </div>
              <div class="input-group">
                <label for="temperature">{{ $t('colorTemperature') }}:</label>
                <input  type="range" id="temperature" min="2000" max="6500" 
                  v-model="selectedDevice.temperature"
                  class="temp-slider"
                  ref="temperatureRef">
              </div>
            </div>

            <div class="control-buttons">
              <button @click="updateDevice(selectedDevice)" class="save-btn">
                {{ $t('saveChanges') }}
              </button>
            </div>
          </div>

          <div class="console-output">
            <div class="console-header">Console Output</div>
            <div class="console-content">
              <div v-for="(log, index) in commandLogs" :key="index" class="log-entry">
                {{ log }}
              </div>
            </div>
          </div>
        </div>
      </div>

      <div v-if="showAddDeviceModal" class="modal">
        <div class="modal-content">
          <h2>{{ $t('addDevice') }}</h2>
          <label for="newDeviceName">{{ $t('deviceName') }}:</label>
          <input type="text" id="newDeviceName" v-model="newDeviceName">
          <div class="modal-buttons">
            <button @click="addDevice" class="confirm-btn">{{ $t('confirmAdd') }}</button>
            <button @click="showAddDeviceModal = false" class="cancel-btn">{{ $t('cancel') }}</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { useI18n } from 'vue-i18n';

const devices = ref([
  { id: 1, name: '灯光设备 1', brightness: 100, temperature: 4000 },
  { id: 2, name: '灯光设备 2', brightness: 100, temperature: 4000 }
]);
const selectedDevice = ref(null);
const isConnected = ref(false);
const showAddDeviceModal = ref(false);
const newDeviceName = ref('');
const isDarkMode = ref(false);
const selectedLanguage = ref('zh');
const currentPage = ref('home');
const currentPort = ref('');
const commandLogs = ref([]);

const brightnessRef = ref(null);
const temperatureRef = ref(null);

const { t: $t, locale } = useI18n({
  legacy: false,
  locale: selectedLanguage.value,
  messages: {
    en: {
      deviceManagement: 'Device Management',
      connectSerial: 'Connect Serial',
      disconnectSerial: 'Disconnect Serial',
      connected: 'Connected',
      notConnected: 'Not Connected',
      addDevice: 'Add Device',
      english: 'English',
      chinese: 'Chinese',
      lightMode: 'Light Mode',
      darkMode: 'Dark Mode',
      controlInterface: 'Control Interface',
      brightness: 'Brightness',
      colorTemperature: 'Color Temperature',
      applySettings: 'Apply Settings',
      saveChanges: 'Save Changes',
      deleteDevice: 'Delete Device',
      deviceName: 'Device Name',
      confirmAdd: 'Confirm',
      cancel: 'Cancel',
      home: 'Home'
    },
    zh: {
      deviceManagement: '设备管理',
      connectSerial: '连接串口',
      disconnectSerial: '断开串口',
      connected: '已连接',
      notConnected: '未连接',
      addDevice: '添加设备',
      english: '英语',
      chinese: '中文',
      lightMode: '浅色模式',
      darkMode: '深色模式',
      controlInterface: '控制界面',
      brightness: '亮度',
      colorTemperature: '色温',
      applySettings: '应用设置',
      saveChanges: '发送',
      deleteDevice: '删除设备',
      deviceName: '设备名称',
      confirmAdd: '确认',
      cancel: '取消',
      home: '首页'
    }
  }
});

let port;
let writer;

const connectSerial = async () => {
  try {
    port = await navigator.serial.requestPort();
    await port.open({ baudRate: 9600 });
    writer = port.writable.getWriter();
    isConnected.value = true;
    const portInfo = port.getInfo();
    currentPort.value = `COM${portInfo.usbProductId || '?'}`;
    addLog(`Connected to ${currentPort.value}`);
  } catch (error) {
    console.error('连接串口时出错:', error);
    addLog(`Connection error: ${error.message}`);
  }
};

const disconnectSerial = async () => {
  try {
    if (writer) {
      await writer.close();
      writer = null;
    }
    if (port) {
      await port.close();
      port = null;
    }
    isConnected.value = false;
    currentPort.value = '';
    addLog('Serial port disconnected');
  } catch (error) {
    console.error('断开串口时出错:', error);
    addLog(`Disconnection error: ${error.message}`);
  }
};

const addLog = (message) => {
  commandLogs.value.unshift(`[${new Date().toLocaleTimeString()}] ${message}`);
  if (commandLogs.value.length > 100) {
    commandLogs.value.pop();
  }
};

const selectDevice = (device) => {
  selectedDevice.value = { ...device };
  currentPage.value = 'device';
  addLog(`Selected device: ${device.name}`);
  setSliderColors();
};

const sendCommand = async (deviceId) => {
  if (!writer) {
    addLog('Error: Serial port not connected');
    return;
  }
  const device = devices.value.find(item => item.id === deviceId);
  if (!device) {
    addLog('Error: Device not found');
    return;
  }
  const brightness = device.brightness;
  const temperature = device.temperature;
  const command = `${deviceId},${brightness},${temperature}\n`;
  try {
    const encoder = new TextEncoder();
    const data = encoder.encode(command);
    await writer.write(data);
    addLog(`Command sent: ${command}`);
  } catch (error) {
    addLog(`Error sending command: ${error.message}`);
  }
};

const addDevice = () => {
  if (newDeviceName.value) {
    const newId = Math.max(...devices.value.map(device => device.id), 0) + 1;
    devices.value.push({
      id: newId,
      name: newDeviceName.value,
      brightness: 100,
      temperature: 4000
    });
    newDeviceName.value = '';
    showAddDeviceModal.value = false;
    addLog(`Added new device: ${newDeviceName.value}`);
  }
};

const updateDevice = (device) => {
  const index = devices.value.findIndex(item => item.id === device.id);
  if (index !== -1) {
    devices.value[index] = { ...device };
    addLog(`Updated device: ${device.name}`);
    sendCommand(device.id);
  }
};

const deleteDevice = (deviceId) => {
  const device = devices.value.find(d => d.id === deviceId);
  devices.value = devices.value.filter(d => d.id !== deviceId);
  if (selectedDevice.value && selectedDevice.value.id === deviceId) {
    selectedDevice.value = null;
    currentPage.value = 'home';
  }
  addLog(`Deleted device: ${device.name}`);
};

const changeLanguage = () => {
  locale.value = selectedLanguage.value;
};

const toggleDarkMode = () => {
  isDarkMode.value = !isDarkMode.value;
  document.documentElement.classList.toggle('dark-mode', isDarkMode.value);
  document.documentElement.classList.toggle('light-mode', !isDarkMode.value);
};

const setSliderColors = () => {
  if (selectedDevice.value && brightnessRef.value && temperatureRef.value) {
    const brightness = selectedDevice.value.brightness;
    const temperature = selectedDevice.value.temperature;

    const brightnessGradient = `linear-gradient(to right, black ${(100 - brightness)}%, white ${brightness}%)`;
    brightnessRef.value.style.background = brightnessGradient;

    const minTemp = 2000;
    const maxTemp = 6500;
    const ratio = (temperature - minTemp) / (maxTemp - minTemp);
    const startColor = `rgb(255, 214, 10)`;
    const endColor = `rgb(255, 255, 255)`;
    const r = Math.round(parseInt(startColor.slice(4, 7)) + (parseInt(endColor.slice(4, 7)) - parseInt(startColor.slice(4, 7))) * ratio);
    const g = Math.round(parseInt(startColor.slice(8, 11)) + (parseInt(endColor.slice(8, 11)) - parseInt(startColor.slice(8, 11))) * ratio);
    const b = Math.round(parseInt(startColor.slice(12, 15)) + (parseInt(endColor.slice(12, 15)) - parseInt(startColor.slice(12, 15))) * ratio);
    const currentColor = `rgb(${r}, ${g}, ${b})`;
    const temperatureGradient = `linear-gradient(to right, rgb(255, 214, 10) ${(1 - ratio) * 100}%, ${currentColor} ${ratio * 100}%)`;
    temperatureRef.value.style.background = temperatureGradient;
  }
};

onMounted(() => {
  if (selectedDevice.value) {
    setSliderColors();
  }
});

const goHome = () => {
  currentPage.value = 'home';
  selectedDevice.value = null;
};
</script>

<style scoped>
body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  margin: 0;
  padding: 0;
  color: #333;
  transition: background-color 0.3s ease;
}

.dark-mode {
  background-color: #222;
  color: #fff;
}

.light-mode {
  background-color: #f4f4f4;
  color: #333;
}

.sidebar {
  position: fixed;
  top: 0;
  left: 0;
  width: 200px;
  height: 100vh;
  background-color: rgba(0, 0, 0, 0.1);
  padding: 20px;
  box-shadow: 2px 0 5px rgba(0, 0, 0, 0.1);
}

.sidebar ul {
  list-style-type: none;
  padding: 0;
}

.sidebar li {
  padding: 10px;
  cursor: pointer;
  border-radius: 4px;
  transition: background-color 0.3s ease;
}

.sidebar li:hover {
  background-color: rgba(0, 0, 0, 0.2);
}

.main-content {
  margin-left: 240px;
  padding: 20px;
}

.control-bar {
  display: flex;
  gap: 1rem;
  padding: 1rem;
  background: rgba(0, 0, 0, 0.1);
  border-radius: 8px;
  margin-bottom: 2rem;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.control-bar button {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.control-bar button:hover {
  background-color: rgba(0, 0, 0, 0.2);
}

.control-bar select {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
}

.home-page {
  background: rgba(0, 0, 0, 0.1);
  border-radius: 8px;
  padding: 2rem;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.home-page h1 {
  margin-top: 0;
}

.add-device-btn {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  background-color: #007BFF;
  color: white;
  transition: background-color 0.3s ease;
}

.add-device-btn:hover {
  background-color: #0056b3;
}

.device-list {
  list-style-type: none;
  padding: 0;
}

.device-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  background: rgba(0, 0, 0, 0.1);
  border-radius: 4px;
  margin-bottom: 10px;
}

.delete-btn {
  padding: 0.2rem 0.5rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  background-color: #dc3545;
  color: white;
  transition: background-color 0.3s ease;
}

.delete-btn:hover {
  background-color: #bd2130;
}

.device-control {
  background: rgba(0, 0, 0, 0.2);
  border-radius: 12px;
  padding: 2rem;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.control-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
}

.connection-status {
  padding: 0.5rem 1rem;
  background: rgba(0, 255, 0, 0.1);
  border-radius: 4px;
  font-family: monospace;
}

.control-panel {
  display: grid;
  grid-template-columns: 1fr 300px;
  gap: 2rem;
}

.parameter-display {
  display: flex;
  gap: 1rem;
  margin-bottom: 2rem;
}

.param-box {
  background: rgba(0, 0, 0, 0.3);
  padding: 1rem;
  border-radius: 8px;
  flex: 1;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.param-label {
  display: block;
  font-size: 0.9em;
  opacity: 0.7;
}

.param-value {
  display: block;
  font-size: 1.2em;
  font-family: monospace;
  margin-top: 0.5rem;
}

.console-output {
  background: rgba(0, 0, 0, 0.3);
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.console-header {
  padding: 0.5rem 1rem;
  background: rgba(255, 255, 255, 0.1);
  font-family: monospace;
}

.console-content {
  height: 400px;
  overflow-y: auto;
  padding: 1rem;
  font-family: monospace;
  font-size: 0.9em;
}

.log-entry {
  margin-bottom: 0.5rem;
  opacity: 0.8;
  transition: opacity 0.3s ease;
}

.log-entry:hover {
  opacity: 1;
}

.control-inputs {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  margin-bottom: 2rem;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

input[type="range"] {
  -webkit-appearance: none;
  width: 100%;
  height: 20px;
  border-radius: 10px;
  outline: none;
  transition: background 0.3s ease;
}

input[type="range"]::-webkit-slider-runnable-track {
  width: 100%;
  height: 20px;
  cursor: pointer;
  background: #ddd;
  border-radius: 10px;
}

input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  background: #007BFF;
  border-radius: 50%;
  cursor: pointer;
  transition: background 0.3s ease;
  margin-top: 0;
}

input[type="range"]::-moz-range-track {
  width: 100%;
  height: 20px;
  cursor: pointer;
  background: #ddd;
  border-radius: 10px;
}

input[type="range"]::-moz-range-thumb {
  width: 20px;
  height: 20px;
  background: #007BFF;
  border-radius: 50%;
  cursor: pointer;
  transition: background 0.3s ease;
}

input[type="range"]::-ms-track {
  width: 100%;
  height: 20px;
  cursor: pointer;
  background: transparent;
  border-color: transparent;
  color: transparent;
}

input[type="range"]::-ms-fill-lower {
  background: #ddd;
  border-radius: 10px;
}

input[type="range"]::-ms-fill-upper {
  background: #ddd;
  border-radius: 10px;
}

input[type="range"]::-ms-thumb {
  width: 20px;
  height: 20px;
  background: #007BFF;
  border-radius: 50%;
  cursor: pointer;
  transition: background 0.3s ease;
}

.control-buttons button {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  background-color: #28a745;
  color: white;
  transition: background-color 0.3s ease;
}

.control-buttons button:hover {
  background-color: #218838;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background: rgba(0, 0, 0, 0.8);
  border-radius: 12px;
  padding: 2rem;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
  width: 300px;
}

.modal h2 {
  margin-top: 0;
}

.modal input {
  width: 100%;
  padding: 0.5rem;
  border: none;
  border-radius: 4px;
  margin-bottom: 1rem;
}

.modal-buttons {
  display: flex;
  justify-content: space-between;
}

.modal-buttons button {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.confirm-btn {
  background-color: #28a745;
  color: white;
}

.confirm-btn:hover {
  background-color: #218838;
}

.cancel-btn {
  background-color: #dc3545;
  color: white;
}

.cancel-btn:hover {
  background-color: #bd2130;
}

.active {
  background: #ff0000 !important;
  color: white;
}
</style>    