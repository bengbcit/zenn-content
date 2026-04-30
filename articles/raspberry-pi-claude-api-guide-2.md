---
title: "ビギナー向け:Raspberry Pi + Claude API統合ガイド　～Claude APIでセンサーデータをAI分析～ （中)"
emoji: "🍓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Raspberry Pi", "Claude", "Python", "IoT", "API"]
published: true
published_at: "2026-05-01 12:00" 
---

# Claude APIでセンサーデータをAI分析

第1回では、Raspberry Piのセンサーから温度・湿度データを取得できるようになりました。

今回は、そのデータを **Claude API** に送信して、AIに分析してもらいます。ただ数字を見るのではなく、**AI が環境について人間が読める形式で提案してくれる**ようになります。

## 🎯 この記事で学べること

```
センサーから取得したJSON
    ↓
Claude APIに送信
    ↓
AI の分析結果をJSON で取得
    ↓
ターミナルに表示
```

この流れを実装します。

### 📋 前提条件

- 第1回の `sensor_reader.py` が動作している
- Python 3.8 以上
- インターネット接続（Claude API 呼び出しのため）

---

## 🔑 ステップ1: Claude API キーの取得

### アカウント作成 & キー生成

1. **Anthropic Console にアクセス:**  
   https://console.anthropic.com/

2. **左メニューから「API Keys」を選択**

3. **「Create Key」をクリック**

4. **キーをコピー** （`sk-ant-` で始まる長い文字列）

> 🔒 **重要**: このキーを他人に教えてはいけません。GitHub にも押さえておいてください。

### Raspberry Pi で環境変数を設定

ターミナルで：

```bash
nano ~/.bashrc
```

ファイルの最後に以下を追加（YOUR_API_KEY をあなたのキーに置き換え）：

```bash
export ANTHROPIC_API_KEY="sk-ant-xxxxxxxxxxxxxxxxxxxxxxxx"
```

Ctrl+X → Y → Enter で保存

反映させる：

```bash
source ~/.bashrc

# 確認
echo $ANTHROPIC_API_KEY
```

`sk-ant-` で始まる文字列が表示されれば OK です。

---

## 📦 ステップ2: Anthropic SDK のインストール

```bash
pip3 install anthropic
```

確認：

```bash
python3 -c "import anthropic; print('✅ OK')"
```

---

## 💬 ステップ3: Claude API を使ってセンサーデータを分析

### claude_analyzer.py を作成

```bash
nano ~/claude_analyzer.py
```

以下をコピー & ペースト：

```python
#!/usr/bin/env python3
# claude_analyzer.py - Claude API でセンサーデータを分析

import anthropic
import json
from datetime import datetime
import sys

# Anthropic クライアントの初期化
# ANTHROPIC_API_KEY 環境変数から自動的に取得
client = anthropic.Anthropic()

def analyze_sensor_data(temperature, humidity):
    """
    温度・湿度データを Claude API で分析する
    
    Args:
        temperature (float): 温度（℃）
        humidity (float): 湿度（%）
    
    Returns:
        dict: 分析結果の JSON
    """
    
    # Claude に送信するプロンプト
    # 日本語で詳しく分析してもらう
    prompt = f"""
あなたは室内環境の専門家です。以下のセンサーデータを分析してください。

【センサーデータ】
- 温度: {temperature}°C
- 湿度: {humidity}%
- 計測時刻: {datetime.now().isoformat()}

【分析してほしい項目】
1. 環境の快適性（快適 / 注意 / 危険）
2. 温度評価（低い / 適切 / 高い）
3. 湿度評価（乾燥 / 適切 / 多湿）
4. 実行可能な改善提案（最大3つ）
5. 健康への影響

【応答形式】
必ず以下の JSON 形式で返してください：
{{
    "comfort_level": "comfortable" | "warning" | "critical",
    "temperature_assessment": "low" | "optimal" | "high",
    "humidity_assessment": "dry" | "normal" | "humid",
    "analysis": "（詳しい分析説明。日本語3〜4文）",
    "recommendations": [
        {{
            "action": "（具体的な行動）",
            "reason": "（理由）",
            "priority": "high" | "medium" | "low"
        }}
    ],
    "health_impact": "（健康への影響。1〜2文）",
    "confidence": 0.95
}}
"""
    
    print("🤖 Claude API を呼び出し中...")
    
    # Claude Sonnet 3.5 を使用（高速で十分な品質）
    message = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        messages=[
            {"role": "user", "content": prompt}
        ]
    )
    
    # 応答テキストを取得
    response_text = message.content[0].text
    
    print("✅ API 応答を受け取りました")
    
    # JSON パースを試みる
    try:
        # ```json ... ``` で囲まれている場合、抽出
        if "```json" in response_text:
            json_str = response_text.split("```json")[1].split("```")[0].strip()
        elif "```" in response_text:
            json_str = response_text.split("```")[1].split("```")[0].strip()
        else:
            json_str = response_text
        
        analysis_result = json.loads(json_str)
        analysis_result["timestamp"] = datetime.now().isoformat()
        
        return analysis_result
    
    except json.JSONDecodeError as e:
        print(f"⚠️ JSON パース失敗: {e}")
        return {
            "status": "parse_error",
            "raw_response": response_text,
            "timestamp": datetime.now().isoformat()
        }

def format_analysis_output(analysis):
    """分析結果を人間が読みやすい形式で表示"""
    
    print("\n" + "=" * 70)
    print("📊 AI 分析結果")
    print("=" * 70)
    
    if analysis.get("status") == "parse_error":
        print("⚠️ JSON パースに失敗しました")
        print(f"生のテキスト:\n{analysis.get('raw_response')}")
        return
    
    # 快適性レベルを表示
    comfort_emoji = {
        "comfortable": "🟢",
        "warning": "🟡",
        "critical": "🔴"
    }
    comfort = analysis.get("comfort_level", "unknown")
    print(f"\n{comfort_emoji.get(comfort, '⚪')} 快適性: {comfort}")
    
    # 温度・湿度評価
    print(f"\n🌡️ 温度: {analysis.get('temperature_assessment')}")
    print(f"💧 湿度: {analysis.get('humidity_assessment')}")
    
    # 詳しい分析
    print(f"\n📝 分析:")
    print(f"   {analysis.get('analysis', 'N/A')}")
    
    # 健康への影響
    print(f"\n⚕️ 健康への影響:")
    print(f"   {analysis.get('health_impact', 'N/A')}")
    
    # 推奨事項
    recommendations = analysis.get("recommendations", [])
    if recommendations:
        print(f"\n💡 推奨事項:")
        for i, rec in enumerate(recommendations, 1):
            priority_emoji = {
                "high": "🔴",
                "medium": "🟡",
                "low": "🟢"
            }
            emoji = priority_emoji.get(rec.get("priority"), "⚪")
            print(f"   {emoji} {i}. {rec.get('action')}")
            print(f"      → {rec.get('reason')}")
    
    # 信頼度
    confidence = analysis.get("confidence", 0)
    print(f"\n🎯 AI 確信度: {confidence*100:.0f}%")
    print("=" * 70 + "\n")

def main():
    """メインプログラム: sensor_reader.py から値を受け取って分析"""
    
    # テスト用のサンプル値
    # 実際には sensor_reader.py から取得することもできます
    test_cases = [
        (23.5, 65.2),  # 快適な環境
        (28.0, 75.0),  # 温度・湿度が高い
        (18.0, 35.0),  # 温度が低く、乾燥している
    ]
    
    print("=" * 70)
    print("Claude API センサー分析デモ")
    print("=" * 70)
    
    for temp, humidity in test_cases:
        print(f"\n📋 テスト #{test_cases.index((temp, humidity))+1}")
        print(f"   入力: 温度 {temp}°C, 湿度 {humidity}%")
        
        # 分析を実行
        analysis = analyze_sensor_data(temp, humidity)
        
        # 結果を表示
        format_analysis_output(analysis)

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\n\nプログラム終了")
        sys.exit(0)
    except Exception as e:
        print(f"\n❌ エラーが発生しました: {e}")
        sys.exit(1)
```

### プログラムを実行

```bash
python3 ~/claude_analyzer.py
```

**期待される出力:**

```
======================================================================
Claude API センサー分析デモ
======================================================================

📋 テスト #1
   入力: 温度 23.5°C, 湿度 65.2%
🤖 Claude API を呼び出し中...
✅ API 応答を受け取りました

======================================================================
📊 AI 分析結果
======================================================================

🟢 快適性: comfortable

🌡️ 温度: optimal
💧 湿度: normal

📝 分析:
   室内環境は非常に快適な状態です。温度と湿度のバランスが理想的です。

⚕️ 健康への影響:
   このような環境は睡眠の質を向上させ、作業効率も高まります。

💡 推奨事項:
   🟢 1. 定期的な換気
      → 現在の環境を保つため、1〜2時間ごとに窓を開けることをお勧めします
   🟢 2. 湿度の維持
      → 加湿器の使用は不要ですが、エアコン使用時は注意してください
   🟢 3. 空気清浄機の活用
      → オプションですが、さらに空気質を向上させることができます

🎯 AI 確信度: 95%
======================================================================
```

---

## 🎨 ステップ4: センサーリーダーと統合

`sensor_reader.py` の出力を直接 Claude API に送信する方法：

```python
# integrated_analyzer.py の例

import subprocess
import json
import anthropic
from datetime import datetime

client = anthropic.Anthropic()

def get_sensor_data():
    """sensor_reader.py を実行してセンサーデータを取得"""
    try:
        result = subprocess.run(
            ["python3", "~/sensor_reader.py"],
            capture_output=True,
            text=True,
            timeout=10
        )
        # JSON行を抽出（複数行の出力から）
        for line in result.stdout.split('\n'):
            try:
                return json.loads(line)
            except:
                continue
    except Exception as e:
        print(f"❌ センサー読み込みエラー: {e}")
        return None

# この関数を main() から呼び出す
sensor_data = get_sensor_data()
if sensor_data:
    analysis = analyze_sensor_data(
        sensor_data["temperature"]["value"],
        sensor_data["humidity"]["value"]
    )
```

---

## 🎓 プロンプトエンジニアリングのコツ

Claude の分析精度を上げるには、プロンプトの書き方が重要です：

### ✅ 良い例

```python
prompt = """
あなたは室内環境の専門家です。
以下のセンサーデータを分析してください。

【データ】
- 温度: {temperature}°C
- 湿度: {humidity}%

【形式】
JSON で以下のフィールドを含めてください：
{
    "comfort_level": "comfortable" | "warning" | "critical",
    "analysis": "詳しい説明",
    ...
}
"""
```

**ポイント:**
- 役割を明確に（「専門家です」）
- 入力形式を明記
- 出力形式を JSON で指定
- 選択肢を明確に（`|` で区切る）

### ❌ 避けるべき例

```python
prompt = f"温度は {temperature} で湿度は {humidity} です。どう思いますか？"
```

**問題:**
- 出力形式が定義されていない
- 曖昧な質問
- JSON パースできない可能性

---

## 🔧 トラブルシューティング

### API キーエラー

```
AuthenticationError: Invalid API key
```

**確認事項:**

```bash
echo $ANTHROPIC_API_KEY
```

`sk-ant-` で始まっていることを確認。変わっていなければ、以下を試す：

```bash
source ~/.bashrc
python3 ~/claude_analyzer.py
```

### タイムアウト

```
Timeout error
```

インターネット接続を確認。またはレート制限の可能性。少し待ってから再度実行。

### JSON パースエラー

Claude の応答が JSON でない場合。プロンプトを調整：

```python
prompt = "以下を JSON で返してください:\n{...}"
```

---

## 🚀 次のステップ

おめでとうございます！これで以下が実現できました：

✅ Claude API キーの取得と設定  
✅ Python から Claude API への呼び出し  
✅ セセンサーデータの AI 分析  
✅ 人間が読める形式での出力  

### 第3回では

分析結果を **Supabase** （PostgreSQL）に保存し、**Web ダッシュボード**でリアルタイム表示します。

```
API 分析結果
    ↓
Supabase に保存
    ↓
Web UI にリアルタイム表示
```

データが蓄積されることで、**傾向分析**もできるようになります。

**「ここ違うよ」「こうした方がいい」があれば、ぜひコメントで教えてください。全部ありがたく受け止めて、直していきます！**
---

## 📚 参考資料

- 🔗 [Claude API Documentation](https://docs.anthropic.com/)
- 🔗 [Anthropic SDK for Python](https://github.com/anthropics/anthropic-sdk-python)
- 🔗 [プロンプトエンジニアリングのベストプラクティス](https://docs.anthropic.com/prompting/)

**質問やフィードバックは、コメント欄へ！次回をお楽しみに！🚀**

