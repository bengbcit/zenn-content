---
title: "ビギナー向け:Raspberry Pi + Claude API統合ガイド　～データ保存と可視化ダッシュボード～ （下)"
emoji: "🍓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Raspberry Pi", "Claude", "Python", "IoT", "API"]
published: true
published_at: "2026-05-02 12:00" 
---

# データ保存＋可視化ダッシュボード：IoT システム完成

第1回でセンサーから数値を読み込み、第2回で AI が分析をしてくれるようになりました。

今回は、その結果を **データベース**に保存して、**Web ダッシュボード**でリアルタイム表示します。これで、単なる「今この瞬間の値」ではなく、**時系列データ**として環境の変化を追跡できるようになります。

## 🎯 この記事で実現すること

```
センサー読み込み（第1回）
    ↓
AI 分析（第2回）
    ↓
Supabase に保存 ← ← ← ここから！
    ↓
Web ダッシュボードに表示
    ↓
スマートフォンやPCからリアルタイム閲覧
```

### 📋 前提条件

- 第1回、第2回の実装が完了
- `sensor_reader.py` と `claude_analyzer.py` が動作
- インターネット接続

---

## 💾 ステップ1: Supabase でデータベースを作成

### Supabase アカウント作成

1. **https://supabase.com にアクセス**

2. **「Sign Up」で無料アカウント作成**  
   GitHub アカウントで簡単にできます

3. **新しいプロジェクトを作成:**
   - 名前: `iot-project`
   - リージョン: `Tokyo` （日本）
   - パスワード: メモしておく

4. **プロジェクトが起動するまで待つ** （2～3分）

### SQL でテーブルを作成

Supabase の左メニューから「SQL Editor」を開き、以下を実行：

```sql
-- テーブル1: sensor_readings（センサー読取）
CREATE TABLE sensor_readings (
    id BIGSERIAL PRIMARY KEY,
    timestamp TIMESTAMP NOT NULL,
    sensor_id VARCHAR(100),
    location VARCHAR(100),
    temperature FLOAT NOT NULL,
    humidity FLOAT NOT NULL,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW()
);

-- テーブル2: analyses（分析結果）
CREATE TABLE analyses (
    id BIGSERIAL PRIMARY KEY,
    reading_id BIGINT REFERENCES sensor_readings(id),
    environment_status VARCHAR(50),
    analysis_text TEXT,
    recommendations JSONB,
    confidence FLOAT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 検索を高速化するインデックス
CREATE INDEX idx_readings_timestamp ON sensor_readings(timestamp DESC);
CREATE INDEX idx_analyses_reading_id ON analyses(reading_id);
```

### API 接続情報を取得

1. **左メニューから「Settings」→「API」を選択**

2. **以下をコピー（環境変数に設定）:**
   - `Project URL`
   - `anon public` キー

Raspberry Pi で：

```bash
nano ~/.bashrc

# 以下を追加（値をあなたのものに置き換え）
export SUPABASE_URL="https://xxxxx.supabase.co"
export SUPABASE_ANON_KEY="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

source ~/.bashrc
```

---

## 📦 ステップ2: Python でデータベースに保存

### ライブラリをインストール

```bash
pip3 install supabase-py python-dotenv
```

### database_manager.py を作成

```bash
nano ~/database_manager.py
```

以下をコピー & ペースト：

```python
#!/usr/bin/env python3
# database_manager.py - Supabase へのデータ保存

import os
import json
from datetime import datetime
from supabase import create_client, Client

# 環境変数から Supabase 認証情報を取得
SUPABASE_URL = os.getenv("SUPABASE_URL")
SUPABASE_ANON_KEY = os.getenv("SUPABASE_ANON_KEY")

if not SUPABASE_URL or not SUPABASE_ANON_KEY:
    raise ValueError("❌ SUPABASE_URL と SUPABASE_ANON_KEY を設定してください")

# Supabase クライアントを初期化
supabase: Client = create_client(SUPABASE_URL, SUPABASE_ANON_KEY)

def save_sensor_reading(sensor_data):
    """
    センサーデータを sensor_readings テーブルに保存
    
    Args:
        sensor_data (dict): sensor_reader.py からの出力
    
    Returns:
        int: 保存されたレコードの ID
    """
    
    try:
        response = supabase.table("sensor_readings").insert({
            "timestamp": sensor_data["timestamp"],
            "sensor_id": sensor_data["sensor_id"],
            "location": sensor_data["location"],
            "temperature": sensor_data["temperature"]["value"],
            "humidity": sensor_data["humidity"]["value"],
            "status": sensor_data["status"]
        }).execute()
        
        record_id = response.data[0]["id"]
        print(f"✅ センサーデータを保存 (ID: {record_id})")
        return record_id
    
    except Exception as e:
        print(f"❌ 保存エラー: {e}")
        return None

def save_analysis(reading_id, analysis_data):
    """
    分析結果を analyses テーブルに保存
    
    Args:
        reading_id (int): sensor_readings のレコード ID
        analysis_data (dict): claude_analyzer.py からの出力
    """
    
    try:
        response = supabase.table("analyses").insert({
            "reading_id": reading_id,
            "environment_status": analysis_data.get("comfort_level"),
            "analysis_text": analysis_data.get("analysis"),
            "recommendations": json.dumps(
                analysis_data.get("recommendations", []),
                ensure_ascii=False
            ),
            "confidence": analysis_data.get("confidence", 0)
        }).execute()
        
        print(f"✅ 分析結果を保存")
        return response.data[0]["id"]
    
    except Exception as e:
        print(f"❌ 分析結果の保存エラー: {e}")
        return None

def get_recent_readings(hours=24):
    """
    過去 N 時間のセンサー読取と分析を取得
    
    Args:
        hours (int): 過去何時間か（デフォルト：24時間）
    
    Returns:
        list: レコードのリスト
    """
    
    try:
        from datetime import datetime, timedelta
        
        # 過去 N 時間を計算
        since = (datetime.now() - timedelta(hours=hours)).isoformat()
        
        response = supabase.table("sensor_readings").select(
            "*, analyses(analysis_text, recommendations, confidence)"
        ).gte("timestamp", since).order("timestamp", desc=True).execute()
        
        return response.data
    
    except Exception as e:
        print(f"❌ 取得エラー: {e}")
        return []

def get_latest():
    """最新の読取と分析を取得"""
    
    try:
        response = supabase.table("sensor_readings").select(
            "*, analyses(analysis_text, recommendations, confidence)"
        ).order("timestamp", desc=True).limit(1).single().execute()
        
        return response.data
    
    except Exception as e:
        print(f"❌ 取得エラー: {e}")
        return None

# 使用例
if __name__ == "__main__":
    print("Supabase 接続テスト")
    
    # テスト用のセンサーデータ
    test_sensor_data = {
        "timestamp": datetime.now().isoformat() + "Z",
        "sensor_id": "dht22_001",
        "location": "living_room",
        "temperature": {"value": 23.5},
        "humidity": {"value": 65.2},
        "status": "success"
    }
    
    # テスト用の分析データ
    test_analysis_data = {
        "comfort_level": "comfortable",
        "analysis": "室内環境は快適です",
        "recommendations": [
            {"action": "換気", "reason": "空気を入れ替える"}
        ],
        "confidence": 0.95
    }
    
    # 保存してみる
    reading_id = save_sensor_reading(test_sensor_data)
    if reading_id:
        save_analysis(reading_id, test_analysis_data)
    
    # 最新データを取得
    latest = get_latest()
    if latest:
        print("\n最新データ:")
        print(json.dumps(latest, indent=2, ensure_ascii=False))
```

### 実行してテスト

```bash
python3 ~/database_manager.py
```

成功すれば、Supabase ダッシュボード（「SQL Editor」の「sensor_readings」タブ）にデータが表示されます。

---

## 🌐 ステップ3: Web ダッシュボードを構築

### Next.js プロジェクト作成（デスクトップ PC で）

```bash
# PC のターミナルで
npx create-next-app@latest iot-dashboard --typescript

cd iot-dashboard
npm install recharts supabase-js
```

### 環境変数を設定

`.env.local` を作成：

```env
NEXT_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### API ルート作成

`pages/api/readings.ts`:

```typescript
import { createClient } from '@supabase/supabase-js'
import { NextApiRequest, NextApiResponse } from 'next'

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  try {
    const { data } = await supabase
      .from('sensor_readings')
      .select('*, analyses(*)')
      .order('timestamp', { ascending: false })
      .limit(48)  // 過去 48 件
    
    return res.status(200).json(data)
  } catch (error) {
    return res.status(500).json({ error: error })
  }
}
```

### React コンポーネント作成

`components/Dashboard.tsx`:

```typescript
import { useEffect, useState } from 'react'
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip } from 'recharts'

interface Reading {
  timestamp: string
  temperature: number
  humidity: number
  analyses: { analysis_text: string; confidence: number }[]
}

export default function Dashboard() {
  const [data, setData] = useState<Reading[]>([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    const fetchData = async () => {
      try {
        const res = await fetch('/api/readings')
        const readings = await res.json()
        
        // グラフ用にデータを整形
        const chartData = readings.reverse().map((r: Reading) => ({
          time: new Date(r.timestamp).toLocaleTimeString('ja-JP'),
          temp: r.temperature,
          humidity: r.humidity
        }))
        
        setData(chartData)
      } catch (error) {
        console.error('Error:', error)
      } finally {
        setLoading(false)
      }
    }

    fetchData()
    // 5 秒ごとに更新
    const interval = setInterval(fetchData, 5000)
    return () => clearInterval(interval)
  }, [])

  if (loading) return <div>読み込み中...</div>

  return (
    <div className="p-8">
      <h1 className="text-3xl font-bold mb-8">IoT ダッシュボード</h1>
      
      <div className="grid grid-cols-2 gap-8">
        {/* 温度グラフ */}
        <div className="bg-white p-4 rounded shadow">
          <h2 className="text-xl font-bold mb-4">温度推移</h2>
          <LineChart width={400} height={300} data={data}>
            <CartesianGrid />
            <XAxis dataKey="time" />
            <YAxis />
            <Tooltip />
            <Line type="monotone" dataKey="temp" stroke="#ff7300" />
          </LineChart>
        </div>
        
        {/* 湿度グラフ */}
        <div className="bg-white p-4 rounded shadow">
          <h2 className="text-xl font-bold mb-4">湿度推移</h2>
          <LineChart width={400} height={300} data={data}>
            <CartesianGrid />
            <XAxis dataKey="time" />
            <YAxis />
            <Tooltip />
            <Line type="monotone" dataKey="humidity" stroke="#0088fe" />
          </LineChart>
        </div>
      </div>
    </div>
  )
}
```

### 実行

```bash
npm run dev

# http://localhost:3000 をブラウザで開く
```

---

## 📊 データを活用する

### 傾向分析

```sql
-- 1 日の平均温度を調べる
SELECT 
    DATE(timestamp) as date,
    AVG(temperature) as avg_temp,
    MIN(temperature) as min_temp,
    MAX(temperature) as max_temp
FROM sensor_readings
GROUP BY DATE(timestamp)
ORDER BY date DESC;
```

### 異常検知

```sql
-- 温度が 28°C を超えた時間を検出
SELECT 
    timestamp,
    temperature,
    humidity
FROM sensor_readings
WHERE temperature > 28
ORDER BY timestamp DESC;
```

### AI 分析の蓄積

```sql
-- 「快適」と分類された時間の平均条件を調べる
SELECT 
    AVG(r.temperature) as avg_temp,
    AVG(r.humidity) as avg_humidity,
    COUNT(*) as count
FROM sensor_readings r
JOIN analyses a ON r.id = a.reading_id
WHERE a.environment_status = 'comfortable';
```

---

## 🔧 トラブルシューティング

### Supabase 接続エラー

```
❌ Error: Failed to fetch data
```

確認事項：
1. 環境変数が正しく設定されているか
2. Supabase のテーブルが作成されているか

```bash
echo $SUPABASE_URL
echo $SUPABASE_ANON_KEY
```

### グラフが表示されない

- Recharts がインストールされているか確認：
  ```bash
  npm list recharts
  ```
- ブラウザのコンソールでエラーを確認

---

## 🎉 プロジェクト完成！

3 回のシリーズで、以下が実現できました：

✅ **第1回**: センサーからデータを読み込む  
✅ **第2回**: AI が分析する  
✅ **第3回**: 結果を保存して可視化  

### 実装したアーキテクチャ

```
【Raspberry Pi】
├─ sensor_reader.py（DHT11 からデータ読み込み）
└─ claude_analyzer.py（AI 分析）
        ↓
【Supabase】
├─ sensor_readings テーブル
├─ analyses テーブル
└─ REST API
        ↓
【Web ダッシュボード】
├─ グラフ表示（温度・湿度の推移）
├─ AI 分析結果
└─ リアルタイム更新
```

---

## 🚀 さらに応用できること

### 1. メール通知
温度が閾値を超えたとき、自動でメール送信

### 2. 複数センサー対応
複数の場所に DHT11 を設置して、比較分析

### 3. 機械学習
過去データから異常検知モデルを構築

### 4. モバイルアプリ化
React Native で iPhone / Android アプリ化

### 5. ホームオートメーション
エアコンを自動制御（温度が高い → 自動でON）

---

## 📚 参考資料

- 🔗 [Supabase Documentation](https://supabase.com/docs)
- 🔗 [Next.js Tutorial](https://nextjs.org/learn)
- 🔗 [Recharts Docs](https://recharts.org/)

---

## 💬 まとめ

ここまでで、あなたは**フルスタック IoT システム**を構築しました。

- センサー値を読み込むだけでなく
- AI に分析させ
- データベースで保存し
- Web で可視化する

これは、実際の企業が使用するシステムと同じパターンです。

**今後の学習では、これらを組み合わせて、さらに複雑なプロジェクトに挑戦できます。**

**質問やフィードバックは、コメント欄へ！**

**「ここ違うよ」「こうした方がいい」があれば、ぜひコメントで教えてください。全部ありがたく受け止めて、直していきます！**

🎊 **シリーズ完結おめでとう！** 🎊

