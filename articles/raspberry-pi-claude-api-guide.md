---
title: "" # ビギナー向け:Raspberry Pi + Claude API統合ガイド　～センサーデータ分析から可視化～ （上）
emoji: "🍓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Raspberry Pi", "Claude", "Python", "IoT", "API"]
published: false
published_at: "2026-04-30 12:00" 
---

# Raspberry Piで温湿度センサーを動かすまで

こんにちは！このシリーズでは、Raspberry Piを使ったIoTプロジェクトを段階的に構築します。まず第1回では、**センサーからデータを読み込むまで**の基本を学びます。

## 📚 この記事で学べること

このシリーズは全3回の構成です：

- **第1回（この記事）**: センサー読み込みの基本
- **第2回**: Claude APIでAI分析
- **第3回**: データベース + ダッシュボード

## 🎯 今回のゴール

記事完了時点で、以下が実現できます：

```
DHT11センサー → Raspberry Pi → JSON形式でターミナルに出力
```

簡潔で、でも実用的。これが出発点です。

---

## 必要な機材（最小限）

| 項目 | 用途 |
|------|------|
| **Raspberry Pi 4B** | メインコンピュータ |
| **DHT11センサー** | 温湿度測定 |
| **microSDカード** | OS用（32GB推奨） |
| **microUSB/USB-C電源** | 給電 |
| **ジャンパーワイヤー** | GPIO配線（赤・黒・黄） |
| **マイクロHDMI** | ディスプレイ接続（初期セットアップ用） |

> ブレッドボードがあると便利ですが、必須ではありません。

---

## 🛠️ ステップ1: Raspberry Pi OS のセットアップ

### Raspberry Pi Imagerで OS をインストール

1. **Imager をダウンロード**  
   https://www.raspberrypi.com/software/ からあなたのOSに合わせてダウンロード

2. **起動して、以下を選択:**
   - Raspberry Pi デバイス: **Raspberry Pi 4**
   - OS: **Raspberry Pi OS (64-bit)**
   - ストレージ: **あなたの microSD カード**

3. **編集を押して、初期設定:**
   ```
   ホスト名: raspberrypi
   ユーザー名: pi
   パスワード: （任意）
   WiFi SSID: （あなたのWiFi）
   WiFiパスワード: （パスワード）
   ```

4. **書き込みをクリック** → 5～10分待つ

### Raspberry Pi を起動

```bash
# インストール後、Raspberry Pi に microSD を挿入して起動
# 初回は自動で展開され、数分かかります

# WiFi接続後、ターミナルを開いて以下を実行
sudo apt update
sudo apt upgrade -y
sudo apt install -y python3-pip python3-dev git

# Pythonパッケージマネージャをアップグレード
pip3 install --upgrade pip
```

### DHT11 ライブラリをインストール

```bash
pip3 install Adafruit-DHT
```

インストール確認：

```bash
python3 -c "import Adafruit_DHT; print('✅ OK')"
```

---

## 🔌 ステップ2: DHT11 をRaspberry Pi に接続

### DHT11 の ピン配置

DHT11センサーは4本のピンを持ちます：

```
DHT11
┌─────────────────┐
│ ① VCC  (赤)    │ → Raspberry Pi +5V (ピン2)
│ ② DATA (黄)    │ → Raspberry Pi GPIO 4 (ピン7)
│ ③ NC          │ → 接続しない
│ ④ GND  (黒)    │ → Raspberry Pi GND (ピン6)
└─────────────────┘
```

### Raspberry Pi のピン図

```
Raspberry Pi 40-pin GPIO

3.3V━━❶  ❷━━5V
GPIO2━━③  ④━━5V
GPIO3━━⑤  ⑥━━GND
GPIO4━━⑦  ⑧━━GPIO14 ⭐ DHT11 のDATA はGPIO4（ピン7）に接続
GPIO17━❾  ⑩━━GPIO15
GPIO27━⑪ ⑫━━GPIO18
GPIO22━⑬ ⑭━━GND
GPIO23━⑮ ⑯━━GPIO24
GPIO24━⑰ ⑱━━GPIO25
GPIO25━⑲ ⑳━━GND
... 以下省略
```

### 配線手順

1. **Raspberry Pi の電源を OFF にする**（重要！）

2. **DHT11 を接続:**

| DHT11 | Raspberry Pi | ワイヤー色 |
|-------|--------------|-----------|
| VCC（①） | ピン2（5V） | 赤 |
| DATA（②） | ピン7（GPIO 4） | 黄 |
| GND（④） | ピン6（GND） | 黒 |

3. **電源を入れる**

4. **接続確認:**
   ```bash
   gpio readall
   ```

---

## 📊 ステップ3: センサー読み込みプログラム

### sensor_reader.py を作成

Raspberry Pi 上で新しいファイルを作成します：

```bash
nano ~/sensor_reader.py
```

以下をコピー & ペーストして、Ctrl+X → Y → Enter で保存：

```python
#!/usr/bin/env python3
# sensor_reader.py - DHT11 センサー読み込みプログラム

import Adafruit_DHT
import json
from datetime import datetime
import time
import sys

# センサー設定
SENSOR = Adafruit_DHT.DHT11
GPIO_PIN = 4

def read_sensor():
    """DHT11 から温度・湿度を読み込む"""
    humidity, temperature = Adafruit_DHT.read_retry(SENSOR, GPIO_PIN)
    
    if humidity is None or temperature is None:
        return None
    
    # JSON形式で返す
    return {
        "timestamp": datetime.now().isoformat() + "Z",
        "sensor_id": "dht11_001",
        "location": "living_room",
        "temperature": {
            "value": round(temperature, 1),
            "unit": "celsius"
        },
        "humidity": {
            "value": round(humidity, 1),
            "unit": "percent"
        },
        "status": "success"
    }

def main():
    print("=" * 60)
    print("DHT11 センサー読み込みプログラム")
    print(f"GPIO ピン: {GPIO_PIN}")
    print("Ctrl+C で終了")
    print("=" * 60)
    
    try:
        while True:
            sensor_data = read_sensor()
            
            if sensor_data:
                print(f"\n✅ [{sensor_data['timestamp']}]")
                print(f"   温度: {sensor_data['temperature']['value']}°C")
                print(f"   湿度: {sensor_data['humidity']['value']}%")
                print(json.dumps(sensor_data, indent=2, ensure_ascii=False))
            else:
                print(f"\n❌ [{datetime.now().isoformat()}] センサーエラー")
            
            print("-" * 60)
            time.sleep(5)
    
    except KeyboardInterrupt:
        print("\n\nプログラム終了")
        sys.exit(0)

if __name__ == "__main__":
    main()
```

### プログラムを実行

```bash
sudo python3 ~/sensor_reader.py
```

**期待される出力:**

```
============================================================
DHT11 センサー読み込みプログラム
GPIO ピン: 4
Ctrl+C で終了
============================================================

✅ [2026-04-29T14:30:00Z]
   温度: 23.5°C
   湿度: 65.2%
{
  "timestamp": "2026-04-29T14:30:00Z",
  "sensor_id": "dht11_001",
  "location": "living_room",
  "temperature": {
    "value": 23.5,
    "unit": "celsius"
  },
  "humidity": {
    "value": 65.2,
    "unit": "percent"
  },
  "status": "success"
}
------------------------------------------------------------
```

---

## 🔧 よくあるエラーと対処法

### ❌ モジュールが見つからない

```
ModuleNotFoundError: No module named 'Adafruit_DHT'
```

**解決策:**

```bash
pip3 install Adafruit-DHT
```

### ❌ センサーが反応しない

```
❌ [2026-04-29T14:30:00Z] センサーエラー
```

**確認事項:**

1. **配線が正しいか確認:**
   - 赤いワイヤー (VCC) が +5V に接続？
   - 黄色いワイヤー (DATA) が GPIO 4 に接続？
   - 黒いワイヤー (GND) が GND に接続？

2. **別のGPIOピンを試す:**
   ```python
   # gpio_pin = 4 を 17 に変更
   GPIO_PIN = 17  # 試す
   ```

3. **センサーの交換を検討**

### ❌ Permission denied

```
PermissionError: [Errno 13] Permission denied
```

**解決策:**

```bash
sudo python3 ~/sensor_reader.py
```

（`sudo` をつけて実行）

---

## 💾 コードの保存・共有

完全な `sensor_reader.py` は GitHub Gist に保存しています：

🔗 https://gist.github.com/DanL/your-gist-id

> 今後、このプログラムを拡張していきます。Gist をスターしておくと、更新を追跡できます。

---

## 📈 次のステップ

おめでとうございます！ここまでで以下が実現できました：

✅ Raspberry Pi OS のセットアップ  
✅ DHT11 の配線  
✅ Python でセンサーから読み込み  
✅ JSON 形式でデータ整形  

### 🚀 次の記事では

センサーから得たデータを **Claude API** に送信して、AI による分析を行います。例えば：

```
温度: 25.5°C
湿度: 68%
↓
Claude API に送信
↓
「温度がやや高いです。エアコンをつけることをお勧めします」
```

データを集めるだけでなく、**AI が意味のある提案をしてくれる**ようになります。

---

## 📝 まとめ

この記事で学んだこと：

1. Raspberry Pi OS のセットアップ
2. DHT11 センサーの配線
3. Python での GPIO 制御
4. センサーデータを JSON に変換
5. 5 秒ごとのデータ読み込みループ

次の記事では、このデータを **Claude API** で分析する方法を解説します。

**日本語もIoTも勉強中の初心者です。「ここ違うよ」「こうした方がいい」があれば、ぜひコメントで教えてください。全部ありがたく受け止めて、直していきます！**

---

## 📚 参考資料

- 🔗 [Adafruit DHT Library](https://github.com/adafruit/Adafruit_Python_DHT)
- 🔗 [Raspberry Pi GPIO Documentation](https://www.raspberrypi.com/documentation/computers/gpio.html)
- 🔗 [JSON チートシート](https://www.json.org/json-ja.html)

**次回をお楽しみに！🚀**
