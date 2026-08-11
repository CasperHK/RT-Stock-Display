# 📊 Solid.js Full-Screen Real-Time Stock Dashboard

這是一個專為全螢幕、大顯示器、電視牆或副螢幕設計的**極簡黑系、高更新率即時股市看板**。本專案利用 **Solid.js** 強大的細粒度更新（Fine-grained reactivity）特性，確保在處理每秒數十次的 WebSocket 高頻 Tick 數據時，介面依然流暢不卡頓、無不必要的 DOM 重新渲染。

---

## ⚡ 核心亮點

*   **極低效能開銷**：利用 Solid.js 的 `createSignal` / `createStore` 精確更新特定文字與樣式，免去虛擬 DOM (VVDOM) 的比對耗能。
*   **全螢幕優化設計**：預設為 Cyberpunk 暗黑風格高對比介面。支援 1080p、2K、4K 響應式佈局，並具備無邊框全螢幕鎖定功能。
*   **實時資料串流**：內建 WebSocket 動態用戶端，支援流暢對接 Finnhub、Twelve Data 或 Alpha Vantage 等主流數據源。
*   **視覺化趨勢**：每檔個股區塊均整合輕量化 Canvas 迷你趨勢圖（Mini-Sparkline），即時動態呈現當日價格走勢。
*   **閃爍視覺反饋**：當價格跳動時，文字欄位會根據漲跌立即觸發「綠閃（Up）」或「紅閃（Down）」動畫。

---

## 🏗️ 專案架構建議

推薦搭配 **Vite** 與 **Tailwind CSS** 以達到最佳開發體驗：

```text
├── src/
│   ├── assets/          # 靜態資源 (字體、圖標)
│   ├── components/      # 組件庫
│   │   ├── StockGrid.tsx     # 多股走勢網格
│   │   ├── StockCard.tsx     # 單股即時卡片
│   │   └── Sparkline.tsx     # 輕量級即時微型圖表
│   ├── services/
│   │   └── websocket.ts      # WebSocket 連線管理與數據分發
│   ├── App.tsx          # 根組件與全螢幕控制器
│   └── index.tsx        # 進入點
├── tailwind.config.js
└── package.json
```

---

## 🚀 核心程式碼實作參考

### 1. 響應式資料狀態 (Store) 與 WebSocket 整合
利用 Solid.js 的 `createStore` 來管理多檔股票的即時狀態，高效處理深層物件更新：

```typescript
// src/services/websocket.ts
import { createStore } from "solid-js/store";

export interface StockTick {
  price: number;
  change: number;
  changePercent: number;
  history: number[];
  direction: "up" | "down" | "none";
}

// 初始化訂閱列表
export const [stocks, setStocks] = createStore<Record<string, StockTick>>({
  AAPL: { price: 0, change: 0, changePercent: 0, history: [], direction: "none" },
  NVDA: { price: 0, change: 0, changePercent: 0, history: [], direction: "none" },
  TSLA: { price: 0, change: 0, changePercent: 0, history: [], direction: "none" },
});

export function connectStockWS(apiKey: string) {
  const ws = new WebSocket(`wss://ws.finnhub.io?token=${apiKey}`);

  ws.onopen = () => {
    // 訂閱所需代號
    Object.keys(stocks).forEach(symbol => {
      ws.send(JSON.stringify({ type: "subscribe", symbol }));
    });
  };

  ws.onmessage = (event) => {
    const response = JSON.parse(event.data);
    if (response.type === "trade") {
      const data = response.data[0]; // 取得最新一筆交易
      const symbol = data.s;
      const newPrice = data.p;

      setStocks(symbol, (prev) => {
        const nextDirection = newPrice > prev.price ? "up" : newPrice < prev.price ? "down" : "none";
        // 限制歷史紀錄長度以優化記憶體，供 Sparkline 繪圖使用
        const nextHistory = [...prev.history, newPrice].slice(-30); 
        
        return {
          price: newPrice,
          direction: nextDirection,
          history: nextHistory
        };
      });
    }
  };
}
```

### 2. 極速渲染組件 (Component)
透過細粒度響應，當 `stocks.AAPL.price` 改變時，只有該 `<span>` 的文字與 class 會變動：

```tsx
// src/components/StockCard.tsx
import { Component, useEffect } from "solid-js";
import { StockTick } from "../services/websocket";
import Sparkline from "./Sparkline";

interface Props {
  symbol: string;
  data: StockTick;
}

const StockCard: Component<Props> = (props) => {
  return (
    <div class="bg-gray-900 border border-gray-800 p-6 rounded-xl flex flex-col justify-between h-full">
      <div>
        <div class="text-gray-400 text-sm font-semibold tracking-wider">{props.symbol}</div>
        <div 
          class="text-4xl font-mono font-bold mt-2 transition-colors duration-150"
          class={{
            "text-green-400": props.data.direction === "up",
            "text-red-400": props.data.direction === "down",
            "text-white": props.data.direction === "none"
          }}
        >
          \${props.data.price.toFixed(2)}
        </div>
      </div>
      
      {/* 傳遞歷史陣列給 Canvas 微型圖表組件 */}
      <div class="h-16 mt-4">
        <Sparkline data={props.data.history} direction={props.data.direction} />
      </div>
    </div>
  );
};

export default StockCard;
```

---

## 🛠️ 開發與建置步驟

### 1. 複製專案與安裝依賴
```bash
git clone https://github.com
cd solid-stock-display
pnpm install
```

### 2. 設定環境變數
在專案根目錄建立 `.env` 檔案：
```env
VITE_STOCK_API_KEY=您的_API_KEY
```

### 3. 開發環境啟動
```bash
pnpm run dev
```

### 4. 編譯生產環境版本
```bash
pnpm run build
```

---

## 📺 最佳化全螢幕投放指南

為了讓此畫面在副螢幕、壁掛電視上完美呈現，建議進行以下設定：

1.  **進入瀏覽器全螢幕**：在 Windows 上按下 `F11`，或在 macOS 上按下 `Cmd + Ctrl + F` 隱藏所有工具列。
2.  **停用螢幕休眠**：建議將投放主機（如 Mac mini、Raspberry Pi 或 NUC）的系統休眠時間設為「從不休眠」。
3.  **瀏覽器記憶體優化**：本專案的圖表陣列已加上最大長度限制 (`.slice(-30)`)，可保障系統連續運行數週不發生記憶體流失（Memory Leak）。

---

## 📝 授權條款
本專案採用 [MIT License](LICENSE) 授權條款。
