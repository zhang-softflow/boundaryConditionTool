---
name: openfoam-boundaryconditions
description: Write, choose or check OpenFOAM (ESI/openfoam.com) boundary conditions in the boundaryField of 0/ field files — e.g. "inlet velocity 10 m/s and outlet open to the atmosphere", "set wall functions for k-omega SST", "which BC for a mass flow inlet", "turbulence intensity at the inlet", "why does my case fail with a boundary condition error", "check my 0/ files". OpenFOAM（ESI版）の境界条件（0/ の boundaryField）を書く・選ぶ・確認するときに使う。「入口は流速10 m/s、出口は大気開放にしたい」「k-omega SST の壁関数を設定して」「質量流量で入口を指定したい」「乱流強度を入口に与えたい」「境界条件のエラーで止まる」「0/ をチェックして」のように、目的を自然文で受け取り、ソースのヘッダと tutorials に基づいて全フィールド分の正しい記述を作る。
---

# OpenFOAM 境界条件設定支援

`foamBC` は、fvPatchField / pointPatchField として登録されている全境界条件のヘッダ（Description/Usage/パラメータ表）と、tutorials の 0/ ファイル（どの場・どのパッチにどの型が使われているか）から作ったデータベースを使うツールです。
このスキルでは、**記述をすべて foamBC が返す内容（＝ソースと tutorials の記述）に基づいて書く**ことを原則とします。記憶や推測で type 名やキーワード名を書かないでください（例: Foundation 版の `flowRateInletVelocity` にある `profile` は ESI 版にはありません）。

ツールの場所:
- `foamBC` が PATH にあればそれを使います（`command -v foamBC` で確認）。
- PATH に無い場合は、このスキルのベースディレクトリから見て `../../foamBC` を使います（リポジトリ同梱）。

データベース:
- 同梱の DB は OpenFOAM-v2606 から作ったものです。
- ユーザーの OpenFOAM の版が違い、「bundled database is for ...」という警告が出た場合は、OpenFOAM 環境を source した状態で `foamBC build` を実行します（数秒）。作られた DB はその版で自動的に使われます。
- `example` コマンドで tutorials の本文を表示するには、OpenFOAM 環境（`$WM_PROJECT_DIR`）が必要です。環境が無い場合は `--src <OpenFOAMのルート>` を指定します。

回答はユーザーの言語で行います。

## 手順

1. **ケースを確認する**（ケースがある場合）
   - `foamBC check -case <case>` で、メッシュのパッチ名と型（`constant/polyMesh/boundary`）、0/ の各場の現在の境界条件、問題点を一覧します。メッシュが無い場合は、パッチ名を 0/ から読みます（パッチの型は不明になります）。
   - 次の点を確認します。
     - **圧力の場**: `p_rgh` があれば p_rgh 系のソルバ（p は全パッチ `calculated`）。`0/p` の dimensions が `[0 2 -2 ...]` なら非圧縮の動圧（m2/s2、基準 0）、`[1 -1 -2 ...]` なら Pa（基準 1e5 など）。
     - **乱流モデル**: 0/ にある場（k, epsilon, omega, nut, nuTilda など）と `constant/turbulenceProperties`。
     - **パッチの型**: 壁関数（nut*WallFunction など）は型が `wall` のパッチにしか使えません。`empty`/`symmetryPlane`/`wedge`/`cyclic` などの constraint 型のパッチには、同じ名前の境界条件を全場に書きます。
   - ケースが無い場合は、ソルバ・乱流モデル・場の名前をユーザーに確認するか、`--fields U,p,k,epsilon,nut` のように指定します。

2. **パッチの役割ごとに全場の記述を作る**（基本はこれを使う）
   - `foamBC recipe` で役割の一覧を表示します: velocityInlet / flowRateInlet / pressureOutlet / totalPressureInlet / openAtmosphere / farfield / wall / movingWall / slipWall / symmetry。
   - `foamBC recipe <role> -case <case> -patch <patch> [--U '(10 0 0)' | --U 10] [--p 0] [--p0 0] [--T 300] [--Tw 350] [--I 0.05] [--L 0.01] [--Q 0.001 | --mdot 0.1] [--alpha 1]` で、0/ の全場の記述を表示します（`--insert` を付けない限り書き込みません）。
     - `--U` にスカラーを与えると、面法線方向の流入（`surfaceNormalFixedValue`、負が流入）になります。
     - 乱流の入口値は、強度 `--I` と混合長 `--L`（目安: 水力直径の 0.07 倍）から計算する境界条件を使います。
     - 出力のコメントに、tutorials での使用回数と、tutorials で最も多い別の型（あれば）が出ます。ユーザーの条件に合う方を選びます。
   - 残った `<…>` のプレースホルダは、ユーザーに値を確認して埋めます。
   - recipe に無い役割や特殊な条件は、手順3で個別に選びます。

3. **個別の境界条件を選ぶ・確認する**
   - `foamBC typical <field> [-r inlet]` で、その場によく使われる型を役割ごとに確認します。
   - `foamBC search <キーワード...>` で候補を探します（英語でも日本語でも可。例: `流量`, `大気開放`, `全圧`, `壁関数`, `熱伝達率`, `時間変化`, `周期`, `heat flux`, `wall function`）。`foamBC list -g <group>`（basic / constraint / derived / wallFunction / turbulence / thermal / atmospheric / wave / multiphase ...）、`-t <tag>`、`--type vector` も参考にします。
   - よく使う対応（手がかり。必ず show で裏付けを取る）:
     - 時間変化する入口値 → `uniformFixedValue`（Function1: table, sine など）、外部データ・実測分布 → `timeVaryingMappedFixedValue`
     - 壁の熱流束・熱伝達率・外気温 → `externalWallHeatFluxTemperature`、勾配指定 → `fixedGradient`
     - 流体と固体の熱連成（CHT）→ `compressible::turbulentTemperatureCoupledBaffleMixed` / `...RadCoupledMixed`
     - 大気境界層の風の入口 → `atmBoundaryLayerInletVelocity` / `...InletK` / `...InletEpsilon`（`libs (atmosphericModels)` が必要な場合あり）
     - 回転壁 → `rotatingWallVelocity`、ファン → `fan`（cyclic パッチ）/ `fanPressure`、多孔質バッフル → `porousBafflePressure`
     - 無反射出口（圧縮性・音響）→ `waveTransmissive` / `advective`
     - 式や自作コード → `exprFixedValue` / `codedFixedValue`
     - 動的メッシュの `pointDisplacement` は point 境界条件です（`--point` を付けて検索・表示）。
   - `foamBC show <type> [--inherited]` で記述例、パラメータ表（Reqd=yes が必須、cond は条件付き）、使える値の型（scalar/vector…）、壁専用か、継承元を確認します。
   - `foamBC example <type> -f <field> -m 3` で tutorials の実例を確認します。
   - `foamBC add <type> -case <case> -patch <patch> -field <field> [--full]` で1パッチ分の雛形を表示します（`--insert` で 0/<field> に書き込み）。
   - 情報が足りない場合は、`show` に表示されるヘッダのパスのファイルを直接 Read して確認します。

4. **ケースに書き込む**（ユーザーが書き込みを求めた場合のみ）
   - `foamBC recipe ... --insert` または `foamBC add ... --insert` を使います。該当パッチのエントリを置き換え、無ければ boundaryField の末尾に追加します。元のファイルは `<field>.bak` に保存されます（最初の1回だけ）。
   - プレースホルダが残っていると `recipe --insert` は書き込みません。値を渡してから実行します。
   - 書き込み後は `foamBC check -case <case>` でエラー0件を確認します。OpenFOAM 環境がある場合は、`foamDictionary 0/U -entry boundaryField -expand` で構文を確認します。
   - `0.orig/` しか無いケース（Allrun でコピーするもの）は `0.orig/` に書きます。`-time 0` などで対象を指定できます。

## 注意点（よくある間違い）
- `value $internalField;` は、internalField が `nonuniform`（setFields、mapFields の後など）のときはサイズが合わずエラーになります。foamBC はこの場合に置き換えます。
- 出口の速度に `zeroGradient` だけを使うと逆流で発散しやすいので、`inletOutlet`（inletValue (0 0 0)）や `pressureInletOutletVelocity` を検討します。
- 非圧縮ソルバの p は動圧（p/ρ）です。圧力を Pa で与えないように注意します。
- p_rgh 系のソルバでは、壁と速度入口の p_rgh は `fixedFluxPressure`、p は `calculated` です。
- `foamBC check` の「header lists as required」は、ヘッダの表に基づく警告です。継承元で既定値がある場合もあるので、`show` で確認してから判断します。

## 出力のしかた
- ユーザーの言語で説明し、0/ のファイルごとに boundaryField のエントリを示します（または書き込んだ結果と check の結果を示します）。
- 各パッチの役割、選んだ型の理由、値と単位（m/s、m2/s2 または Pa、K）、残っているプレースホルダを添えます。
