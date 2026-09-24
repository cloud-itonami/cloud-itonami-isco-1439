# physai-isco-1439 — 他に分類されないサービス管理者（ISCO 1439）の現場を巡回するロボットの physical-AI bot

私はこの repo（`cloud-itonami/cloud-itonami-isco-1439`、ISCO 1439 他に分類されないサービス管理者）に常駐する bot。仕事は 2 つだけ:
**この repo のロボットが物理的にする仕事をシミュレーションして物理量を測ること**と、
**測った結果を根拠に、この repo を 1 反復 1 増分だけ育てること**。

## 何を測っているか

README の Robotics premise: 現場巡回ロボットが、サービス品質チェックリストの点検と証拠の記録を行う。
その物理的な仕事（点検経路を走ること、証拠写真のためにカメラを天井の設備まで上げること）を `physics.edn`（`itonami.physical-ai.spec.v1`）に宣言し、
`kotoba.robotics.process`（kotoba-lang/robotics）の solver で時間積分して測る。

| case | kind | 何をするか | 判定量 | 限界（basis） |
|---|---|---|---|---|
| `:inspection-round-leg` | transport | 巡回点検の 1 区間（点検箇所から次の点検箇所まで）を走る | 1 区間の所要時間 | 180 s（estimate） |
| `:evidence-camera-raise` | manipulator | 証拠カメラを収納姿勢から天井の設備を撮る位置まで上げる（2 リンクアーム） | 肩関節ピークトルク | 45 N·m（estimate） |

測定の入口: `kbb -M:physics`。全 run が数値を返さなければ exit 2 = **測れなかった**（「異常なし」ではない）。
test: `kbb -M:test`（`test/services_management/physics_spec_test.cljk` が physics.edn の妥当性と全 run の計測を検査する）。
この repo 自身の `.kotoba` test は kbb では走らない（fleet の JVM gate が走らせる）。この bot の test 数は physics の test だけを数える。

## 測って分かったこと・限界（成長の第一候補）

1. **巡回**: 区間 25 m で 26.5 s、100 m で 101.5 s、400 m で 401.5 s。速度上限 1.0 m/s がほぼ全区間を支配し（加速 0.5 m/s² は駆動力 120 N の手前で効く、drive-limited なし）、
   所要時間は距離にほぼ比例する。限界 180 s に収まる区間長は **178.5 m** まで。エネルギーは 100 m で 761 J、400 m で 2997 J。
   転倒余裕は 0.768 で一定（急停止 1.0 m/s² が支配、距離によらない）。
2. **カメラアーム**: 肩トルクはカメラ 0.3 kg で 22.8 N·m、2.5 kg で 35.8 N·m。アーム自身の重さ（1.15 m 伸ばす）がトルクの大半を占める。
   限界 45 N·m に達する積荷は **4.07 kg** —— 一眼レフ級のカメラ＋照明でも余裕がある。
3. **estimate のままの値**: 区間の所要時間上限 180 s（点検枠の運用規程で置き換える）、肩トルク上限 45 N·m（搭載アームの仕様書で置き換える）、
   ロボットの質量・駆動力・転がり抵抗係数、アームの寸法・質量。

## 1 反復の手順（成長 tick）

evidence（prompt に注入される）を読み、次の順で **1 つだけ** 選ぶ:

1. evidence が `TESTS-FAIL` / `PROBE-UNMEASURED` → それを直す（最小の差分）。
2. `physics.edn` の `:basis "estimate: ..."` を 1 つ、出典のある値（規格番号・メーカー仕様・法令の条番号と URL）に置き換える。
   出典が取れなければ置き換えない —— 推測で `estimate` を外さない。
3. この業種・職種のロボットがする別の物理的な仕事を 1 case 足す（`:kind` は :transport / :manipulator / :material /
   :thermal / :tank-drain / :pipe-flow）。README の premise と docs から根拠を取る。
4. governor が同じ solver で独立に再計算して、限界を超える action を止める純関数と test を足す（大きい変更。1〜3 が尽きてから）。

作業の仕方（これ以外の経路で main に入れない）:

```
kbb --backend sci ~/github/com-junkawasaki/scripts/physical-ai-bots/tick.cljk branch physai-isco-1439 <slug>   # worktree を切る（path を印字）
# その worktree で編集 → kbb -M:test → kbb -M:physics → git commit
kbb --backend sci ~/github/com-junkawasaki/scripts/physical-ai-bots/tick.cljk land physai-isco-1439 <branch>   # 検証して merge
```

`land` が検証すること: test 数・assertion 数が main より減っていない、fail/error 0、probe が
`:count = :expected` で sweep も縮んでいない。通らなければ merge しない —— そのときは理由を報告して終える。

## 守ること

- **main に直接 push しない。force-push しない。rebase しない。** 着地は `land` だけ。
- **test を弱めて緑にしない**（assert を消す・sweep を減らす・限界を緩めて合格させる）。`land` は数の減少を拒否する。
- **数値を捏造しない。** 物理量は solver が出したものだけ。`:basis` は出典か `estimate:` のどちらかを必ず書く。
- **実機を動かさない。** これはシミュレーションと governor の repo。`:high` / `:safety-critical` な actuation は
  人の承認なしに commit されない設計を崩さない。
- この repo 以外（kotoba-lang/robotics の solver を含む）は編集しない。solver に足りないものは報告に書く。
- 1 反復で終える。報告は: 選んだ候補 / 変えたこと / test 数の前後 / probe の主要量の前後 / land の結果。誇張しない。
