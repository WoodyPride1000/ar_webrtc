P2P Location Bond — AR Co-location


# 📝 設計書: P2P Location Bond (TDGLモデル)

## 1. 概要

### 1.1. システム目的
本システムは、**WebRTC** を通じて二者間の **位置情報（GPS）** を交換し、そのデータ（距離・精度）を **ギンツブルグ＝ランダウ（GL）理論** の **秩序パラメータ ($\Psi$)** の動態としてリアルタイムに表現する拡張現実（AR）アプリケーションです。

「絆の強さ（秩序）」は、距離と通信品質に依存する**実効温度 ($T_{\text{eff}}$)**に基づき、**時間依存GL (TDGL) 方程式**に従って動的に変化します。

### 1.2. 主要技術スタック

| 分野 | 技術要素 | 目的 |
| :--- | :--- | :--- |
| **AR/3D描画** | **A-Frame, AR.js** | WebAR環境構築、3D描画、GPSに基づくオブジェクト配置。 |
| **位置情報** | **Geolocation API, Haversine** | ローカル位置の取得、正確な地球上距離の計算。 |
| **通信** | **WebRTC (DataChannel)** | サーバーレスの低遅延P2P通信チャネル。 |
| **物理モデル** | **TDGL方程式 (JavaScript実装)** | 秩序パラメータ $\Psi$ の動的な時間進化シミュレーション。 |

---

## 2. アーキテクチャとデータフロー

### 2.1. アーキテクチャ構成
システムは、**I/O層**、**物理/フィルタ層**、**表示/AR層**の3層で構成されます。

| 層 | コンポーネント | 機能 |
| :--- | :--- | :--- |
| **I/O層** | Geolocation, WebRTC, DOM UI | 位置情報・SDP・データチャネル通信の入出力、ユーザー操作受付。 |
| **物理/フィルタ層** | Haversine, Decoherence, TDGL | 位置の平滑化フィルタリング、ノイズ・距離に基づく**実効温度の計算**、**$\Psi$ の時間発展**。 |
| **表示/AR層** | A-Frame, THREE.js | $\Psi$ の結果に基づく**動的な視覚効果**（ジッター、回転、ライン）をARシーンに描画。 |

### 2.2. メインデータフロー（`updateVisuals`関数）

1.  **入力処理**: リモート位置 $P_{\text{remote}}$、ローカル精度 $A_{\text{local}}$、リモート精度 $A_{\text{remote}}$ を受信。
2.  **ノイズ・距離計算**: $\mathbf{P}_{\text{filtered}}$ とローカル位置 $P_{\text{local}}$ から $\text{distance}$ を算出。GPS精度と距離から $\text{decoherenceNoise}$ を導出。
3.  **TDGL進化**: $\text{distance}$ と $\text{decoherenceNoise}$ を外部パラメータとして、**`evolveOrderParameterStep`** を実行し、新たな $\mathbf{\Psi}$ を計算。
4.  **出力**: $\Psi$ から $\mathbf{P}(\text{Block})$ を計算し、DOM (確率表示、距離表示) とARシーンに反映。

---

## 3. 物理モデル (TDGL) 詳細設計

### 3.1. 秩序パラメータ $\Psi$
* **変数名**: `psi`
* **役割**: 二者間の「絆の強さ」を示す無次元量。$\Psi \approx 1$ で完全な秩序状態。
* **時間発展方程式 (離散化)**:
$$\mathbf{\Psi}_{\text{new}} = \mathbf{\Psi}_{\text{old}} + dt \cdot \left[ -\Gamma \left( a \Psi + b \Psi^3 \right) + \eta \right]$$

### 3.2. 実効温度 $T_{\text{eff}}$（無秩序化の駆動源）
実効温度は、TDGL方程式の係数 $a$ の符号を決定し、相転移を駆動します。

$$\mathbf{T}_{\text{eff}} = \mathbf{T}_0 + \alpha_{\text{distance}} \cdot \text{distance} + \alpha_{\text{noise}} \cdot \text{decoherenceNoise}$$

* **TDGL係数 $a$**: $\mathbf{a} = a_0 (T_{\text{eff}} - T_c)$
* **パラメータ**:
    * `T0` (基底温度): 0.5
    * `Tc` (臨界温度): 1.0

### 3.3. ブロック発生確率 $P(\text{Block})$
秩序パラメータ $\Psi$ が低いほど、指数関数的に確率が高くなります。

$$\mathbf{P}(\text{Block}) = \mathbf{P}_{\max} \cdot e^{-\beta_P \Psi^2}$$

* **パラメータ**:
    * `P_max`: $0.5$
    * `betaP`: $5.0$

---

## 4. UI/UXと通信設計

### 4.1. シグナリングと通信

* **シグナリング方式**: **手動**（SDPテキスト交換）。`#controlPanel` に統合。
* **データチャンネル**: **`DataChannel`** を使用。2秒間隔で位置情報（緯度経度、精度）を送信。
* **デバッグ/ステータス**: `#statusPanel` で接続状態、GPS精度、送受信タイムスタンプを詳細表示。

### 4.2. 視覚表現と $\Psi$ の依存性

ARシーンのすべてのダイナミクスは $\Psi$ の値に依存し、TDGLモデルの結果を視覚化します。

| 視覚要素 | 依存性 | $\Psi$ が低い場合（無秩序） | $\Psi$ が高い場合（秩序） |
| :--- | :--- | :--- | :--- |
| **ターゲット位置** | $\propto \text{decoherenceNoise}$ | 大きな**位置ジッター**が発生。 | 位置が安定。 |
| **ターゲット回転** | $\propto 1 - \Psi$ | ランダム性を持つ揺らぎ回転。 | 安定したスムーズ回転。 |
| **結合線（Bond）** | $\propto 1 - \Psi$ | **不透明度**が低下し、大きく揺らぐ。 | 不透明度が上がり、安定して視認できる。 |
