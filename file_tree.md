# ArduPilot リポジトリのファイル構造調査

ここではArduPilotリポジトリのファイル構造、特に自己位置推定に関連する部分を調査します。

## ファイル構造

```
/f:/googledrive/git/ardupilot/
├── libraries/                  # ライブラリ群
│   ├── AP_AHRS/               # 姿勢推定システム (Attitude Heading Reference System)
│   ├── AP_Baro/               # 気圧センサーライブラリ
│   ├── AP_Compass/            # 磁気センサーライブラリ
│   ├── AP_DAL/                # データ抽象化レイヤー
│   ├── AP_GPS/                # GPSライブラリ
│   ├── AP_InertialSensor/     # IMUセンサーライブラリ 
│   ├── AP_Math/               # 数学ライブラリ
│   ├── AP_NavEKF/             # EKF1 (初期実装の拡張カルマンフィルタ)
│   ├── AP_NavEKF2/            # EKF2 (第2世代拡張カルマンフィルタ)
│   ├── AP_NavEKF3/            # EKF3 (現行の拡張カルマンフィルタ)
│   │   ├── AP_NavEKF3.cpp     # メインのEKF3実装
│   │   ├── AP_NavEKF3.h       # EKF3のヘッダーファイル
│   │   ├── AP_NavEKF3_core.cpp # EKF3のコア実装
│   │   ├── AP_NavEKF3_core.h   # EKF3コアのヘッダーファイル
│   │   ├── AP_NavEKF3_Measurements.cpp # センサー測定値処理
│   │   ├── AP_NavEKF3_MagFusion.cpp    # 磁気センサーフュージョン
│   │   ├── AP_NavEKF3_VehicleStatus.cpp # 車両状態処理
│   │   ├── AP_NavEKF3_GyroBias.cpp     # ジャイロバイアス推定
│   │   ├── AP_NavEKF3_Outputs.cpp      # 出力処理
│   │   └── ...
│   ├── AP_OpticalFlow/        # 光学フローセンサーライブラリ
│   ├── AP_RangeFinder/        # 距離センサー(ライダー、超音波)ライブラリ
│   └── AP_Vehicle/            # 車両タイプ定義ライブラリ
│
├── APMrover2/                 # 地上車両向けのアプリケーションコード
└── ...
```

## 利用しているモジュール調査

ArduPilotでは以下の主要なセンサーモジュールを活用しています：

1. **IMU (慣性計測装置)**
   - ライブラリ: `AP_InertialSensor`
   - 対応デバイス: MPU6000, MPU9250, ICM20xxx, BMI160など多数
   - `AP_InertialSensor.cpp/h`でドライバーインターフェース定義

2. **気圧センサー**
   - ライブラリ: `AP_Baro`
   - 対応デバイス: MS5611, BMP280, BMP388, LPS2Xなど
   - 高度推定に利用

3. **磁気センサー (コンパス)**
   - ライブラリ: `AP_Compass`
   - 対応デバイス: HMC5883L, QMC5883L, MMC3416, IST8310など
   - 方位推定に利用

4. **GPS**
   - ライブラリ: `AP_GPS`
   - 対応プロトコル: NMEA, UBX, SiRF, NOVA, MTKなど
   - 位置・速度のリファレンスとして使用

5. **光学フローセンサー**
   - ライブラリ: `AP_OpticalFlow`
   - 視覚的な速度推定に使用

6. **距離センサー (レンジファインダー)**
   - ライブラリ: `AP_RangeFinder`
   - 超音波、レーザーなどの距離センサー
   - 高度推定やマッピングに使用

## IMUのバイアスオフセット、温度特性補正機能

IMUのバイアスオフセットと温度特性補正は主に以下のファイルで実装されています：

1. **バイアス推定と補正**:
   - `AP_NavEKF3_GyroBias.cpp`: ジャイロスコープのバイアス推定
   - `AP_NavEKF3_core.cpp`内の`resetGyroBias`および`resetAccelBias`関数
   - `AP_NavEKF3_core.cpp`内の各種バイアス処理コード

```cpp
// ジャイロバイアス推定の例 (AP_NavEKF3_GyroBias.cpp)
// ジャイロバイアスの処理ノイズパラメータ
AP_GROUPINFO("GBIAS_P_NSE", 26, NavEKF3, _gyroBiasProcessNoise, GBIAS_P_NSE_DEFAULT),

// 加速度計バイアスの処理ノイズパラメータ
AP_GROUPINFO("ABIAS_P_NSE", 28, NavEKF3, _accelBiasProcessNoise, ABIAS_P_NSE_DEFAULT),

// バイアス制限値
AP_GROUPINFO("ACC_BIAS_LIM", 48, NavEKF3, _accBiasLim, 1.0f),
```

2. **温度補正**:
   - `AP_InertialSensor`内に温度補正機能
   - IMUセンサーに応じた温度特性補正カーブ
   - キャリブレーションデータに基づく補正

## カルマンフィルタのソースコード

拡張カルマンフィルタは主に`AP_NavEKF3`ディレクトリに実装されています：

1. **コアアルゴリズム**:
   - `AP_NavEKF3_core.cpp`: 拡張カルマンフィルタの主要なアルゴリズム
   - `AP_NavEKF3_Measurements.cpp`: 測定モデル
   - `AP_NavEKF3_Predictions.cpp`: 状態予測モデル

2. **センサーフュージョン**:
   - `AP_NavEKF3_MagFusion.cpp`: 磁気センサーデータの融合
   - `AP_NavEKF3_OptFlowFusion.cpp`: 光学フローデータの融合
   - `AP_NavEKF3_VelPosFusion.cpp`: 速度・位置データの融合
   - `AP_NavEKF3_AirDataFusion.cpp`: 気圧データの融合

3. **特殊機能**:
   - `AP_NavEKF3_EKFGSF_yaw.cpp`: GPS速度に基づく方位推定（GPSなしでも運用可能）

主要なカルマンフィルタ数式処理は以下の関数で行われています：
- `UpdateFilter()`: メインフィルタ更新
- `predictState()`: 状態予測ステップ 
- `predictCovariance()`: 共分散予測ステップ
- 各種`FuseXXX()`関数: 測定値の融合

## 地上を動くロボット向けのカルマンフィルタ

ArduPilotは地上を動くロボット（ローバー）用の設定をEKF3に含んでいます：

1. **ローバー向け設定**:
   ```cpp
   // AP_NavEKF3.cpp内のローバー用パラメータデフォルト値
   #elif APM_BUILD_TYPE(APM_BUILD_Rover)
   // rover defaults
   #define VELNE_M_NSE_DEFAULT     0.5f
   #define VELD_M_NSE_DEFAULT      0.7f
   #define POSNE_M_NSE_DEFAULT     0.5f
   #define ALT_M_NSE_DEFAULT       2.0f
   #define MAG_M_NSE_DEFAULT       0.05f
   // ...他のローバー用パラメータ
   ```

2. **車輪オドメトリー**:
   - `AP_NavEKF3`には車輪オドメトリーのサポートがあります
   ```cpp
   void NavEKF3::writeWheelOdom(float delAng, float delTime, uint32_t timeStamp_ms, const Vector3f &posOffset, float radius)
   ```

3. **ローバーアプリケーション**:
   - `APMrover2`ディレクトリに地上ロボット向けのアプリケーションコード
   - ローバー向けのEKF設定と利用方法が実装されています

4. **3次元/2次元の制約**:
   - ローバーモードでは、地上制約（2Dモード）を使用して高度方向の位置推定精度を向上
   - 垂直速度の制約と地面基準の高度測定

## まとめ

ArduPilotのEKF3は非常に柔軟で多機能な実装であり、以下の特徴があります：

1. **多センサー対応**: IMU、GPS、気圧計、磁気センサー、光学フロー、視覚オドメトリー、レンジファインダーなど
2. **バイアス補正**: 高度なバイアス推定と補正機能
3. **フォールトトレラント**: マルチコア設計による冗長性
4. **プラットフォーム適応**: 航空機、マルチコプター、ローバー向け設定
5. **GPS非依存**: 光学フローや視覚オドメトリーによるGPS非依存動作

特に地上ロボット向けには、車輪オドメトリー、特化したノイズモデル、2D動作モードなどが実装されており、GPS無しでも自己位置推定が可能になっています。
