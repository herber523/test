<script setup>
import { ref, onMounted, onUnmounted, computed, watch, nextTick } from 'vue';
import L from 'leaflet';
import { joinRoom as joinNostr } from '@trystero-p2p/nostr';
import { joinRoom as joinMqtt } from '@trystero-p2p/mqtt';
import { joinRoom as joinTorrent } from '@trystero-p2p/torrent';

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

  STRATEGIES.forEach(strat => {
    try {
      const room = strat.join({ appId: 'p2p-pin-locator-v1', rtcConfig: RTC_CONFIG }, roomId);
      const locAction = room.makeAction('loc');
      rooms.value.set(strat.name, room);
      locActions.value.set(strat.name, locAction);

      locAction.onMessage = (data, meta) => {
        if (!data.uid) return;
        const peerId = typeof meta === 'object' ? meta.peerId : meta;
        console.log(`[P2P Receive] ${data.uid} via ${strat.name}`);
        
        const now = Date.now();
        const existing = others.value.get(data.uid) || { peerIds: new Map() };
        
        others.value.set(data.uid, {
          ...existing,
          ...data,
          lastSeen: now,
          isOnline: true,
          peerId // Store for reference
        });
        
        others.value.get(data.uid).peerIds.set(strat.name, peerId);
        updateMapMarkers();
      };

      room.onPeerJoin = (peerId) => {
        console.log(`[P2P Join] ${strat.name} found: ${peerId}`);
        if (myLocation.value) {
          locAction.send({ uid: myId.value, name: myName.value, lat: myLocation.value.lat, lng: myLocation.value.lng }, peerId);
        }
      };

      room.onPeerLeave = (peerId) => {
        console.log(`[P2P Leave] ${strat.name} lost: ${peerId}`);
        for (const [uid, data] of others.value.entries()) {
          if (data.peerIds.get(strat.name) === peerId) {
            data.peerIds.delete(strat.name);
            if (data.peerIds.size === 0) {
              data.isOnline = false;
              // We DON'T delete the record anymore. Keep it as "Offline".
            }
            break;
          }
        }
        updateMapMarkers();
      };
    } catch (e) {
      console.error(`Failed strategy ${strat.name}:`, e);
    }
  });
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
  // Sort: online first, then by lastSeen desc
  return Array.from(others.value.entries()).sort((a, b) => {
    if (a[1].isOnline !== b[1].isOnline) return b[1].isOnline ? 1 : -1;
    return b[1].lastSeen - a[1].lastSeen;
  });
});

const shortId = (id) => id ? id.replace('u_', '').slice(0, 4) : '....';
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
  if (!navigator.geolocation) return isManualClick && showToast('不支援定位');
  if (!myName.value.trim()) return isManualClick && showToast('請先設名稱');

  isLocating.value = true;
  navigator.geolocation.getCurrentPosition(
    (pos) => {
      isLocating.value = false;
      const { latitude: lat, longitude: lng } = pos.coords;
      myLocation.value = { lat, lng };
      lastSyncAt.value = Date.now();

      const payload = { uid: myId.value, name: myName.value, lat, lng };
      locActions.value.forEach(action => action.send(payload));

      if (isManualClick) showToast('📍 位置已更新！');
      updateMapMarkers();
      nextSyncIn.value = 10; 
    },
    (err) => {
      isLocating.value = false;
      if (isManualClick) showToast('定位失敗，請檢查權限');
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

const getInitials = (name) => (name || '?').charAt(0).toUpperCase();

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

  // Others
  activeOthers.value.forEach(([uid, data]) => {
    if (!markers.value[uid]) {
      markers.value[uid] = L.marker([data.lat, data.lng], { icon: createMarkerIcon(data.name, false, data.isOnline) })
        .bindPopup(`<div class="font-sans text-sm font-medium px-1">${data.name} <small class="text-gray-400">(${shortId(uid)})</small><br/><span class="text-[10px] text-gray-500">最後更新: ${formatTime(data.lastSeen)}</span></div>`)
        .addTo(map);
    } else {
      markers.value[uid].setLatLng([data.lat, data.lng]);
      markers.value[uid].setIcon(createMarkerIcon(data.name, false, data.isOnline));
    }
  });

  // Self
  if (myLocation.value && myName.value) {
    if (!markers.value['me']) {
      markers.value['me'] = L.marker([myLocation.value.lat, myLocation.value.lng], { icon: createMarkerIcon(myName.value, true), zIndexOffset: 1000 })
        .bindPopup(`<div class="font-sans text-sm font-medium px-1 text-blue-600">${myName.value} (You)<br/><span class="text-[10px] text-gray-500">最後更新: ${formatTime(lastSyncAt.value)}</span></div>`)
        .addTo(map);
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
    locActions.value.forEach(action => action.send({ uid: myId.value, name: newName.trim(), ...myLocation.value }));
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
  <div class="h-full flex flex-col md:flex-row p-4 md:p-6 lg:p-8 gap-6 max-w-7xl mx-auto w-full">
    <div class="w-full md:w-[380px] lg:w-[420px] shrink-0 flex flex-col gap-5 md:overflow-y-auto pb-4">
      <div class="text-center md:text-left space-y-1 mb-2">
        <div class="inline-flex items-center justify-center w-12 h-12 rounded-full bg-blue-100 text-blue-600 mb-3 mx-auto md:mx-0 text-2xl font-bold">📍</div>
        <h1 class="text-2xl font-bold tracking-tight">群組定點尋人 (P2P)</h1>
        <p class="text-sm text-gray-500">多路徑同步 (倒數：{{ nextSyncIn }}s) · 0 Server</p>
      </div>

      <div class="bg-white rounded-2xl shadow-sm border border-gray-100 p-4">
        <h2 class="text-sm font-semibold mb-2 text-gray-700">👤 我是誰</h2>
        <input v-model="myName" type="text" maxlength="12" class="w-full px-4 py-2 rounded-xl border border-gray-200 outline-none">
      </div>

      <button @click="handleShare" :disabled="!myName" class="w-full bg-blue-600 hover:bg-blue-700 text-white rounded-xl py-4 font-semibold shadow-sm">🔗 分享網址並複製</button>

      <div class="bg-white rounded-2xl shadow-sm border border-gray-100 overflow-hidden min-h-[200px]">
        <div class="p-4 border-b border-gray-50 bg-gray-50/50">
          <h2 class="text-sm font-semibold text-gray-700">📍 位置名單</h2>
        </div>
        <div class="divide-y divide-gray-50">
          <!-- ME -->
          <div class="p-4 flex items-center gap-3 bg-blue-50/30">
            <div class="w-10 h-10 rounded-full bg-blue-100 flex items-center justify-center text-blue-600 font-bold text-sm">{{ getInitials(myName) }}</div>
            <div class="min-w-0">
              <div class="font-medium text-gray-900 truncate">{{ myName }} <span class="text-[10px] bg-blue-100 text-blue-600 px-1 rounded">YOU</span></div>
              <div class="text-[10px] text-gray-400 font-mono">ID: {{ myId }} · 最近: {{ formatTime(lastSyncAt) }}</div>
            </div>
          </div>
          <!-- OTHERS -->
          <div v-if="activeOthers.length === 0" class="p-6 text-center text-sm text-gray-400">📭 等待其他人加入...</div>
          <div v-for="[uid, data] in activeOthers" :key="uid" class="p-4" :class="{'opacity-60': !data.isOnline}">
            <div class="flex items-center gap-3">
              <div class="w-10 h-10 rounded-full flex items-center justify-center font-bold text-sm" :class="data.isOnline ? 'bg-gray-100 text-gray-600' : 'bg-gray-200 text-gray-400'">
                {{ getInitials(data.name) }}
              </div>
              <div class="min-w-0 flex-grow">
                <div class="font-medium text-gray-900 truncate flex items-center gap-2">
                  {{ data.name }}
                  <span v-if="!data.isOnline" class="text-[9px] bg-gray-200 text-gray-500 px-1 rounded font-normal">離線</span>
                </div>
                <div class="text-[10px] text-gray-400 font-mono">ID: {{ uid }} · 最近: {{ formatTime(data.lastSeen) }}</div>
                <div class="flex gap-1 mt-1" v-if="data.isOnline">
                   <span v-for="strat in data.peerIds.keys()" :key="strat" class="px-1 py-0.5 rounded bg-gray-100 text-[8px] text-gray-500 uppercase">{{ strat }}</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="w-full flex-grow min-h-[50vh] md:h-full bg-white rounded-2xl border border-gray-200 overflow-hidden shadow-sm relative">
       <div id="map" class="w-full h-full"></div>
       <div class="absolute top-4 right-4 z-[1000] bg-white/90 backdrop-blur px-3 py-1.5 rounded-full shadow-sm border border-gray-100 text-[10px] font-medium text-gray-500">
         最後同步：{{ formatTime(lastSyncAt) }}
       </div>
    </div>

    <transition name="toast">
      <div v-if="toastVisible" class="fixed bottom-10 left-1/2 -translate-x-1/2 bg-gray-800 text-white px-6 py-3 rounded-full z-50 text-sm shadow-xl">{{ toastMessage }}</div>
    </transition>
  </div>
</template>

<style>
.toast-enter-active, .toast-leave-active { transition: all 0.3s ease; }
.toast-enter-from, .toast-leave-to { opacity: 0; transform: translate(-50%, 20px); }
.leaflet-container { background-color: #f8fafc; }
</style>
