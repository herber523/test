<script setup>
import { ref, onMounted, onUnmounted, computed, watch, nextTick } from 'vue';
import L from 'leaflet';
import { joinRoom as joinNostr } from '@trystero-p2p/nostr';
import { joinRoom as joinMqtt } from '@trystero-p2p/mqtt';
import { joinRoom as joinTorrent } from '@trystero-p2p/torrent';

// --- 0. On-Screen Debugger (Improved for Mobile) ---
const logs = ref([]);
const isLogVisible = ref(false);
const addLog = (msg, type = 'info') => {
  const time = new Date().toLocaleTimeString();
  logs.value.unshift({ time, msg, type });
  if (logs.value.length > 50) logs.value.pop();
  console.log(`[${type.toUpperCase()}] ${msg}`);
};

// --- 1. Identity & P2P Data Management ---
const myId = ref(''); 
const myName = ref('');
const myLocation = ref(null);
const lastSyncAt = ref(Date.now());
const others = ref(new Map()); // uid -> {name, lat, lng, peerIds: Map, lastSeen: number, isOnline: bool}

// --- 2. Trystero P2P Setup ---
const RTC_CONFIG = {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' },
    { urls: 'stun:stun2.l.google.com:19302' },
    { urls: 'stun:stun.cloudflare.com:3478' }
  ]
};

const STRATEGIES = [
  { name: 'nostr', join: joinNostr },
  { name: 'mqtt', join: joinMqtt },
  { name: 'torrent', join: joinTorrent }
];

const rooms = ref(new Map()); 
const locActions = ref(new Map()); 

const initP2P = () => {
  const params = new URLSearchParams(window.location.search);
  let roomId = params.get('room');
  if (!roomId) {
    roomId = 'r_' + Math.random().toString(36).substring(2, 8);
    params.set('room', roomId);
    window.history.replaceState({}, '', `${window.location.pathname}?${params.toString()}`);
  }

  addLog(`Room Target: ${roomId}`, 'sys');

  STRATEGIES.forEach(strat => {
    try {
      addLog(`[${strat.name}] Init...`, 'info');
      const room = strat.join({ appId: 'p2p-pin-locator-v1', rtcConfig: RTC_CONFIG }, roomId);
      const locAction = room.makeAction('loc');
      
      rooms.value.set(strat.name, room);
      locActions.value.set(strat.name, locAction);

      locAction.onMessage = (data, meta) => {
        const peerId = typeof meta === 'object' ? meta.peerId : meta;
        addLog(`[${strat.name}] 📥 RECV from ${data.uid}`, 'recv');
        
        if (!data.uid) return;
        const now = Date.now();
        const existing = others.value.get(data.uid) || { peerIds: new Map() };
        others.value.set(data.uid, { ...existing, ...data, lastSeen: now, isOnline: true, peerId });
        others.value.get(data.uid).peerIds.set(strat.name, peerId);
        updateMapMarkers();
      };

      room.onPeerJoin = (peerId) => {
        addLog(`[${strat.name}] 🟢 PEER IN: ${peerId.slice(0,6)}`, 'success');
        if (myLocation.value) {
          const payload = { uid: myId.value, name: myName.value, lat: myLocation.value.lat, lng: myLocation.value.lng };
          addLog(`[${strat.name}] 📤 Greeting to ${peerId.slice(0,6)}`, 'info');
          locAction.send(payload, peerId);
        }
      };

      room.onPeerLeave = (peerId) => {
        addLog(`[${strat.name}] 🔴 PEER OUT: ${peerId.slice(0,6)}`, 'warn');
        for (const [uid, data] of others.value.entries()) {
          if (data.peerIds.get(strat.name) === peerId) {
            data.peerIds.delete(strat.name);
            if (data.peerIds.size === 0) data.isOnline = false;
            break;
          }
        }
        updateMapMarkers();
      };

    } catch (e) {
      addLog(`[${strat.name}] ❌ ERROR: ${e.message}`, 'error');
    }
  });

  setInterval(() => {
    rooms.value.forEach((room, name) => {
      if (room.getRelaySockets) {
        const sockets = room.getRelaySockets();
        const connected = Object.values(sockets).filter(s => s.readyState === 1).length;
        if (connected === 0) addLog(`[${name}] ⚠️ Connection Lost`, 'warn');
      }
    });
  }, 10000);
};

// --- 3. UI & State ---
const isLocating = ref(false);
const mapInstance = ref(null);
const markers = ref({}); 
const hasInitiallyCentered = ref(false);
const nextSyncIn = ref(10); 
const toastVisible = ref(false);
const toastMessage = ref('');
let toastTimeout = null;

const showToast = (msg) => {
  toastMessage.value = msg;
  toastVisible.value = true;
  if (toastTimeout) clearTimeout(toastTimeout);
  toastTimeout = setTimeout(() => { toastVisible.value = false; }, 3000);
};

const activeOthers = computed(() => {
  return Array.from(others.value.entries()).sort((a, b) => {
    if (a[1].isOnline !== b[1].isOnline) return b[1].isOnline ? 1 : -1;
    return b[1].lastSeen - a[1].lastSeen;
  });
});

const getInitials = (name) => (name || '?').charAt(0).toUpperCase();
const formatTime = (ts) => {
  if (!ts) return '未知';
  const sec = Math.floor((Date.now() - ts) / 1000);
  if (sec < 60) return `${sec}s 前`;
  return new Date(ts).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
};

// --- 4. Main Location Logic ---
let syncInterval = null;
let timerInterval = null;

const updateLocation = (isManualClick = false) => {
  if (!navigator.geolocation) return addLog('GPS: Not Supported', 'error');
  if (!myName.value.trim()) return;

  addLog('GPS: Fetching...', 'info');
  isLocating.value = true;
  navigator.geolocation.getCurrentPosition(
    (pos) => {
      isLocating.value = false;
      const { latitude: lat, longitude: lng } = pos.coords;
      myLocation.value = { lat, lng };
      lastSyncAt.value = Date.now();

      const payload = { uid: myId.value, name: myName.value, lat, lng };
      addLog(`GPS: OK (${lat.toFixed(4)},${lng.toFixed(4)})`, 'success');

      let sentCount = 0;
      locActions.value.forEach((action, name) => {
        action.send(payload);
        sentCount++;
      });
      if (sentCount > 0) addLog(`📤 BROADCAST (${sentCount} strats)`, 'info');

      if (isManualClick) showToast('📍 位置已更新！');
      updateMapMarkers();
      nextSyncIn.value = 10; 
    },
    (err) => {
      isLocating.value = false;
      addLog(`GPS: FAIL - ${err.message}`, 'error');
    },
    { enableHighAccuracy: true, timeout: 10000, maximumAge: 0 }
  );
};

const startAutoSync = () => {
  if (syncInterval) clearInterval(syncInterval);
  if (timerInterval) clearInterval(timerInterval);
  updateLocation(false);
  syncInterval = setInterval(() => updateLocation(false), 10000);
  timerInterval = setInterval(() => { if (nextSyncIn.value > 0) nextSyncIn.value--; }, 1000);
};

const handleShare = () => {
  navigator.clipboard.writeText(window.location.href);
  showToast('網址已複製！');
  updateLocation(false);
};

// --- 5. Map Rendering ---
const fixLeafletIcon = () => {
  delete L.Icon.Default.prototype._getIconUrl;
  L.Icon.Default.mergeOptions({
    iconRetinaUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon-2x.png',
    iconUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon.png',
    shadowUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png',
  });
};

const createMarkerIcon = (name, isMe = false, isOnline = true) => {
  const colorClass = isMe ? 'bg-blue-600' : (isOnline ? 'bg-gray-800' : 'bg-gray-400 grayscale');
  return L.divIcon({
    className: 'bg-transparent border-none',
    html: `<div class="flex items-center justify-center w-8 h-8 rounded-full ${colorClass} text-white font-bold shadow-lg ring-2 ring-white text-xs opacity-${isOnline ? '100' : '70'}">${getInitials(name)}</div>`,
    iconSize: [32, 32],
    iconAnchor: [16, 32],
    popupAnchor: [0, -32]
  });
};

const initMap = () => {
  fixLeafletIcon();
  const map = L.map('map', { zoomControl: false }).setView([23.58, 120.58], 15);
  L.control.zoom({ position: 'bottomright' }).addTo(map);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', { attribution: '&copy; OSM' }).addTo(map);
  mapInstance.value = map;
};

const updateMapMarkers = () => {
  if (!mapInstance.value) return;
  const map = mapInstance.value;
  activeOthers.value.forEach(([uid, data]) => {
    if (!markers.value[uid]) {
      markers.value[uid] = L.marker([data.lat, data.lng], { icon: createMarkerIcon(data.name, false, data.isOnline) })
        .bindPopup(`<div class="font-sans text-sm font-medium px-1">${data.name}<br/><span class="text-[10px] text-gray-400">最後更新: ${formatTime(data.lastSeen)}</span></div>`)
        .addTo(map);
    } else {
      markers.value[uid].setLatLng([data.lat, data.lng]);
      markers.value[uid].setIcon(createMarkerIcon(data.name, false, data.isOnline));
    }
  });
  if (myLocation.value && myName.value) {
    if (!markers.value['me']) {
      markers.value['me'] = L.marker([myLocation.value.lat, myLocation.value.lng], { icon: createMarkerIcon(myName.value, true), zIndexOffset: 1000 }).addTo(map);
    } else {
      markers.value['me'].setLatLng([myLocation.value.lat, myLocation.value.lng]);
    }
    if (!hasInitiallyCentered.value) {
      map.flyTo([myLocation.value.lat, myLocation.value.lng], 15, { animate: true, duration: 1 });
      hasInitiallyCentered.value = true;
    }
  }
};

watch(myName, (newName) => {
  if (newName) {
    localStorage.setItem('locator_name', newName.trim());
    if (markers.value['me']) markers.value['me'].setIcon(createMarkerIcon(newName, true));
    const payload = { uid: myId.value, name: newName.trim(), ...myLocation.value };
    locActions.value.forEach(action => action.send(payload));
  }
});

onMounted(() => {
  myId.value = localStorage.getItem('locator_uid') || ('u_' + Math.random().toString(36).substring(2, 8));
  localStorage.setItem('locator_uid', myId.value);
  myName.value = localStorage.getItem('locator_name') || ('訪客_' + Math.floor(Math.random() * 9999));
  localStorage.setItem('locator_name', myName.value);
  initP2P();
  nextTick(() => { initMap(); startAutoSync(); });
});

onUnmounted(() => {
  if (syncInterval) clearInterval(syncInterval);
  if (timerInterval) clearInterval(timerInterval);
});
</script>

<template>
  <!-- Main App Layout (Ensured Scrollable) -->
  <div class="h-screen w-full flex flex-col md:flex-row bg-gray-50 overflow-y-auto md:overflow-hidden relative">
    
    <!-- Floating Debug Toggle Button -->
    <button @click="isLogVisible = !isLogVisible" class="fixed bottom-4 right-4 z-[3000] w-12 h-12 bg-black/80 text-white rounded-full shadow-2xl flex items-center justify-center text-xl border border-white/20 active:scale-90 transition-transform">
      🛠️
    </button>

    <!-- Debug Modal Overlay -->
    <div v-if="isLogVisible" class="fixed inset-0 z-[4000] bg-black/95 flex flex-col p-4 animate-fade-in">
       <div class="flex justify-between items-center mb-4 border-b border-white/10 pb-2">
         <h3 class="text-white font-bold">P2P DEBUG LOGS</h3>
         <button @click="isLogVisible = false" class="px-4 py-1 bg-white/10 text-white rounded-lg">關閉</button>
       </div>
       <div class="flex-grow overflow-y-auto font-mono text-[10px] space-y-1">
         <div v-for="(log, i) in logs" :key="i" class="border-l-2 pl-2" :class="{
          'border-blue-500 text-blue-200': log.type === 'info',
          'border-green-500 text-green-200': log.type === 'success',
          'border-red-500 text-red-200': log.type === 'error',
          'border-yellow-500 text-yellow-200': log.type === 'warn',
          'border-gray-500 text-gray-400': log.type === 'sys',
          'border-purple-500 text-purple-200': log.type === 'recv'
        }">
          <span class="opacity-50">[{{ log.time }}]</span> {{ log.msg }}
        </div>
       </div>
    </div>

    <!-- UI Panel -->
    <div class="w-full md:w-[380px] lg:w-[420px] shrink-0 p-4 md:p-6 flex flex-col gap-5 md:overflow-y-auto">
      <div class="text-center md:text-left space-y-1">
        <div class="inline-flex items-center justify-center w-12 h-12 rounded-full bg-blue-100 text-blue-600 mb-3 mx-auto md:mx-0 text-2xl font-bold">📍</div>
        <h1 class="text-2xl font-bold tracking-tight text-gray-900">群組定點尋人 (P2P)</h1>
        <p class="text-sm text-gray-500">同步中 ({{ nextSyncIn }}s) · 0 Server</p>
      </div>

      <div class="bg-white rounded-2xl shadow-sm border border-gray-100 p-4">
        <h2 class="text-xs font-bold text-gray-400 uppercase tracking-wider mb-2">👤 我的名稱</h2>
        <input v-model="myName" type="text" maxlength="12" class="w-full px-4 py-3 rounded-xl border border-gray-200 focus:ring-2 focus:ring-blue-500 outline-none transition-all text-lg font-medium">
      </div>

      <button @click="handleShare" :disabled="!myName" class="w-full bg-blue-600 hover:bg-blue-700 text-white rounded-2xl py-4 font-bold shadow-lg shadow-blue-100 active:scale-95 transition-transform flex items-center justify-center gap-2">
        <span>🔗</span> 複製分享網址
      </button>

      <div class="bg-white rounded-2xl shadow-sm border border-gray-100 overflow-hidden flex-grow min-h-[300px]">
        <div class="p-4 border-b border-gray-50 bg-gray-50/50 flex justify-between items-center">
          <h2 class="text-sm font-bold text-gray-700">📍 位置名單</h2>
          <span v-if="isLocating" class="w-2 h-2 bg-blue-500 rounded-full animate-ping"></span>
        </div>
        <div class="divide-y divide-gray-50 max-h-[500px] overflow-y-auto">
          <!-- ME -->
          <div class="p-4 flex items-center gap-4 bg-blue-50/30">
            <div class="w-12 h-12 rounded-full bg-blue-600 flex items-center justify-center text-white font-bold text-lg shadow-inner">{{ getInitials(myName) }}</div>
            <div class="min-w-0">
              <div class="font-bold text-gray-900 truncate">{{ myName || '未命名' }} <span class="ml-1 text-[10px] bg-blue-200 text-blue-700 px-1.5 py-0.5 rounded-full">YOU</span></div>
              <div class="text-[10px] text-gray-400 font-mono mt-0.5">ID: {{ myId }} · 最近: {{ formatTime(lastSyncAt) }}</div>
            </div>
          </div>
          <!-- OTHERS -->
          <div v-if="others.size === 0" class="p-10 text-center text-sm text-gray-400">
            <span class="block text-2xl mb-2">📭</span> 暫無其他夥伴加入...
          </div>
          <div v-for="[uid, data] in activeOthers" :key="uid" class="p-4 flex items-center gap-4 transition-opacity" :class="{'opacity-50 grayscale': !data.isOnline}">
            <div class="w-12 h-12 rounded-full flex items-center justify-center font-bold text-lg shadow-sm" :class="data.isOnline ? 'bg-gray-800 text-white' : 'bg-gray-200 text-gray-400'">
                {{ getInitials(data.name) }}
            </div>
            <div class="min-w-0 flex-grow">
              <div class="font-bold text-gray-900 truncate flex items-center gap-2">
                {{ data.name }}
                <span v-if="!data.isOnline" class="text-[9px] bg-gray-200 text-gray-500 px-1.5 py-0.5 rounded-full font-normal">離線</span>
              </div>
              <div class="text-[10px] text-gray-400 font-mono">ID: {{ uid }} · {{ formatTime(data.lastSeen) }}</div>
              <div class="flex gap-1 mt-1.5" v-if="data.isOnline">
                 <span v-for="strat in data.peerIds.keys()" :key="strat" class="px-1.5 py-0.5 rounded-md bg-gray-100 border border-gray-200 text-[8px] text-gray-500 font-bold uppercase">{{ strat }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Map Container -->
    <div class="w-full flex-grow min-h-[50vh] md:h-full bg-white rounded-t-3xl md:rounded-l-3xl md:rounded-tr-none border-t md:border-t-0 md:border-l border-gray-200 overflow-hidden shadow-2xl relative z-0">
       <div id="map" class="w-full h-full"></div>
    </div>

    <!-- Toast Notification -->
    <transition name="toast">
      <div v-if="toastVisible" class="fixed bottom-20 left-1/2 -translate-x-1/2 bg-gray-800 text-white px-6 py-3 rounded-full z-[5000] text-sm shadow-2xl">{{ toastMessage }}</div>
    </transition>
  </div>
</template>

<style>
@keyframes fade-in { from { opacity: 0; } to { opacity: 1; } }
.animate-fade-in { animation: fade-in 0.2s ease-out; }
.toast-enter-active, .toast-leave-active { transition: all 0.3s ease; }
.toast-enter-from, .toast-leave-to { opacity: 0; transform: translate(-50%, 20px); }
.leaflet-container { background-color: #f8fafc; cursor: crosshair !important; }

/* Custom Scrollbar */
::-webkit-scrollbar { width: 4px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb { background: #e2e8f0; border-radius: 10px; }
</style>
