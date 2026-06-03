<script setup>
import { ref, onMounted, onUnmounted, computed, watch, nextTick } from 'vue';
import L from 'leaflet';
import { joinRoom as joinNostr } from '@trystero-p2p/nostr';
import { joinRoom as joinMqtt } from '@trystero-p2p/mqtt';
import { joinRoom as joinTorrent } from '@trystero-p2p/torrent';

// --- 1. Identity & P2P Data Management ---
const myId = ref(''); // Persistent locator_uid (localStorage)
const myName = ref('');
const myLocation = ref(null);
const others = ref(new Map()); // uid -> {name, lat, lng, peerIds: Map<strat, peerId>}

// --- 2. Trystero P2P Setup ---
const STRATEGIES = [
  { name: 'nostr', join: joinNostr },
  { name: 'mqtt', join: joinMqtt },
  { name: 'torrent', join: joinTorrent }
];

const rooms = ref(new Map()); // stratName -> room
const locActions = ref(new Map()); // stratName -> action

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
      const room = strat.join({ appId: 'p2p-pin-locator-v1' }, roomId);
      const locAction = room.makeAction('loc');

      rooms.value.set(strat.name, room);
      locActions.value.set(strat.name, locAction);

      locAction.onMessage = (data, meta) => {
        if (!data.uid) return;
        const peerId = typeof meta === 'object' ? meta.peerId : meta;
        
        console.log(`[P2P Receive] From: ${data.uid} via ${strat.name} (${peerId})`);
        
        const existing = others.value.get(data.uid) || { ...data, peerIds: new Map() };
        existing.lat = data.lat;
        existing.lng = data.lng;
        existing.name = data.name;
        existing.peerIds.set(strat.name, peerId);
        
        others.value.set(data.uid, existing);
        updateMapMarkers();
      };

      room.onPeerJoin = (peerId) => {
        console.log(`[P2P Join] ${strat.name} connected: ${peerId}`);
        if (myLocation.value) {
          locAction.send({
            uid: myId.value,
            name: myName.value,
            lat: myLocation.value.lat,
            lng: myLocation.value.lng
          }, peerId);
        }
      };

      room.onPeerLeave = (peerId) => {
        console.log(`[P2P Leave] ${strat.name} disconnected: ${peerId}`);
        for (const [uid, data] of others.value.entries()) {
          if (data.peerIds.get(strat.name) === peerId) {
            data.peerIds.delete(strat.name);
            if (data.peerIds.size === 0) {
              others.value.delete(uid);
              if (markers.value[uid]) {
                mapInstance.value.removeLayer(markers.value[uid]);
                delete markers.value[uid];
              }
            }
            break;
          }
        }
      };
    } catch (e) {
      console.error(`Failed to init strategy ${strat.name}:`, e);
    }
  });
};

// --- 3. UI & State ---
const isLocating = ref(false);
const mapInstance = ref(null);
const markers = ref({}); // uid -> Marker instance
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
  return Array.from(others.value.entries()); // [uid, data]
});

const shortId = (id) => id ? id.replace('u_', '').slice(0, 4) : '....';

// --- 4. Main Location Logic ---
let syncInterval = null;
let timerInterval = null;

const updateLocation = (isManualClick = false) => {
  if (!navigator.geolocation) {
    if (isManualClick) showToast('您的瀏覽器不支援地理位置功能');
    return;
  }
  if (!myName.value.trim()) {
    if (isManualClick) showToast('請先設定您的名稱');
    return;
  }

  isLocating.value = true;
  navigator.geolocation.getCurrentPosition(
    (pos) => {
      isLocating.value = false;
      const { latitude: lat, longitude: lng } = pos.coords;
      myLocation.value = { lat, lng };

      // Broadcast through ALL active strategies
      const payload = { uid: myId.value, name: myName.value, lat, lng };
      locActions.value.forEach(action => action.send(payload));

      if (isManualClick) {
        navigator.clipboard.writeText(window.location.href).then(() => {
          showToast('📍 網址已複製！已透過多重管道同步。');
        });
      }

      updateMapMarkers();
      nextSyncIn.value = 10; 
    },
    (err) => {
      isLocating.value = false;
      if (isManualClick) showToast('無法取得位置，請確認定位權限已開啟');
    },
    { enableHighAccuracy: true, timeout: 10000, maximumAge: 0 }
  );
};

const startAutoSync = () => {
  if (syncInterval) clearInterval(syncInterval);
  if (timerInterval) clearInterval(timerInterval);
  updateLocation(false);
  syncInterval = setInterval(() => updateLocation(false), 10000);
  timerInterval = setInterval(() => {
    if (nextSyncIn.value > 0) nextSyncIn.value--;
  }, 1000);
};

const handleShare = () => {
  navigator.clipboard.writeText(window.location.href).then(() => {
    showToast('📍 網址已複製！分享網址即可即時尋人。');
  });
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

const createMarkerIcon = (name, isMe = false) => {
  return L.divIcon({
    className: 'bg-transparent border-none',
    html: `<div class="flex items-center justify-center w-8 h-8 rounded-full ${isMe ? 'bg-blue-600' : 'bg-gray-800'} text-white font-bold shadow-lg ring-2 ring-white text-xs">${getInitials(name)}</div>`,
    iconSize: [32, 32],
    iconAnchor: [16, 32],
    popupAnchor: [0, -32]
  });
};

const initMap = () => {
  fixLeafletIcon();
  const map = L.map('map', { zoomControl: false }).setView([23.58, 120.58], 15);
  L.control.zoom({ position: 'bottomright' }).addTo(map);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; OpenStreetMap'
  }).addTo(map);
  mapInstance.value = map;
};

const updateMapMarkers = () => {
  if (!mapInstance.value) return;
  const map = mapInstance.value;

  activeOthers.value.forEach(([uid, data]) => {
    if (!markers.value[uid]) {
      markers.value[uid] = L.marker([data.lat, data.lng], { icon: createMarkerIcon(data.name) })
        .bindPopup(`<div class="font-sans text-sm font-medium px-1">${data.name} <small class="text-gray-400">(${shortId(uid)})</small></div>`)
        .addTo(map);
    } else {
      markers.value[uid].setLatLng([data.lat, data.lng]);
    }
  });

  if (myLocation.value && myName.value) {
    if (!markers.value['me']) {
      markers.value['me'] = L.marker([myLocation.value.lat, myLocation.value.lng], { 
        icon: createMarkerIcon(myName.value, true),
        zIndexOffset: 1000 
      })
        .bindPopup(`<div class="font-sans text-sm font-medium px-1 text-blue-600">${myName.value} (You)</div>`)
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
  rooms.value.forEach(room => room.leave());
});
</script>

<template>
  <div class="h-full flex flex-col md:flex-row p-4 md:p-6 lg:p-8 gap-6 max-w-7xl mx-auto w-full">
    <div class="w-full md:w-[380px] lg:w-[420px] shrink-0 flex flex-col gap-5 md:overflow-y-auto pb-4">
      <div class="text-center md:text-left space-y-1 mb-2">
        <div class="inline-flex items-center justify-center w-12 h-12 rounded-full bg-blue-100 text-blue-600 mb-3 mx-auto md:mx-0 text-2xl font-bold">📍</div>
        <h1 class="text-2xl font-bold tracking-tight">群組定點尋人 (P2P)</h1>
        <p class="text-sm text-gray-500">多重管道同步中 (倒數：{{ nextSyncIn }}s) · 0 Server</p>
      </div>

      <div class="bg-white rounded-2xl shadow-sm border border-gray-100 overflow-hidden shrink-0">
        <div class="p-4 border-b border-gray-50 bg-gray-50/50">
          <h2 class="text-sm font-semibold flex items-center gap-2 text-gray-700">👤 我是誰 (可支援中文)</h2>
        </div>
        <div class="p-4">
          <input v-model="myName" type="text" maxlength="12" placeholder="請輸入您的名稱..." class="w-full px-4 py-3 rounded-xl border border-gray-200 focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none transition-all placeholder:text-gray-300 text-lg font-medium">
        </div>
      </div>

      <div class="grid gap-3 shrink-0">
        <button @click="handleShare" :disabled="!myName" class="w-full bg-blue-600 hover:bg-blue-700 text-white rounded-xl py-4 px-6 font-semibold shadow-sm shadow-blue-200 transition-all active:scale-[0.98] flex items-center justify-center gap-2 disabled:bg-blue-300 disabled:cursor-not-allowed">
          <span>🔗</span> 分享網址並複製
        </button>
      </div>

      <div class="bg-white rounded-2xl shadow-sm border border-gray-100 overflow-hidden min-h-[200px]">
        <div class="p-4 border-b border-gray-50 bg-gray-50/50 flex justify-between items-center">
          <h2 class="text-sm font-semibold flex items-center gap-2 text-gray-700">📍 即時位置名單</h2>
          <span v-if="isLocating" class="text-xs text-blue-500 animate-pulse font-medium">更新中...</span>
        </div>
        <div class="divide-y divide-gray-50">
          <div class="p-4 flex items-center justify-between hover:bg-gray-50/50 transition-colors">
            <div class="flex items-center gap-3">
              <div class="w-10 h-10 shrink-0 rounded-full bg-blue-100 flex items-center justify-center text-blue-600 font-bold text-sm">{{ getInitials(myName) }}</div>
              <div class="min-w-0">
                <div class="font-medium text-gray-900 flex items-center gap-2 truncate">
                  {{ myName || '未設定名稱' }}
                  <span class="px-2 py-0.5 rounded text-[10px] font-bold bg-gray-100 text-gray-500 shrink-0">YOU</span>
                </div>
                <div class="text-[10px] text-gray-400 font-mono -mt-1 mb-1">ID: {{ myId }}</div>
                <div class="text-xs text-gray-500">{{ myLocation ? `${myLocation.lat.toFixed(5)}, ${myLocation.lng.toFixed(5)}` : '尚未取得定位' }}</div>
              </div>
            </div>
          </div>

          <div v-if="activeOthers.length === 0" class="p-6 text-center text-sm text-gray-400">
            <span class="block mb-1">📭</span> 等待其他人加入房間...
          </div>
          <div v-for="[uid, data] in activeOthers" :key="uid" class="p-4 flex items-center justify-between hover:bg-gray-50/50 transition-colors">
            <div class="flex items-center gap-3">
              <div class="w-10 h-10 shrink-0 rounded-full bg-gray-100 flex items-center justify-center text-gray-600 font-bold text-sm">{{ getInitials(data.name) }}</div>
              <div class="min-w-0">
                <div class="font-medium text-gray-900 truncate">{{ data.name }}</div>
                <div class="text-[10px] text-gray-400 font-mono -mt-1 mb-1">ID: {{ uid }}</div>
                <div class="text-xs text-gray-500">{{ data.lat.toFixed(5) }}, {{ data.lng.toFixed(5) }}</div>
                <div class="flex gap-1 mt-1">
                   <span v-for="strat in data.peerIds.keys()" :key="strat" class="px-1 py-0.5 rounded bg-gray-100 border border-gray-200 text-[8px] text-gray-500 uppercase">{{ strat }}</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="w-full flex-grow min-h-[50vh] md:h-full bg-white rounded-2xl border border-gray-200 overflow-hidden relative z-0 shadow-sm">
      <div id="map" class="w-full h-full"></div>
    </div>

    <transition name="toast">
      <div v-if="toastVisible" class="fixed bottom-10 left-1/2 -translate-x-1/2 bg-gray-800 text-white px-6 py-3 rounded-full shadow-lg z-50 text-sm font-medium whitespace-nowrap">{{ toastMessage }}</div>
    </transition>
  </div>
</template>

<style>
.toast-enter-active, .toast-leave-active { transition: all 0.3s ease; }
.toast-enter-from, .toast-leave-to { opacity: 0; transform: translate(-50%, 20px); }
.leaflet-container { background-color: #f8fafc; }
</style>
