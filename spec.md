# 老後資金ギャップ（65歳→死亡）モンテカルロ・シミュレーター（単身） 現行仕様（実装版）

更新日: 2026-02-22  
実装形態: **単一HTML（サーバー不要） + Web Worker**  
目的: **年金等の収入では賄いきれない支出（不足額）**を、数万回のランダム試行で分布として推計する

---

## 1. 主要アウトプット（必須）

### 1.1 年金ギャップ（不足額）
- 定義（試行ごと）  
  月次で `gap_month = max(0, expense_month - income_month)`  
  `gap_total = Σ gap_month`
- 集計（全試行）
  - 平均 / 中央値 / 最小 / 最大
  - p10 / p25 / p75 / p90 / p95（参考）

### 1.2 破綻確率（任意だが実装済）
- 初期資産を入れた場合、月次で資産を更新し `assets < 0` になった試行の割合
- 破綻は「一度でも 0 未満になったら ruined=1（継続）」として扱う

### 1.3 参考指標（実装済）
- 平均死亡年齢
- 年次還付（外来年間合算 / 医療・介護合算）の平均
- 医療自己負担（総）平均
- 介護自己負担（総）平均
- 実行時間、Gompertzパラメータ

### 1.4 分布可視化（実装済）
- `log10(gap_total + 1)` のヒストグラム（50 bin、レンジ 0〜8）

---

## 2. 時間・状態モデル

### 2.1 時間刻み
- **月次**
- 開始: 65歳0か月（t=0）
- 終了: 死亡月（t=deathMonth）

### 2.2 状態（試行ごと）
- 生存: `alive`
- 介護状態: `inCare`（介護開始後、終了月まで継続）
- 医療ショック状態: `shockRemainingMonths > 0` の間「入院等（高額）」が追加で発生

---

## 3. 寿命（死亡年齢）モデル：Gompertz（簡易キャリブレーション）

### 3.1 入力
- 性別: `male / female / avg`
- `leMale, leFemale`: 65歳時点の平均余命（年）
- `surv90Male, surv90Female`: 90歳まで生存する確率
- `survAge`（既定 90）
- `maxAge`（既定 110、上限クリップ）

### 3.2 キャリブレーション
- Gompertz生存関数: `S(t)=exp(-(b/c)*(exp(c t)-1))`（tは65歳からの経過年）
- `mean remaining life` と `S(t0)=survProbAtAge`（t0=survAge-65）から `b,c` を数値的に決定（2条件合わせ）
- 乱数で残余寿命を逆関数サンプルし `deathAge = 65 + remYears`

### 3.3 健康相関による死亡率補正（実装済）
- 試行ごとに潜在健康リスク `z ~ Normal(0, healthSigma)` をサンプル
- 死亡ハザード側の補正: `b' = b * exp(healthToMortK * z)`  
  → 不健康（z>0）ほど早死にしやすい（平均寿命が短くなる）

---

## 4. 収入モデル（年金など）

### 4.1 月次収入
- `income_month = pension_month + other_income_month + refunds_month`
- `refunds_month` は年次還付を年度末（または死亡月）にまとめて加算（タイムラグ無し近似）

### 4.2 年金改定（簡易）
- 毎月: `pension_month *= (1 + pensionGrowth)^(1/12)`
- **現行実装では otherIncomeMonthly も同じ改定率で増加**（必要なら別レート化が拡張点）

---

## 5. 支出モデル（基本）

### 5.1 生活費（毎月）
- 65-69基準: `livingMonthly`
- インフレ: `livingMonthly * (1+livingInflation)^(m/12)`
- 年齢スケール（ON時）
  - 70-79: `* scale70s`
  - 80+: `* scale80p`

### 5.2 住居費（毎月）
- `housingType = rent / owner_house / owner_condo`
- 賃貸: `rentMonthly * housingInfl`
- 持家: `propertyTaxMonthly * housingInfl`
- マンション: 上に加えて `condoMonthly * housingInfl`

### 5.3 葬儀（死亡月に一時）
- `funeralCost * generalInfl`

---

## 6. 医療モデル（自己負担：高額療養費テーブル + 多数回該当 + 外来上限）

医療は「保険診療（高額療養費の上限対象）」と「保険外（上限対象外）」に分離。

### 6.1 保険診療（10割）生成
- 外来（ベース）:  
  `outInsured10 = LogNormal(median=medOutMedian, sigma=medOutSigma) * medicalInfl * medMult`
- 入院等（ショック）:  
  - 年確率 `medShockProb` を月確率に変換し、年齢倍率（75+/80+/85+）と健康倍率を掛けて発火
  - 期間: 平均 `medShockMonthsMean` の指数分布近似で月数をサンプル（最低1）
  - 発生月は毎月 `inInsured10 = LogNormal(medShockMedian, medShockSigma) * medicalInfl * medMult`
- 健康倍率（相関）:
  - `medMult = exp(healthToMedK * z)`
  - `shockProbMult = exp(healthToShockK * z)`

### 6.2 窓口負担
- `copayRate` は年齢で切替
  - 65-69: `copay6569`
  - 70-74: `copay7074`
  - 75+: `copay75p`
- 外来: `outRaw = outInsured10 * copayRate`
- 入院等: `inRaw = inInsured10 * copayRate`

### 6.3 70歳以上の外来上限（月）
- 所得区分が `GEN/L2/L1` の場合に外来部分へ適用:
  - GEN: `outCap=18,000`
  - L2/L1: `outCap=8,000`
- `outPaid = min(outRaw, outCap)`、入院はそのまま `inRaw`
- `totalAfterOutCap = outPaid + inRaw`

### 6.4 高額療養費（月次上限）
- 高額療養費ONの場合、月次で `oopInsured = min(totalAfterOutCap, cap_month)`
- `cap_month` は年齢・所得区分で決定（以下の埋め込みテーブル）
- **多数回該当**（ON時）  
  過去12か月の「高額療養費が発動した月」をリングバッファで追跡し、  
  `直近12か月で3回以上発動 → 4回目以降` の月は `multiCap` を適用（該当区分のみ）

#### 6.4.1 70歳未満（A〜E）テーブル（埋め込み）
- A: `252,600 + (総医療費 - 842,000)*1%` / 多数回 `140,100`
- B: `167,400 + (総医療費 - 558,000)*1%` / 多数回 `93,000`
- C: `80,100  + (総医療費 - 267,000)*1%` / 多数回 `44,400`
- D: 固定 `57,600` / 多数回 `44,400`
- E: 固定 `35,400` / 多数回 `24,600`

#### 6.4.2 70歳以上（G3/G2/G1/GEN/L2/L1）テーブル（埋め込み）
- G3: `252,600 + (総医療費 - 842,000)*1%` / 多数回 `140,100`
- G2: `167,400 + (総医療費 - 558,000)*1%` / 多数回 `93,000`
- G1: `80,100  + (総医療費 - 267,000)*1%` / 多数回 `44,400`
- GEN: 固定 `57,600` / 多数回 `44,400` / 外来上限 `18,000` / 外来年間上限 `144,000`
- L2: 固定 `24,600` / 外来上限 `8,000`
- L1: 固定 `15,000` / 外来上限 `8,000`

> 注: 実制度は外来/入院・世帯合算・食事代など細部があるが、本実装は「月上限」と「70+外来上限」を中心に再現。

### 6.5 保険外（上限対象外）
- `nonCovered = LogNormal(medNonCoveredMedian, medNonCoveredSigma) * medicalInfl`
- 医療自己負担（総）:
  - `medOOP = oopInsured + nonCovered`

### 6.6 外来年間合算（70歳以上・一般のみの近似）
- ON時、かつ `70歳以上 & 所得区分=GEN` のとき
- **外来のみの月（入院ショックがない月）**を対象に `outPaid` を年度内で積算
- 年度末（または死亡月）に `max(0, outAnnualSum - 144,000)` を還付として収入へ加算
- 還付は年度末に一括受給（タイムラグ無し近似）

---

## 7. 介護モデル（自己負担：1〜3割 + 高額介護サービス費）

介護は「介護保険サービス（上限対象）」と「食費/居住費等（上限対象外）」に分離。

### 7.1 介護の発生・開始・継続
- 生涯発生確率: `careLifetimeProb`（健康補正あり）
  - `probAdj = careLifetimeProb * exp(healthToCareProbK * z)` を 0..1 にクリップ
- 開始年齢:
  - `startAge ~ Normal(meanAdj, careStartSd)`
  - `meanAdj = careStartMean - healthToCareStartShift * z`
  - 65〜死亡年齢にクリップ
- 継続期間（月）:
  - `durationMonths ~ LogNormal(median=careDurationMedian, sigma=careDurationSigma)`、最低1
  - 終了は `min(start + duration, deathMonth+1)`
- 施設フラグ:
  - `facility ~ Bernoulli(careFacilityProb)`

### 7.2 月次介護費（介護期間中）
#### 7.2.1 介護保険サービス（10割）
- `svcTotal10 = LogNormal(careServiceMedian, careServiceSigma) * careInfl`

#### 7.2.2 窓口負担（1〜3割）
- `copayRaw = svcTotal10 * careCopayRate`（0.1/0.2/0.3）

#### 7.2.3 高額介護サービス費（月上限）
- ON時: `careServiceCopay = min(copayRaw, careCap)`
- 埋め込み上限（careCapTierで選択）:
  - T5: 140,100
  - T4: 93,000
  - T3: 44,400
  - T2: 24,600
  - T1: 15,000

#### 7.2.4 上限対象外（食費/居住費等）
- 在宅: `LogNormal(careNonCovHomeMedian, careNonCovSigma) * careInfl`
- 施設: `LogNormal(careNonCovFacilityMedian, careNonCovSigma) * careInfl`
- 介護開始月に `careOneTime`（一時費用）を上限対象外に加算

#### 7.2.5 介護自己負担（総）
- `careOOP = careServiceCopay + careNonCovered`

---

## 8. 年次合算（高額医療・高額介護合算療養費）※任意ON

### 8.1 年度
- `fiscalStartMonth`（既定8）で年度を定義（8月→翌7月）
- 年度末 or 死亡月に清算（還付を収入へ加算）

### 8.2 対象となる合算額（近似）
- 年度内に支払った「上限対象分」の自己負担の合計を `coveredAnnualSum` として追跡
- 現行実装では **以下を加算**:
  - 医療: `oopInsured`（高額療養費適用後、保険外を除く）
  - 介護: `careServiceCopay`（高額介護適用後、食費居住費等を除く）

### 8.3 年次上限（埋め込み）
- 70歳未満（comboTierUnder70）:
  - A: 2,120,000
  - B: 1,410,000
  - C: 670,000
  - D: 600,000
  - E: 340,000
- 70歳以上（comboTier70p）:
  - G3: 2,120,000
  - G2: 1,410,000
  - G1: 670,000
  - GEN: 560,000
  - L2: 310,000
  - L1: 190,000

### 8.4 還付
- 年度末: `refund += max(0, coveredAnnualSum - annualCap)`
- その月の `income_month` に加算
- 年度の `coveredAnnualSum` は 0 にリセット

---

## 9. 突発イベントモデル（起きやすい/起きにくい）

### 9.1 イベント定義（UI編集可）
各イベントは以下のパラメータ:
- `enabled`（ON/OFF）
- `annualProb`（年確率）
- `median, sigma`（費用の対数正規分布）
- `cooldown`（月、連発防止）
- `applies`（rent/owner/all）

### 9.2 発火
- 月確率へ変換: `pM = 1 - (1 - annualProb)^(1/12)`
- cooldown中は発火しない
- 発火したら費用: `LogNormal(median, sigma) * generalInfl`
- 住居タイプにより適用判定（rent/owner）

---

## 10. 資産（任意）

- 初期資産: `initialAssets`
- 月次更新:
  - `assets += (income_month - expense_month)`
  - 運用ON時: 月次リターン `r ~ Normal(muM, sigM)` を適用
    - `muM = (1+assetReturn)^(1/12) - 1`
    - `sigM = assetVol / sqrt(12)`
    - `assets *= (1 + r)`
- `assets < 0` になった最初の時点で `ruined=1`

---

## 11. 月次の計算フロー（擬似コード）


for trial in 1..N:
z ~ Normal(0, healthSigma)

deathAge = sampleGompertz(bexp(healthToMortKz), c)
deathMonth = floor((deathAge-65)*12)

careSchedule = maybeStartCare(z, deathAge)

hceRing[12]=0; hceCount=0; idx=0
outAnnualSum=0
coveredAnnualSum=0
totalRefund=0

assets = initialAssets
gap = 0

for m in 0..deathMonth:
age = 65 + m/12

income = pension + otherIncome

living = livingMonthly * livingInfl^m * (ageScale)
housing = housingByType * housingInfl^m

// 医療：外来+入院
outInsured10 = LN(medOutMedian, medOutSigma) * medicalInfl^m * exp(healthToMedK*z)
inInsured10  = shockIfAny(...) * medicalInfl^m * exp(healthToMedK*z)
oopInsured = applyHCE(outInsured10, inInsured10, copayRate, tiers, manyTimes, outCap)
nonCovered = LN(medNonCoveredMedian, medNonCoveredSigma) * medicalInfl^m
medOOP = oopInsured + nonCovered

// 介護：開始後は継続
careOOP = 0
if inCare:
  svc10 = LN(careServiceMedian, careServiceSigma) * careInfl^m
  svcCopay = min(svc10*careCopayRate, careCapTierCap) (if ON)
  nonCov = LN(home/facility median, careNonCovSigma)*careInfl^m + oneTimeIfStart
  careOOP = svcCopay + nonCov

// 年次合算対象（近似）
coveredAnnualSum += oopInsured + svcCopay

// 突発イベント
eventCost = sum(events triggered this month)

// 死亡月
funeral = (m==deathMonth) ? funeralCost*generalInfl^m : 0

// 年度末還付（外来年間合算 + 合算療養費）
if fiscalYearEnd(m) or m==deathMonth:
  refund = calcRefund(outAnnualSum, coveredAnnualSum, caps)
  income += refund
  totalRefund += refund
  reset annual accumulators

expense = living + housing + medOOP + careOOP + eventCost + funeral

gap += max(0, expense - income)

if useAssets:
  assets += income - expense
  assets *= (1 + monthlyReturn)
  if assets<0: ruined=1

pension *= pensionInfl
otherIncome *= pensionInfl

---

## 12. UI（入力項目一覧：現行）

### 12.1 実行設定
- 試行回数 `nTrials`
- 乱数シード `seed`
- 性別 `sex`

### 12.2 収入
- 年金タイプ（プリセット）/ 年金月額 `pensionMonthly`
- その他収入 `otherIncomeMonthly`
- 年金改定率 `pensionGrowth`

### 12.3 生活費・住居
- 生活費月額 `livingMonthly`
- 生活費インフレ `livingInflation`
- 年齢スケール `useAgeScale, scale70s, scale80p`
- 住居タイプ `housingType`
- 家賃/管理費/固定資産税 等
- 住居インフレ `housingInflation`

### 12.4 医療
- 窓口負担（65-69/70-74/75+）
- 高額療養費ON `useHCE`
- 多数回該当ON `useManyTimes`
- 所得区分（70未満） `hceTierUnder70`
- 所得区分（70以上） `hceTier70p`
- 外来ベース分布（中央値/σ）
- 入院ショック（年確率/平均継続/中央値/σ）
- 保険外分布（中央値/σ）
- 医療インフレ `medicalInflation`
- 外来年間合算ON `useOutAnnualCap`

### 12.5 介護
- 介護ON `useCare`
- 生涯発生確率 `careLifetimeProb`
- 介護負担割合 `careCopayRate`（1〜3割）
- 高額介護ON `useCareCap` と所得区分 `careCapTier`
- 開始年齢（平均/SD）
- サービス10割分布（中央値/σ）
- 上限対象外（在宅/施設中央値、σ）
- 一時費用 `careOneTime`
- 継続期間（中央値/σ）
- 施設移行確率 `careFacilityProb`
- 介護インフレ `careInflation`

### 12.6 健康相関
- `healthSigma`
- `healthToMedK`
- `healthToShockK`
- `healthToCareProbK`
- `healthToCareStartShift`
- `healthToMortK`

### 12.7 突発イベント
- イベント別 ON/OFF、年確率、中央値、σ、クールダウン、適用範囲
- 一般インフレ `generalInflation`
- 葬儀費用 `funeralCost`

### 12.8 年次合算（任意）
- 合算ON `useCombinedCap`
- 年度開始月 `fiscalStartMonth`（既定8）
- 所得区分（70未満） `comboTierUnder70`
- 所得区分（70以上） `comboTier70p`

### 12.9 寿命モデル
- `leMale, leFemale, surv90Male, surv90Female, survAge, maxAge`

### 12.10 出力（重い）
- `keepResults`（試行ごとの配列保持 → CSV可能）

---

## 13. データ出力（実装）

### 13.1 画面表示
- KPI（平均/中央値/最小/最大）
- 破綻確率
- 追加統計（p10/p25/p75/p90/p95、平均死亡年齢、平均還付、医療/介護平均、実行時間）

### 13.2 CSV（keepResults ON時）
列:
- `trial`
- `gap_total_yen`
- `ruined`
- `death_age`
- `total_refund`
- `total_med_oop`
- `total_care_oop`

---

## 14. 既知の簡略化 / 注意点（現行）

- 税・社会保険料（国保/後期/介護保険料）を厳密には計算していない（生活費に含める前提）
- 高額療養費は世帯合算/外来と入院の厳密な分離、食事代、医療機関別の集計などを省略
- 年次還付（外来年間合算/合算療養費）は **年度末に一括受給**（タイムラグ無し近似）
- 介護の補足給付（食費/居住費軽減）や住宅改修・福祉用具購入の保険給付は未実装（上限対象外として近似）
- 入力したテーブル定数は時期により改定されうるため、必要なら将来更新する設計が望ましい

---

## 15. 変更履歴（要約）

- v0: ベース（生活費・住居・医療・介護・突発イベント・葬儀・資産・Gompertz寿命）
- v1（現行）:
  - 高額療養費：70未満A〜E、70以上区分、外来上限、**多数回該当**
  - 介護：1〜3割 + **高額介護サービス費（月上限）**
  - 年次：外来年間合算（一般 14.4万円） + 医療・介護合算療養費（任意ON）
  - 医療×介護×寿命の相関：潜在健康リスク `z` を導入