# Nozomi 実装用マスタープロンプト

**Version:** 1.0.0
**対象:** AI開発アシスタント / 開発者
**目的:** 統合型AI鑑定システム「Nozomi」の完全実装

---

## このプロンプトの使い方

このマスタープロンプトは、AI開発アシスタント（Claude、GPT-4等）に渡すことで、Nozomiシステムの各コンポーネントを段階的に実装できるように設計されています。

### 推奨される使用方法

1. **フェーズ分割**: このプロンプトを6つのフェーズに分けて実装
2. **検証**: 各フェーズ完了後、ユニットテストと統合テストを実行
3. **反復改善**: ユーザーフィードバックを元に継続的に改善

---

## マスタープロンプト本文

```
# 統合型AI鑑定システム「Nozomi」実装指示書

あなたは経験豊富なフルスタック開発者です。以下の仕様に基づき、統合型AI鑑定システム「Nozomi」を実装してください。

## プロジェクト概要

**システム名**: Nozomi（のぞみ / 希）
**目的**: 6大占術（西洋占星術、四柱推命、タロット、数秘術、九星気学、姓名判断）と霊視AIを統合し、6000文字の詳細鑑定を生成。3段階のアップセル機能により収益化。

## 技術スタック

### フロントエンド
- Next.js 14.2+ (App Router)
- TypeScript 5.3+
- Tailwind CSS
- React Hook Form + Zod（バリデーション）
- Framer Motion（アニメーション）

### バックエンド
- Node.js 20+ / Express
- Python 3.11+（占術計算エンジン）
- PostgreSQL 16（ユーザー・決済データ）
- MongoDB 7（鑑定結果・ドキュメント）
- Redis 7（キャッシュ・セッション）

### AI / ML
- Anthropic Claude Sonnet 4.5 API
- Swiss Ephemeris（天体計算）
- TensorFlow 2.15（霊視AIモデル - オプション）

### インフラ
- Docker + Docker Compose
- Kubernetes（本番環境）
- GCP（Cloud Run, Cloud SQL, GKE）
- Stripe（決済）

---

## 実装フェーズ

### Phase 1: 基盤構築（Week 1-2）

#### タスク 1.1: プロジェクトセットアップ

**指示:**
以下のディレクトリ構造でプロジェクトを作成してください。

\`\`\`
nozomi/
├── frontend/                 # Next.js アプリ
│   ├── src/
│   │   ├── app/             # App Router
│   │   ├── components/      # Reactコンポーネント
│   │   ├── lib/             # ユーティリティ
│   │   └── types/           # TypeScript型定義
│   ├── public/
│   ├── package.json
│   └── tsconfig.json
│
├── backend/                 # Express API
│   ├── src/
│   │   ├── routes/          # APIルート
│   │   ├── controllers/     # ビジネスロジック
│   │   ├── services/        # 外部API連携
│   │   ├── models/          # データモデル
│   │   └── middleware/      # 認証・エラーハンドリング
│   ├── package.json
│   └── tsconfig.json
│
├── divination-engine/       # Python占術エンジン
│   ├── modules/
│   │   ├── astrology.py     # 西洋占星術
│   │   ├── four_pillars.py  # 四柱推命
│   │   ├── tarot.py         # タロット
│   │   ├── numerology.py    # 数秘術
│   │   ├── nine_star.py     # 九星気学
│   │   └── name_analysis.py # 姓名判断
│   ├── orchestrator.py      # 統合制御
│   ├── requirements.txt
│   └── tests/
│
├── spiritual-vision/        # 霊視AIモジュール
│   ├── model/               # 機械学習モデル
│   ├── inference.py         # 推論エンジン
│   └── requirements.txt
│
├── docker/
│   ├── Dockerfile.frontend
│   ├── Dockerfile.backend
│   ├── Dockerfile.python
│   └── docker-compose.yml
│
├── k8s/                     # Kubernetes設定
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
│
└── docs/
    ├── API.md
    └── DESIGN.md
\`\`\`

**実装要件:**
1. package.json / requirements.txt を作成
2. TypeScript strict mode有効化
3. ESLint + Prettier設定
4. .env.example ファイル作成（API key template）

**期待される出力:**
- すべてのディレクトリとファイルが作成されている
- \`npm install\` / \`pip install\` が成功する
- \`npm run dev\` でNext.jsが起動する

---

#### タスク 1.2: データベーススキーマ実装

**指示:**
PostgreSQLとMongoDBのスキーマを実装してください。

**PostgreSQL（backend/src/models/schema.sql）:**

\`\`\`sql
-- Users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),  -- bcrypt
    name VARCHAR(100),
    birth_date DATE,
    birth_time TIME,
    birth_latitude DECIMAL(10, 8),
    birth_longitude DECIMAL(11, 8),
    birth_timezone VARCHAR(50) DEFAULT 'Asia/Tokyo',
    gender VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- サブスクリプション
    subscription_tier VARCHAR(50) DEFAULT 'free',
    subscription_started_at TIMESTAMP,
    subscription_expires_at TIMESTAMP,
    stripe_customer_id VARCHAR(255),

    -- 統計
    total_readings INTEGER DEFAULT 0,
    last_reading_at TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_subscription ON users(subscription_tier, subscription_expires_at);

-- Readings table
CREATE TABLE readings (
    reading_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    question TEXT,
    status VARCHAR(50) DEFAULT 'processing',  -- processing, completed, failed
    tier VARCHAR(50) DEFAULT 'free',          -- free, detailed, premium
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    processing_time_ms INTEGER,
    word_count INTEGER,

    -- 鑑定設定
    divination_types TEXT[],  -- ['astrology', 'tarot', ...]
    include_spiritual_vision BOOLEAN DEFAULT true
);

CREATE INDEX idx_readings_user ON readings(user_id, created_at DESC);
CREATE INDEX idx_readings_status ON readings(status);

-- Payments table
CREATE TABLE payments (
    payment_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id),
    reading_id UUID REFERENCES readings(reading_id),
    payment_type VARCHAR(50),  -- one_time, subscription
    amount INTEGER,            -- 金額（円）
    currency VARCHAR(3) DEFAULT 'JPY',
    payment_method VARCHAR(50),
    stripe_payment_intent_id VARCHAR(255),
    status VARCHAR(50),        -- pending, succeeded, failed, refunded
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    paid_at TIMESTAMP
);

CREATE INDEX idx_payments_user ON payments(user_id, created_at DESC);
CREATE INDEX idx_payments_status ON payments(status);

-- Consultations table（個別相談予約）
CREATE TABLE consultations (
    consultation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id),
    consultant_name VARCHAR(100),
    scheduled_at TIMESTAMP NOT NULL,
    duration_minutes INTEGER DEFAULT 60,
    zoom_link TEXT,
    status VARCHAR(50) DEFAULT 'scheduled',  -- scheduled, completed, cancelled
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_consultations_user ON consultations(user_id);
CREATE INDEX idx_consultations_schedule ON consultations(scheduled_at);
\`\`\`

**MongoDB（スキーマ設計書）:**

\`\`\`javascript
// Collection: readings_data
// 鑑定結果の詳細データを格納（PostgreSQLには入らない大容量データ）

{
  _id: ObjectId("..."),
  reading_id: "rdg_12345",  // PostgreSQLと紐付け
  user_id: "usr_67890",

  // 占術計算結果
  divination_results: {
    astrology: {
      sun_sign: "牡羊座",
      moon_sign: "蟹座",
      ascendant: "獅子座",
      planets: [
        { name: "sun", sign: "牡羊座", house: 9, degree: 15.23 },
        { name: "moon", sign: "蟹座", house: 12, degree: 8.45 }
        // ...
      ],
      aspects: [
        { planet1: "sun", planet2: "mars", aspect: "trine", orb: 2.3 }
      ],
      interpretation: {
        personality: "リーダーシップに優れ...",
        strength: "決断力、行動力",
        challenge: "衝動性、短気"
      }
    },
    four_pillars: { /* 四柱推命データ */ },
    tarot: { /* タロット結果 */ },
    numerology: { /* 数秘術 */ },
    nine_star: { /* 九星気学 */ },
    name_analysis: { /* 姓名判断 */ }
  },

  // 霊視AIの出力
  spiritual_vision: {
    insight_text: "私にはあなたの周りに...",
    confidence_score: 0.87,
    key_symbols: ["光", "扉", "新しい道"],
    model_version: "spiritual-v1.2.0"
  },

  // 生成されたテキスト
  generated_text: {
    sections: {
      header: "あなたの運命を紐解く...",
      overall_fortune: "現在、あなたは...",
      personality: "あなたは生まれながらに...",
      love: "恋愛面では...",
      career: "仕事運は...",
      money: "金運について...",
      health: "健康面では...",
      spiritual: "霊視による洞察...",
      monthly_forecast: "今月の運勢...",
      quarterly_forecast: "3ヶ月の展望...",
      yearly_forecast: "1年の流れ...",
      action_plan: "具体的には...",
      upsell: "さらに詳しく知りたい方へ..."
    },
    full_text: "（6000文字の完全版テキスト）",
    word_count: 6024,
    prompts_used: [
      { section: "overall_fortune", prompt: "...", tokens_used: 450 }
    ]
  },

  // メタデータ
  metadata: {
    processing_time_ms: 4200,
    model_version: "nozomi-v1.2.0",
    claude_model: "claude-sonnet-4.5-20250929",
    total_tokens: 8500,
    estimated_cost_usd: 0.42
  },

  created_at: ISODate("2026-01-03T12:00:00Z"),
  updated_at: ISODate("2026-01-03T12:00:04Z")
}

// Collection: user_feedback
// ユーザーフィードバック（品質改善用）

{
  _id: ObjectId("..."),
  reading_id: "rdg_12345",
  user_id: "usr_67890",
  rating: 5,  // 1-5
  feedback_text: "とても当たっていて驚きました！",
  helpful_sections: ["spiritual", "career"],
  created_at: ISODate("2026-01-03T14:00:00Z")
}
\`\`\`

**実装要件:**
1. PostgreSQL接続（pg / Prisma使用推奨）
2. MongoDB接続（mongoose使用）
3. マイグレーションスクリプト作成
4. シードデータ（テスト用ユーザー5件）

**期待される出力:**
- テーブル・コレクションが作成される
- \`npm run migrate\` でマイグレーション成功
- \`npm run seed\` でテストデータ挿入成功

---

### Phase 2: 占術エンジン実装（Week 3-5）

#### タスク 2.1: 西洋占星術モジュール

**指示:**
Swiss Ephemerisを使用して西洋占星術の計算を実装してください。

**ファイル:** \`divination-engine/modules/astrology.py\`

\`\`\`python
from datetime import datetime
import swisseph as swe
from typing import Dict, List, Tuple
import pytz

class AstrologyCalculator:
    """西洋占星術計算エンジン"""

    # 天体定義
    PLANETS = {
        'sun': swe.SUN,
        'moon': swe.MOON,
        'mercury': swe.MERCURY,
        'venus': swe.VENUS,
        'mars': swe.MARS,
        'jupiter': swe.JUPITER,
        'saturn': swe.SATURN,
        'uranus': swe.URANUS,
        'neptune': swe.NEPTUNE,
        'pluto': swe.PLUTO
    }

    # 星座名
    SIGNS = [
        "牡羊座", "牡牛座", "双子座", "蟹座", "獅子座", "乙女座",
        "天秤座", "蠍座", "射手座", "山羊座", "水瓶座", "魚座"
    ]

    # アスペクト定義（角度, 許容オーブ, 名称）
    ASPECTS = [
        (0, 8, "合(Conjunction)"),
        (60, 6, "六分(Sextile)"),
        (90, 8, "矩(Square)"),
        (120, 8, "三分(Trine)"),
        (180, 8, "衝(Opposition)")
    ]

    def __init__(self):
        # Swiss Ephemeris データパス設定
        swe.set_ephe_path('/usr/share/ephe')  # Ephemerisデータの場所

    def calculate(
        self,
        birth_date: datetime,
        birth_time: str,  # "HH:MM"
        latitude: float,
        longitude: float,
        timezone: str = "Asia/Tokyo"
    ) -> Dict:
        """
        西洋占星術の計算を実行

        Returns:
            {
                'planets': {...},
                'houses': {...},
                'aspects': [...],
                'elements': {...},
                'interpretation': {...}
            }
        """
        # タイムゾーン処理
        tz = pytz.timezone(timezone)
        dt_str = f"{birth_date.strftime('%Y-%m-%d')} {birth_time}"
        dt_local = datetime.strptime(dt_str, "%Y-%m-%d %H:%M")
        dt_aware = tz.localize(dt_local)
        dt_utc = dt_aware.astimezone(pytz.UTC)

        # ユリウス日計算
        jd = swe.julday(
            dt_utc.year, dt_utc.month, dt_utc.day,
            dt_utc.hour + dt_utc.minute / 60.0
        )

        # 天体位置計算
        planets = self._calculate_planets(jd)

        # ハウス計算
        houses = self._calculate_houses(jd, latitude, longitude)

        # アスペクト計算
        aspects = self._calculate_aspects(planets)

        # エレメント・クオリティ分析
        elements = self._analyze_elements(planets)
        qualities = self._analyze_qualities(planets)

        # 解釈生成
        interpretation = self._generate_interpretation(
            planets, houses, aspects, elements, qualities
        )

        return {
            'planets': planets,
            'houses': houses,
            'aspects': aspects,
            'elements': elements,
            'qualities': qualities,
            'interpretation': interpretation
        }

    def _calculate_planets(self, jd: float) -> Dict:
        """天体位置を計算"""
        results = {}
        for name, planet_id in self.PLANETS.items():
            pos, ret = swe.calc_ut(jd, planet_id)
            longitude = pos[0]
            sign_index = int(longitude / 30)
            degree_in_sign = longitude % 30

            results[name] = {
                'absolute_longitude': longitude,
                'sign': self.SIGNS[sign_index],
                'sign_index': sign_index,
                'degree': degree_in_sign,
                'retrograde': ret < 0
            }
        return results

    def _calculate_houses(
        self, jd: float, latitude: float, longitude: float
    ) -> Dict:
        """ハウスシステムを計算（Placidus方式）"""
        cusps, ascmc = swe.houses(jd, latitude, longitude, b'P')

        return {
            'ascendant': {
                'degree': ascmc[0],
                'sign': self.SIGNS[int(ascmc[0] / 30)]
            },
            'mc': {
                'degree': ascmc[1],
                'sign': self.SIGNS[int(ascmc[1] / 30)]
            },
            'houses': [
                {
                    'number': i + 1,
                    'cusp_degree': cusps[i],
                    'sign': self.SIGNS[int(cusps[i] / 30)]
                }
                for i in range(12)
            ]
        }

    def _calculate_aspects(self, planets: Dict) -> List[Dict]:
        """アスペクトを計算"""
        aspects = []
        planet_names = list(planets.keys())

        for i, p1_name in enumerate(planet_names):
            for p2_name in planet_names[i+1:]:
                p1_lon = planets[p1_name]['absolute_longitude']
                p2_lon = planets[p2_name]['absolute_longitude']

                angle = abs(p1_lon - p2_lon)
                if angle > 180:
                    angle = 360 - angle

                # アスペクトチェック
                for aspect_angle, orb, aspect_name in self.ASPECTS:
                    if abs(angle - aspect_angle) <= orb:
                        aspects.append({
                            'planet1': p1_name,
                            'planet2': p2_name,
                            'aspect': aspect_name,
                            'angle': aspect_angle,
                            'orb': abs(angle - aspect_angle),
                            'exact': abs(angle - aspect_angle) < 1
                        })
                        break

        return aspects

    def _analyze_elements(self, planets: Dict) -> Dict:
        """エレメント（火・地・風・水）の配分を分析"""
        elements = {'fire': 0, 'earth': 0, 'air': 0, 'water': 0}
        element_map = {
            0: 'fire', 1: 'earth', 2: 'air', 3: 'water',  # 牡羊-牡牛-双子-蟹
            4: 'fire', 5: 'earth', 6: 'air', 7: 'water',  # 獅子-乙女-天秤-蠍
            8: 'fire', 9: 'earth', 10: 'air', 11: 'water' # 射手-山羊-水瓶-魚
        }

        for planet in planets.values():
            sign_idx = planet['sign_index']
            element = element_map[sign_idx]
            elements[element] += 1

        return elements

    def _analyze_qualities(self, planets: Dict) -> Dict:
        """クオリティ（活動・不動・柔軟）の配分を分析"""
        qualities = {'cardinal': 0, 'fixed': 0, 'mutable': 0}
        quality_map = {
            0: 'cardinal', 1: 'fixed', 2: 'mutable', 3: 'cardinal',
            4: 'fixed', 5: 'mutable', 6: 'cardinal', 7: 'fixed',
            8: 'mutable', 9: 'cardinal', 10: 'fixed', 11: 'mutable'
        }

        for planet in planets.values():
            sign_idx = planet['sign_index']
            quality = quality_map[sign_idx]
            qualities[quality] += 1

        return qualities

    def _generate_interpretation(
        self, planets, houses, aspects, elements, qualities
    ) -> Dict:
        """基本的な解釈を生成（後でClaude APIで拡張）"""
        sun_sign = planets['sun']['sign']
        moon_sign = planets['moon']['sign']
        asc_sign = houses['ascendant']['sign']

        # 簡易解釈テンプレート
        interpretations = {
            "牡羊座": {"trait": "リーダーシップ、情熱的", "challenge": "衝動的"},
            "牡牛座": {"trait": "安定志向、忍耐強い", "challenge": "頑固"},
            # ... 他の星座も定義
        }

        return {
            'personality': f"太陽{sun_sign}のあなたは、{interpretations.get(sun_sign, {}).get('trait', '')}な性格です。",
            'inner_self': f"月{moon_sign}により、内面は{interpretations.get(moon_sign, {}).get('trait', '')}です。",
            'first_impression': f"アセンダント{asc_sign}により、第一印象は{interpretations.get(asc_sign, {}).get('trait', '')}と見られます。",
            'dominant_element': max(elements, key=elements.get),
            'strengths': [],
            'challenges': []
        }
```

**実装要件:**
1. Swiss Ephemerisライブラリのインストール（`pip install pyswisseph`）
2. Ephemerisデータファイルのダウンロード
3. ユニットテスト作成（5ケース以上）

**期待される出力:**
```python
# テスト実行例
calculator = AstrologyCalculator()
result = calculator.calculate(
    birth_date=datetime(1990, 3, 25),
    birth_time="14:30",
    latitude=35.6762,
    longitude=139.6503
)
print(result['planets']['sun'])  # {'sign': '牡羊座', 'degree': 4.5, ...}
```

---

#### タスク 2.2-2.6: 他の占術モジュール実装

**指示:**
同様の手法で以下のモジュールを実装してください。各モジュールの詳細仕様は「開発設計書」を参照。

- `four_pillars.py`: 四柱推命
- `tarot.py`: タロット（78枚のカードデータベース + ランダム選択）
- `numerology.py`: 数秘術（ライフパス、ディスティニーナンバー）
- `nine_star.py`: 九星気学（本命星、吉方位）
- `name_analysis.py`: 姓名判断（画数計算、81数理）

**実装要件:**
- 各モジュールはクラスベース
- 入力検証（バリデーション）
- エラーハンドリング
- ユニットテスト（各5ケース以上）

---

#### タスク 2.7: Orchestrator（統合制御）実装

**指示:**
6つの占術モジュールを統合し、並行実行する制御層を実装してください。

**ファイル:** `divination-engine/orchestrator.py`

```python
import asyncio
from typing import Dict, List
from datetime import datetime

from modules.astrology import AstrologyCalculator
from modules.four_pillars import FourPillarsCalculator
from modules.tarot import TarotCalculator
from modules.numerology import NumerologyCalculator
from modules.nine_star import NineStarCalculator
from modules.name_analysis import NameAnalysisCalculator

class DivinationOrchestrator:
    """占術エンジンの統合制御"""

    def __init__(self):
        self.calculators = {
            'astrology': AstrologyCalculator(),
            'four_pillars': FourPillarsCalculator(),
            'tarot': TarotCalculator(),
            'numerology': NumerologyCalculator(),
            'nine_star': NineStarCalculator(),
            'name_analysis': NameAnalysisCalculator()
        }

    async def calculate_all(
        self,
        user_data: Dict,
        divination_types: List[str] = None
    ) -> Dict:
        """
        複数の占術を並行実行

        Args:
            user_data: {
                'name': '田中花子',
                'birth_date': datetime(1990, 3, 25),
                'birth_time': '14:30',
                'latitude': 35.6762,
                'longitude': 139.6503,
                'timezone': 'Asia/Tokyo',
                'gender': 'female',
                'question': '今年の仕事運について'
            }
            divination_types: 使用する占術のリスト（Noneなら全て）

        Returns:
            全占術の結果を統合した辞書
        """
        if divination_types is None:
            divination_types = list(self.calculators.keys())

        # 並行実行タスクを作成
        tasks = []
        for div_type in divination_types:
            if div_type in self.calculators:
                task = self._run_divination(div_type, user_data)
                tasks.append(task)

        # 全タスクを並行実行
        results = await asyncio.gather(*tasks, return_exceptions=True)

        # 結果をまとめる
        combined_results = {}
        for div_type, result in zip(divination_types, results):
            if isinstance(result, Exception):
                combined_results[div_type] = {
                    'error': str(result),
                    'status': 'failed'
                }
            else:
                combined_results[div_type] = result

        return combined_results

    async def _run_divination(
        self, div_type: str, user_data: Dict
    ) -> Dict:
        """個別の占術を実行（非同期）"""
        calculator = self.calculators[div_type]

        # 各占術に応じた引数を準備
        if div_type == 'astrology':
            result = calculator.calculate(
                birth_date=user_data['birth_date'],
                birth_time=user_data['birth_time'],
                latitude=user_data['latitude'],
                longitude=user_data['longitude'],
                timezone=user_data.get('timezone', 'Asia/Tokyo')
            )

        elif div_type == 'four_pillars':
            result = calculator.calculate(
                birth_date=user_data['birth_date'],
                birth_time=user_data['birth_time'],
                gender=user_data.get('gender', 'unknown')
            )

        elif div_type == 'tarot':
            result = calculator.draw_cards(
                user_id=user_data.get('user_id', ''),
                birth_date=user_data['birth_date'],
                question=user_data.get('question', ''),
                spread='three_card'
            )

        elif div_type == 'numerology':
            result = calculator.calculate(
                birth_date=user_data['birth_date'],
                full_name=user_data.get('name', '')
            )

        elif div_type == 'nine_star':
            result = calculator.calculate(
                birth_date=user_data['birth_date']
            )

        elif div_type == 'name_analysis':
            result = calculator.calculate(
                full_name=user_data.get('name', '')
            )

        return result

# FastAPIエンドポイント（Flask/Expressと連携）
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()
orchestrator = DivinationOrchestrator()

class DivinationRequest(BaseModel):
    user_data: dict
    divination_types: list = None

@app.post("/divination/calculate")
async def calculate_divination(request: DivinationRequest):
    try:
        results = await orchestrator.calculate_all(
            user_data=request.user_data,
            divination_types=request.divination_types
        )
        return {"status": "success", "results": results}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

**実装要件:**
- FastAPI または Flask でREST API化
- 非同期処理（asyncio）
- エラーハンドリング
- ロギング（処理時間計測）

---

### Phase 3: AI出力生成エンジン（Week 6-7）

#### タスク 3.1: Claude API統合

**指示:**
Anthropic Claude APIを使用して、6000文字の鑑定レポートを生成してください。

**ファイル:** `backend/src/services/output-generator.ts`

```typescript
import Anthropic from '@anthropic-ai/sdk';

interface DivinationResults {
  astrology?: any;
  four_pillars?: any;
  tarot?: any;
  numerology?: any;
  nine_star?: any;
  name_analysis?: any;
  spiritual_vision?: any;
}

interface UserProfile {
  name: string;
  age: number;
  gender: string;
  question: string;
}

interface Section {
  name: string;
  targetLength: number;
  prompt: string;
}

export class OutputGenerator {
  private client: Anthropic;

  constructor(apiKey: string) {
    this.client = new Anthropic({ apiKey });
  }

  async generateReading(
    divinationResults: DivinationResults,
    userProfile: UserProfile
  ): Promise<string> {
    // セクション定義
    const sections: Section[] = [
      {
        name: 'header',
        targetLength: 200,
        prompt: this.buildHeaderPrompt(userProfile)
      },
      {
        name: 'overall_fortune',
        targetLength: 800,
        prompt: this.buildOverallFortunePrompt(divinationResults, userProfile)
      },
      {
        name: 'personality',
        targetLength: 500,
        prompt: this.buildPersonalityPrompt(divinationResults)
      },
      {
        name: 'love',
        targetLength: 600,
        prompt: this.buildLovePrompt(divinationResults, userProfile)
      },
      {
        name: 'career',
        targetLength: 600,
        prompt: this.buildCareerPrompt(divinationResults, userProfile)
      },
      {
        name: 'money',
        targetLength: 500,
        prompt: this.buildMoneyPrompt(divinationResults)
      },
      {
        name: 'health',
        targetLength: 400,
        prompt: this.buildHealthPrompt(divinationResults)
      },
      {
        name: 'spiritual',
        targetLength: 400,
        prompt: this.buildSpiritualPrompt(divinationResults, userProfile)
      },
      {
        name: 'monthly_forecast',
        targetLength: 400,
        prompt: this.buildMonthlyForecastPrompt(divinationResults)
      },
      {
        name: 'quarterly_forecast',
        targetLength: 400,
        prompt: this.buildQuarterlyForecastPrompt(divinationResults)
      },
      {
        name: 'yearly_forecast',
        targetLength: 400,
        prompt: this.buildYearlyForecastPrompt(divinationResults)
      },
      {
        name: 'action_plan',
        targetLength: 500,
        prompt: this.buildActionPlanPrompt(divinationResults)
      },
      {
        name: 'upsell',
        targetLength: 300,
        prompt: this.buildUpsellPrompt(userProfile)
      }
    ];

    // 各セクションを並行生成
    const sectionPromises = sections.map(section =>
      this.generateSection(section.prompt, section.targetLength)
    );

    const generatedSections = await Promise.all(sectionPromises);

    // セクションを結合
    const fullText = generatedSections.join('\n\n');

    // 文字数調整（6000文字±200）
    const adjustedText = this.adjustLength(fullText, 6000, 200);

    return adjustedText;
  }

  private async generateSection(
    prompt: string,
    targetLength: number
  ): Promise<string> {
    const maxTokens = Math.ceil(targetLength * 1.5); // 日本語は1文字≒1.5トークン

    const response = await this.client.messages.create({
      model: 'claude-sonnet-4.5-20250929',
      max_tokens: maxTokens,
      temperature: 0.7,
      messages: [
        {
          role: 'user',
          content: prompt
        }
      ]
    });

    return response.content[0].type === 'text'
      ? response.content[0].text
      : '';
  }

  private buildOverallFortunePrompt(
    results: DivinationResults,
    profile: UserProfile
  ): string {
    const summary = this.summarizeDivinationResults(results);

    return `
あなたはプロの占い師です。以下の占術結果を統合し、
ユーザーの総合運勢を800文字程度で解説してください。

【ユーザー情報】
- 名前: ${profile.name}
- 年齢: ${profile.age}
- 性別: ${profile.gender}
- 相談内容: ${profile.question}

【占術結果サマリー】
${summary}

【出力要件】
- 800文字前後（±50文字）
- 共感的で励ましのトーン
- 現在の運気の全体像を提示
- ユーザーの質問に関連づける
- 「〜でしょう」「〜かもしれません」など断定を避ける

【出力】
`;
  }

  private buildSpiritualPrompt(
    results: DivinationResults,
    profile: UserProfile
  ): string {
    return `
あなたは経験豊富な霊能者です。以下の占術結果を元に、
「見えてくるもの」「感じ取れるもの」を400文字程度で表現してください。

【占術結果】
${JSON.stringify(results, null, 2)}

【ユーザー情報】
${profile.question}

【出力要件】
- 400文字前後
- 「私には〜が見えます」「〜という感覚があります」などの表現
- 具体的すぎず、象徴的・抽象的な表現
- ポジティブな未来への示唆
- 光、扉、道、色、感情などのシンボルを使用

【出力】
`;
  }

  private buildUpsellPrompt(profile: UserProfile): string {
    return `
以下のユーザーに対して、追加の鑑定サービスへの誘導文を300文字で作成してください。

【ユーザー情報】
- 質問: ${profile.question}

【サービス】
1. 詳細鑑定（¥2,980）: ケルト十字タロット + 3年運勢グラフ
2. プレミアム会員（¥980/月）: 毎日の運勢 + 吉日カレンダー
3. 個別相談（¥9,800/60分）: プロ占い師とのオンラインセッション

【出力要件】
- 押し付けがましくない
- 「さらに詳しく知りたい方へ」というトーン
- 3つのサービスを簡潔に紹介
- CTAボタンのテキストは含めない（UIで別途表示）

【出力】
`;
  }

  private summarizeDivinationResults(results: DivinationResults): string {
    let summary = '';

    if (results.astrology) {
      summary += `- 西洋占星術: 太陽${results.astrology.planets.sun.sign}, 月${results.astrology.planets.moon.sign}\n`;
    }
    if (results.four_pillars) {
      summary += `- 四柱推命: ${results.four_pillars.strength}, 用神${results.four_pillars.favorable_god}\n`;
    }
    if (results.tarot) {
      summary += `- タロット: ${results.tarot.cards.map((c: any) => c.name).join(', ')}\n`;
    }
    // ... 他の占術も同様

    return summary;
  }

  private adjustLength(
    text: string,
    target: number,
    tolerance: number
  ): string {
    const currentLength = text.length;

    if (Math.abs(currentLength - target) <= tolerance) {
      return text;
    }

    // 長すぎる場合: 最後のセクション（アップセル）を短縮
    if (currentLength > target + tolerance) {
      const excess = currentLength - target;
      return text.slice(0, -excess);
    }

    // 短すぎる場合: 各セクションを若干拡張（再生成）
    // （実装は簡略化のため省略）
    return text;
  }
}
```

**実装要件:**
- Anthropic SDK のインストール
- 環境変数でAPI key管理
- トークン使用量のロギング
- エラーハンドリング（APIレート制限対応）

---

#### タスク 3.2: 文字数調整・品質チェック

**指示:**
生成されたテキストが6000文字±200文字の範囲に収まるように調整し、
禁止表現をフィルタリングする品質チェック機能を実装してください。

**ファイル:** `backend/src/services/quality-checker.ts`

```typescript
export class QualityChecker {
  private forbiddenPhrases = [
    '必ず〜します',
    'このままでは',
    '絶対に',
    '病気',
    '死',
    '離婚すべき',
    '訴訟',
    '株を買う',
    'ギャンブル'
    // ... 他の禁止表現
  ];

  check(text: string): { passed: boolean; issues: string[] } {
    const issues: string[] = [];

    // 文字数チェック
    if (text.length < 5800 || text.length > 6200) {
      issues.push(`文字数が範囲外: ${text.length}文字`);
    }

    // 禁止表現チェック
    for (const phrase of this.forbiddenPhrases) {
      if (text.includes(phrase)) {
        issues.push(`禁止表現を検出: "${phrase}"`);
      }
    }

    // ネガティブ度チェック（簡易実装）
    const negativeWords = ['不安', '失敗', '悪い', '危険'];
    const negativeCount = negativeWords.reduce(
      (count, word) => count + (text.match(new RegExp(word, 'g')) || []).length,
      0
    );

    if (negativeCount > 10) {
      issues.push(`ネガティブ表現が多すぎます: ${negativeCount}個`);
    }

    return {
      passed: issues.length === 0,
      issues
    };
  }
}
```

---

### Phase 4: アップセル機能実装（Week 8）

#### タスク 4.1: Stripe決済連携

**指示:**
Stripeを使用して、3種類のアップセルを実装してください。

**ファイル:** `backend/src/services/payment.ts`

```typescript
import Stripe from 'stripe';

export class PaymentService {
  private stripe: Stripe;

  constructor(secretKey: string) {
    this.stripe = new Stripe(secretKey, {
      apiVersion: '2024-12-18.acacia'
    });
  }

  // Type 1: 詳細鑑定（一回購入）
  async createDetailedReadingCheckout(
    userId: string,
    readingId: string
  ): Promise<string> {
    const session = await this.stripe.checkout.sessions.create({
      payment_method_types: ['card'],
      line_items: [
        {
          price_data: {
            currency: 'jpy',
            product_data: {
              name: '詳細鑑定レポート',
              description: 'ケルト十字タロット + 3年運勢グラフ'
            },
            unit_amount: 2980
          },
          quantity: 1
        }
      ],
      mode: 'payment',
      success_url: `https://nozomi.example.com/reading/${readingId}?payment=success`,
      cancel_url: `https://nozomi.example.com/reading/${readingId}?payment=cancel`,
      metadata: {
        user_id: userId,
        reading_id: readingId,
        type: 'detailed_reading'
      }
    });

    return session.url!;
  }

  // Type 2: サブスクリプション
  async createSubscriptionCheckout(userId: string): Promise<string> {
    const session = await this.stripe.checkout.sessions.create({
      payment_method_types: ['card'],
      line_items: [
        {
          price: 'price_xxxxx',  // Stripeで事前作成したPrice ID
          quantity: 1
        }
      ],
      mode: 'subscription',
      success_url: `https://nozomi.example.com/subscription/success`,
      cancel_url: `https://nozomi.example.com/subscription/cancel`,
      metadata: {
        user_id: userId,
        type: 'premium_subscription'
      }
    });

    return session.url!;
  }

  // Webhook処理
  async handleWebhook(
    payload: string | Buffer,
    signature: string
  ): Promise<void> {
    const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!;
    const event = this.stripe.webhooks.constructEvent(
      payload,
      signature,
      webhookSecret
    );

    switch (event.type) {
      case 'checkout.session.completed':
        await this.handleCheckoutCompleted(event.data.object);
        break;

      case 'invoice.payment_succeeded':
        await this.handleSubscriptionRenewal(event.data.object);
        break;

      case 'invoice.payment_failed':
        await this.handlePaymentFailed(event.data.object);
        break;
    }
  }

  private async handleCheckoutCompleted(session: any): Promise<void> {
    const { user_id, reading_id, type } = session.metadata;

    if (type === 'detailed_reading') {
      // DBのreadingレコードをtier='detailed'に更新
      await db.readings.update({
        where: { reading_id },
        data: { tier: 'detailed' }
      });
    } else if (type === 'premium_subscription') {
      // ユーザーのサブスクリプション情報を更新
      await db.users.update({
        where: { user_id },
        data: {
          subscription_tier: 'premium',
          subscription_started_at: new Date(),
          subscription_expires_at: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
        }
      });
    }
  }
}
```

**実装要件:**
- Stripe Test Mode で動作確認
- Webhook endpoint実装（`/api/webhooks/stripe`）
- 決済ログの保存
- エラーハンドリング

---

### Phase 5: フロントエンド実装（Week 9-10）

#### タスク 5.1: Next.js UI実装

**指示:**
美しく使いやすいUIを実装してください。

**主要ページ:**
1. `/` - ランディングページ
2. `/reading/new` - 鑑定入力フォーム
3. `/reading/[id]` - 鑑定結果表示
4. `/subscription` - サブスク登録ページ
5. `/consultation/book` - 個別相談予約ページ

**実装例（鑑定入力フォーム）:**

```tsx
// frontend/src/app/reading/new/page.tsx

'use client';

import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  name: z.string().min(1, '名前を入力してください'),
  birthDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, '正しい形式で入力'),
  birthTime: z.string().regex(/^\d{2}:\d{2}$/, 'HH:MM形式'),
  birthPlace: z.string().min(1, '出生地を入力'),
  gender: z.enum(['male', 'female', 'other']),
  question: z.string().min(10, '10文字以上で入力してください')
});

type FormData = z.infer<typeof schema>;

export default function NewReadingPage() {
  const [isSubmitting, setIsSubmitting] = useState(false);
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema)
  });

  const onSubmit = async (data: FormData) => {
    setIsSubmitting(true);

    try {
      const response = await fetch('/api/readings', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data)
      });

      const { reading_id } = await response.json();
      window.location.href = `/reading/${reading_id}`;
    } catch (error) {
      alert('エラーが発生しました');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-3xl font-bold mb-8">無料鑑定を受ける</h1>

      <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
        <div>
          <label className="block mb-2">お名前</label>
          <input
            {...register('name')}
            className="w-full border p-2 rounded"
            placeholder="山田 花子"
          />
          {errors.name && <p className="text-red-500">{errors.name.message}</p>}
        </div>

        <div>
          <label className="block mb-2">生年月日</label>
          <input
            {...register('birthDate')}
            type="date"
            className="w-full border p-2 rounded"
          />
          {errors.birthDate && <p className="text-red-500">{errors.birthDate.message}</p>}
        </div>

        {/* 他のフィールドも同様 */}

        <button
          type="submit"
          disabled={isSubmitting}
          className="w-full bg-purple-600 text-white py-3 rounded-lg hover:bg-purple-700 disabled:opacity-50"
        >
          {isSubmitting ? '鑑定中...' : '無料で鑑定する'}
        </button>
      </form>
    </div>
  );
}
```

**実装要件:**
- レスポンシブデザイン
- ローディング状態の表示
- エラーハンドリング
- アクセシビリティ（ARIA属性）

---

### Phase 6: テスト・デプロイ（Week 11-12）

#### タスク 6.1: テスト実装

**指示:**
包括的なテストスイートを作成してください。

**テスト種類:**
1. ユニットテスト（Jest / pytest）
2. 統合テスト（API E2E）
3. E2Eテスト（Playwright）
4. 品質テスト（鑑定結果の妥当性）

**実装例:**

```typescript
// backend/tests/output-generator.test.ts

import { OutputGenerator } from '../src/services/output-generator';

describe('OutputGenerator', () => {
  let generator: OutputGenerator;

  beforeAll(() => {
    generator = new OutputGenerator(process.env.ANTHROPIC_API_KEY!);
  });

  it('should generate 6000-character reading', async () => {
    const mockResults = {
      astrology: { /* ... */ },
      tarot: { /* ... */ }
    };

    const mockProfile = {
      name: 'テスト太郎',
      age: 30,
      gender: 'male',
      question: '今年の運勢'
    };

    const reading = await generator.generateReading(mockResults, mockProfile);

    expect(reading.length).toBeGreaterThan(5800);
    expect(reading.length).toBeLessThan(6200);
  });

  it('should not contain forbidden phrases', async () => {
    // ... テスト実装
  });
});
```

#### タスク 6.2: デプロイ

**指示:**
Docker + Kubernetesでデプロイしてください。

**Dockerfile例:**

```dockerfile
# frontend/Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package*.json ./
RUN npm ci --production
EXPOSE 3000
CMD ["npm", "start"]
```

**Kubernetes deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nozomi-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nozomi-frontend
  template:
    metadata:
      labels:
        app: nozomi-frontend
    spec:
      containers:
      - name: frontend
        image: gcr.io/PROJECT_ID/nozomi-frontend:latest
        ports:
        - containerPort: 3000
        env:
        - name: NEXT_PUBLIC_API_URL
          value: "https://api.nozomi.example.com"
```

---

## 最終チェックリスト

実装完了前に、以下をすべて確認してください:

- [ ] 全6大占術が正しく動作する
- [ ] 霊視AIモジュールが統合されている
- [ ] 6000文字±200文字の出力が生成される
- [ ] 3種類のアップセル機能が動作する
- [ ] Stripe決済が正常に処理される
- [ ] フロントエンドUIが美しく使いやすい
- [ ] API応答時間が5秒以内（P95）
- [ ] テストカバレッジ80%以上
- [ ] 禁止表現フィルターが機能する
- [ ] エラーハンドリングが適切
- [ ] ログが適切に記録される
- [ ] セキュリティ対策（SQL injection, XSS等）
- [ ] GDPR/個人情報保護法対応
- [ ] Docker環境で起動できる

---

## サポートリソース

### ドキュメント
- [開発設計書](./NOZOMI_DESIGN_DOCUMENT.md)
- [API仕様書](./API_SPECIFICATION.md)
- [デプロイガイド](./DEPLOYMENT_GUIDE.md)

### 外部リンク
- [Anthropic Claude API Docs](https://docs.anthropic.com/)
- [Swiss Ephemeris Documentation](https://www.astro.com/swisseph/)
- [Stripe API Reference](https://stripe.com/docs/api)
- [Next.js Documentation](https://nextjs.org/docs)

---

**最後に:**

このマスタープロンプトは、Nozomiシステムの完全な実装ガイドです。
各フェーズを着実に進め、高品質なAI鑑定システムを構築してください。

不明点があれば、開発設計書を参照するか、各ライブラリの公式ドキュメントを確認してください。

🌸 **Nozomi - 希望を照らすAI鑑定システム** 🌸

頑張ってください！
```
