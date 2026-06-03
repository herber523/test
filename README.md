# P2P 即時定點尋人 (Vite + Vue 3 版)

這是一個基於 P2P 技術的即時位置分享工具。不需經過任何後端伺服器，直接在瀏覽器之間透過 WebRTC 傳遞座標。

## 特色
- **真正的即時**：座標透過 Trystero (WebRTC) 傳輸，反應極快。
- **隱私安全**：資料不落地伺服器，端到端加密。
- **無縫分享**：複製帶有 `?room=...` 的網址即可讓其他人加入同一房間。
- **現代架構**：使用 Vue 3 SFC、Vite 與 Tailwind CSS。

## 開發與建置

### 安裝依賴
```bash
npm install
```

### 啟動開發環境
```bash
npm run dev
```

### 產出正式環境檔案 (Build)
```bash
npm run build
```
產出的檔案將位於 `dist/` 目錄，可直接部署至 GitHub Pages、Vercel 或 Netlify。

## 技術棧
- **UI**: Vue 3 (Composition API)
- **Map**: Leaflet
- **P2P**: Trystero (@trystero-p2p/nostr)
- **Styling**: Tailwind CSS
- **Bundler**: Vite

## 注意事項
- **Secure Context**: 由於瀏覽器安全性限制，Geolocation API 與 WebRTC (P2P) 需要在 `localhost` 或 `https` 協議下才能正常運作。
- **打洞技術**: 在複雜的企業網路下可能需要 STUN/TURN 伺服器支援。
