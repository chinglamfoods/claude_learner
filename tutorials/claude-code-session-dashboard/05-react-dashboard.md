# 05：React 儀表板前端

> 使用 React + TypeScript + Vite 建構儀表板 UI，搭配 Chart.js 呈現 token 用量圖表，透過 WebSocket 接收即時更新。

## 前端技術堆疊

| 工具 | 用途 |
|------|------|
| React 18+ | UI 元件框架 |
| TypeScript | 型別安全 |
| Vite | 建構與開發伺服器 |
| Tailwind CSS | Utility-first 樣式 |
| Chart.js + react-chartjs-2 | Token 用量時間序列圖表 |

## 專案初始化

```bash
npm create vite@latest web -- --template react-ts
cd web
npm install chart.js react-chartjs-2 tailwindcss @tailwindcss/vite
```

## 型別定義

```tsx
// src/types.ts
export interface Session {
  id: string
  projectPath: string
  startedAt: string
  lastActiveAt: string
  isActive: boolean
  totalInputTokens: number
  totalOutputTokens: number
  estimatedCostUSD: number
  fileChanges: number
  messageCount: number
}

export interface TokenSnapshot {
  timestamp: string
  inputTokens: number
  outputTokens: number
  cumulativeCost: number
}

export interface WSMessage {
  type: 'session_update' | 'sessions_snapshot' | 'session_removed'
  payload: Session | Session[] | string
}
```

## WebSocket Hook

封裝 WebSocket 連線管理為 custom hook，處理自動重連與訊息派發：

```tsx
// src/hooks/useWebSocket.ts
import { useEffect, useRef, useCallback, useState } from 'react'
import type { WSMessage } from '../types'

export function useWebSocket(url: string) {
  const wsRef = useRef<WebSocket | null>(null)
  const [isConnected, setIsConnected] = useState(false)
  const [lastMessage, setLastMessage] = useState<WSMessage | null>(null)
  const reconnectTimeoutRef = useRef<ReturnType<typeof setTimeout>>()

  const connect = useCallback(() => {
    const ws = new WebSocket(url)

    ws.onopen = () => {
      setIsConnected(true)
    }

    ws.onmessage = (event) => {
      try {
        // 後端可能在一個 frame 中批次發送多條訊息，以換行分隔
        const messages = event.data.split('\n')
        for (const msg of messages) {
          if (msg.trim()) {
            setLastMessage(JSON.parse(msg))
          }
        }
      } catch {
        // 忽略格式錯誤的訊息
      }
    }

    ws.onclose = () => {
      setIsConnected(false)
      // 斷線後 3 秒自動重連
      reconnectTimeoutRef.current = setTimeout(connect, 3000)
    }

    ws.onerror = () => {
      ws.close()
    }

    wsRef.current = ws
  }, [url])

  useEffect(() => {
    connect()
    return () => {
      clearTimeout(reconnectTimeoutRef.current)
      wsRef.current?.close()
    }
  }, [connect])

  return { isConnected, lastMessage }
}
```

## 主應用程式元件

```tsx
// src/App.tsx
import { useEffect, useState } from 'react'
import { useWebSocket } from './hooks/useWebSocket'
import { SessionCard } from './components/SessionCard'
import { ConnectionStatus } from './components/ConnectionStatus'
import type { Session, WSMessage } from './types'

const WS_URL = `ws://${window.location.host}/ws`

export default function App() {
  const [sessions, setSessions] = useState<Map<string, Session>>(new Map())
  const { isConnected, lastMessage } = useWebSocket(WS_URL)

  // 處理 WebSocket 訊息
  useEffect(() => {
    if (!lastMessage) return

    switch (lastMessage.type) {
      case 'sessions_snapshot': {
        // 完整快照：替換所有 session
        const snapshot = lastMessage.payload as Session[]
        const map = new Map<string, Session>()
        snapshot.forEach((s) => map.set(s.id, s))
        setSessions(map)
        break
      }
      case 'session_update': {
        // 增量更新：更新單一 session
        const session = lastMessage.payload as Session
        setSessions((prev) => new Map(prev).set(session.id, session))
        break
      }
      case 'session_removed': {
        const sessionId = lastMessage.payload as string
        setSessions((prev) => {
          const next = new Map(prev)
          next.delete(sessionId)
          return next
        })
        break
      }
    }
  }, [lastMessage])

  // 依活躍狀態與最後活動時間排序
  const sortedSessions = Array.from(sessions.values()).sort((a, b) => {
    if (a.isActive !== b.isActive) return a.isActive ? -1 : 1
    return new Date(b.lastActiveAt).getTime() - new Date(a.lastActiveAt).getTime()
  })

  return (
    <div className="min-h-screen bg-gray-950 text-white p-6">
      <header className="flex items-center justify-between mb-8">
        <h1 className="text-2xl font-bold">Claude Sessions</h1>
        <ConnectionStatus connected={isConnected} />
      </header>

      <div className="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-4">
        {sortedSessions.map((session) => (
          <SessionCard key={session.id} session={session} />
        ))}
      </div>

      {sessions.size === 0 && (
        <p className="text-gray-500 text-center mt-12">
          尚未偵測到任何 Claude session
        </p>
      )}
    </div>
  )
}
```

## Session 卡片元件

```tsx
// src/components/SessionCard.tsx
import type { Session } from '../types'

interface Props {
  session: Session
}

export function SessionCard({ session }: Props) {
  const lastActive = new Date(session.lastActiveAt)
  const duration = session.startedAt
    ? formatDuration(new Date(session.startedAt), lastActive)
    : '—'

  return (
    <div
      className={`rounded-lg border p-4 ${
        session.isActive
          ? 'border-green-500/50 bg-green-500/5'
          : 'border-gray-800 bg-gray-900'
      }`}
    >
      {/* 狀態指示燈 */}
      <div className="flex items-center gap-2 mb-3">
        <span
          className={`h-2 w-2 rounded-full ${
            session.isActive ? 'bg-green-400 animate-pulse' : 'bg-gray-600'
          }`}
        />
        <span className="text-sm text-gray-400 truncate">
          {session.projectPath}
        </span>
      </div>

      {/* 指標 */}
      <div className="grid grid-cols-2 gap-3 text-sm">
        <Metric label="成本" value={`$${session.estimatedCostUSD.toFixed(2)}`} />
        <Metric label="持續時間" value={duration} />
        <Metric
          label="Input tokens"
          value={formatTokens(session.totalInputTokens)}
        />
        <Metric
          label="Output tokens"
          value={formatTokens(session.totalOutputTokens)}
        />
        <Metric label="訊息數" value={session.messageCount.toString()} />
        <Metric label="檔案變更" value={session.fileChanges.toString()} />
      </div>
    </div>
  )
}

function Metric({ label, value }: { label: string; value: string }) {
  return (
    <div>
      <p className="text-gray-500 text-xs">{label}</p>
      <p className="text-white font-mono">{value}</p>
    </div>
  )
}

function formatTokens(n: number): string {
  if (n >= 1_000_000) return `${(n / 1_000_000).toFixed(1)}M`
  if (n >= 1_000) return `${(n / 1_000).toFixed(1)}K`
  return n.toString()
}

function formatDuration(start: Date, end: Date): string {
  const diffMs = end.getTime() - start.getTime()
  const hours = Math.floor(diffMs / 3_600_000)
  const minutes = Math.floor((diffMs % 3_600_000) / 60_000)
  if (hours > 0) return `${hours}h ${minutes}m`
  return `${minutes}m`
}
```

## Token 用量圖表

```tsx
// src/components/TokenChart.tsx
import { useEffect, useState } from 'react'
import { Line } from 'react-chartjs-2'
import {
  Chart as ChartJS,
  LineElement,
  PointElement,
  LinearScale,
  TimeScale,
  Tooltip,
  Legend,
  Filler,
} from 'chart.js'
import 'chartjs-adapter-date-fns'
import type { TokenSnapshot } from '../types'

ChartJS.register(
  LineElement, PointElement, LinearScale,
  TimeScale, Tooltip, Legend, Filler
)

interface Props {
  sessionId: string
}

export function TokenChart({ sessionId }: Props) {
  const [snapshots, setSnapshots] = useState<TokenSnapshot[]>([])

  useEffect(() => {
    fetch(`/api/sessions/${sessionId}/metrics`)
      .then((r) => r.json())
      .then(setSnapshots)
      .catch(() => {})
  }, [sessionId])

  if (snapshots.length < 2) return null

  const data = {
    labels: snapshots.map((s) => new Date(s.timestamp)),
    datasets: [
      {
        label: '累計成本 (USD)',
        data: snapshots.map((s) => s.cumulativeCost),
        borderColor: 'rgb(34, 197, 94)',
        backgroundColor: 'rgba(34, 197, 94, 0.1)',
        fill: true,
        tension: 0.3,
      },
    ],
  }

  const options = {
    responsive: true,
    scales: {
      x: {
        type: 'time' as const,
        time: { unit: 'hour' as const },
        grid: { color: 'rgba(255,255,255,0.05)' },
        ticks: { color: '#6b7280' },
      },
      y: {
        grid: { color: 'rgba(255,255,255,0.05)' },
        ticks: {
          color: '#6b7280',
          callback: (value: number) => `$${value.toFixed(0)}`,
        },
      },
    },
    plugins: {
      legend: { display: false },
    },
  }

  return (
    <div className="mt-4 p-3 bg-gray-900 rounded-lg">
      <Line data={data} options={options} />
    </div>
  )
}
```

## 連線狀態指示器

```tsx
// src/components/ConnectionStatus.tsx
interface Props {
  connected: boolean
}

export function ConnectionStatus({ connected }: Props) {
  return (
    <div className="flex items-center gap-2 text-sm">
      <span
        className={`h-2 w-2 rounded-full ${
          connected ? 'bg-green-400' : 'bg-red-400 animate-pulse'
        }`}
      />
      <span className={connected ? 'text-green-400' : 'text-red-400'}>
        {connected ? '已連線' : '連線中斷...'}
      </span>
    </div>
  )
}
```

## 建構與整合

前端建構後的靜態檔案由 Go 後端的 Gin 伺服器直接提供：

```bash
cd web
npm run build
# 產出在 web/dist/ 目錄，由 Go 後端的 r.Static() 提供服務
```

這樣整個應用只需要一個服務程序——Go 後端同時處理 API、WebSocket 與前端靜態檔案。

## 重點整理

- WebSocket hook 封裝連線管理與自動重連，元件只需消費 `lastMessage`
- 使用 `Map<string, Session>` 管理 session 狀態，支援 O(1) 的更新與刪除
- 活躍的 session 排在前面並以綠色邊框標示，提供視覺區分
- Chart.js 搭配 time scale 呈現 token 用量的時間序列圖表
- 前端建構後由 Go 後端提供靜態檔案，簡化部署為單一服務

---

下一篇：[Docker 部署與實務洞察](./06-docker-deployment.md)
