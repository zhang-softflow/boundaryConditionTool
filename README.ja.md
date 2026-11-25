# foamBC — OpenFOAM 境界条件の検索・雛形生成・チェックツール

[English](README.md) | 日本語

OpenFOAM（ESI 版、openfoam.com）には、有限体積法（fvPatchField）だけで約250種類、point（pointPatchField、動的メッシュ用）で約40種類の境界条件があります。使うたびに「type 名は何か」「必須項目は何か」「この場（U/p/k/…）にはどれを使うのか」「入口・出口・壁の定番の組み合わせは何か」をソースや tutorials から探すのは手間がかかります。
**foamBC** は、境界条件として登録されている全クラスのヘッダの doc コメント（`Description`、`Usage`、`\verbatim` の記述例、`\table` のパラメータ表）と、tutorials の 0/ ファイル約3000個の `boundaryField` を解析してデータベースを作ります。そのデータベースを使って、検索、表示、雛形の生成、**パッチの役割ごとの全場一括生成（recipe）**、**ケースのチェック（check）** ができます。

functionObject 用の [foamFO](../functionObjects)、fvOptions 用の [foamFV](../fvOptions) と同じ使い方です。

| ファイル | 内容 |
|---|---|
| `foamBC` | CLI（Python 3.8 以上、標準ライブラリのみ） |
| `catalog.html` | 検索できるカタログ。ブラウザで開くだけで使えます（オフライン、単一ファイル） |
| `bcDatabase.json` | 同梱のデータベース（OpenFOAM-v2606 から生成） |
| `catalog_template.html` | カタログの雛形 |
| `skill/openfoam-boundaryconditions/` | Claude Code 用スキル |
| `install.sh` | インストール補助（スキルのリンク、PATH の設定） |

## インストール

```bash
cd boundaryConditions   # このディレクトリ
./install.sh            # ~/.claude/skills にスキルをリンクし、PATH 設定の1行を表示
./install.sh --bashrc   # ~/.bashrc に PATH を追記する場合（バックアップあり）
./install.sh --uninstall
```

使う OpenFOAM が v2606 以外の場合は、その環境を読み込んでからデータベースを作ってください（数秒で終わります）。

```bash
source /usr/lib/openfoam/openfoam2412/etc/bashrc
foamBC build            # -> ~/.local/share/foamBC/bcDatabase-openfoam2412.json
```

- **探索順**: `$FOAMBC_DB` → `~/.local/share/foamBC/bcDatabase-<版>.json` → 同梱の `bcDatabase.json`
- **版が違う場合**: 同梱のデータベースの版と環境の版が違うと警告が出ます（`FOAMBC_QUIET=1` で抑止できます）。

## 使い方

### 1. ケースのチェック

```bash
foamBC check -case .
```

`constant/polyMesh/boundary` のパッチ（名前と型）と、0/（無ければ 0.orig/）の全フィールドの `boundaryField` を突き合わせ、パッチ×場の一覧と問題点を表示します。エラーがあると終了コード 1 を返します。

検出するもの:
- パッチに対応するエントリが無い（正規表現 `"(inlet|outlet)"`、パッチグループ、`#includeEtc "caseDicts/setConstraintTypes"`、`#include` も解釈します）
- 存在しない type 名（綴り間違いには候補を表示）
- 場の型に無い境界条件（例: `totalPressure` を U に使う）
- constraint 型の不一致（例: `empty` パッチに `zeroGradient`、`patch` 型に `symmetryPlane`）
- 型が `wall` でないパッチへの壁関数（nut 系はソースで abort するためエラー、k/epsilon/omega 系は警告）
- ヘッダの表で必須とされているキーワードの欠落（警告）
- メッシュに無いパッチ名のエントリ（警告）

### 2. パッチの役割ごとに全場を一括生成（recipe）

```bash
foamBC recipe                                     # 役割の一覧
foamBC recipe velocityInlet  -case . -patch inlet  --U '(10 0 0)' --I 0.05 --L 0.01
foamBC recipe pressureOutlet -case . -patch outlet --p 0 --insert
foamBC recipe wall           -case . -patch heater --Tw 350 --insert
foamBC recipe velocityInlet  --fields U,p,k,omega,nut --U 5      # ケースが無い場合
```

| 役割 | 内容 |
|---|---|
| `velocityInlet` | 速度固定の入口（`--U` にベクトル、またはスカラーで面法線方向の流入）。k/epsilon/omega は乱流強度 `--I` と混合長 `--L` から |
| `flowRateInlet` | 体積流量 `--Q` [m3/s] または質量流量 `--mdot` [kg/s] の入口 |
| `pressureOutlet` | 静圧固定の出口（`--p`）。速度とスカラーは逆流に強い `inletOutlet` |
| `totalPressureInlet` | 全圧固定の入口（`--p0`） |
| `openAtmosphere` | 大気開放（流入・流出の両方。interFoam の atmosphere など） |
| `farfield` | 外部流の遠方境界（freestream 系） |
| `wall` / `movingWall` / `slipWall` | 静止壁（壁関数）/ 接線方向に動く壁 / 滑り壁 |
| `symmetry` | constraint 型（メッシュのパッチ型を全場に使用: symmetryPlane, empty, wedge, cyclic …） |

- 0/ にある全フィールドについて、場の名前と次元から種類を判定して記述を作ります（U、p、p_rgh、k、epsilon、omega、nut、nuTilda、alphat、T、alpha.*、その他のスカラー・ベクトル）。
- `p_rgh` がある場合は p_rgh 系（p は `calculated`、壁・入口の p_rgh は `fixedFluxPressure`、出口は `prghPressure`）。`0/p` の次元が Pa なら圧縮性として扱います（`compressible::alphatWallFunction`）。出口・大気開放の圧力は、`--p`/`--p0` を省略すると、非圧縮では 0、圧縮性では `0/p` の internalField の値になります。
- メッシュのパッチ型が constraint 型の場合は自動で `symmetry` 役割に切り替えます。`wall` 役割で型が `wall` でない場合は警告します。
- 各エントリのコメントに、tutorials でその役割・その場に使われた回数を表示します。tutorials で最も多い型が別にある場合は、それも表示します。
- internalField が `nonuniform` の場（setFields の後など）では、`$internalField` を使いません（サイズ不一致のエラーになるため）。
- `--insert` は、プレースホルダ `<…>` が残っていると書き込みません。

### 3. 個別の境界条件

| コマンド | 説明 |
|---|---|
| `foamBC search 大気開放` | 名前、説明、パラメータ名、同義語を全文検索します（`流量`、`全圧`、`壁関数`、`熱伝達率`、`時間変化`、`周期` なども可） |
| `foamBC list [-g wallFunction] [-t inlet] [--type vector] [-u]` | 一覧（T=tutorials に例あり、W=wall 型パッチ専用、w=壁関数、C=constraint 型）。`-u` は tutorials での使用回数順 |
| `foamBC list --groups` | グループ別の件数 |
| `foamBC typical k [-r inlet]` | その場に tutorials で使われている型を、パッチの役割（パッチ名から推定）ごとに集計して表示 |
| `foamBC show totalPressure [--inherited]` | 説明、記述例、パラメータ表（必須は赤、条件付きは紫）、値の型、継承元、壁専用か、tutorials での使用状況 |
| `foamBC example inletOutlet -f k -m 3` | tutorials 内の実際の記述を抜き出して表示（OpenFOAM 環境または `--src` が必要） |
| `foamBC add externalWallHeatFluxTemperature -patch heater [--full]` | 1パッチ分の雛形を表示 |
| `foamBC add fixedValue -case . -patch inlet -field U --insert` | `0/U` の該当パッチを置き換え（無ければ追加）。元のファイルは `U.bak` |
| `foamBC show solidBodyMotionDisplacement --point` | point 境界条件（`pointDisplacement` など）は `--point` |
| `foamBC build [--src DIR] [-o FILE]` | データベースを作ります |
| `foamBC html` | `catalog.html` を作り直します |

雛形の作り方:
- ヘッダの記述例のコメント（`// Mandatory entries`、`// Optional entries` など）とパラメータ表の Required 列から、必須項目を取り出します。
- 継承元のヘッダ（`atmBoundaryLayer.H`、`temperatureCoupledBase.H` など）の表で必須とされている項目を加えます。親の境界条件（inletOutlet など）の項目は、派生クラス側で設定されることが多いので加えません。
- `-field` に vector の場を指定すると、`uniform 0` を `uniform (0 0 0)` にします。

## catalog.html

ブラウザで開くだけで使えます。OpenFOAM がインストールされていなくても見られます。

- **Catalog タブ**: インクリメンタル検索（`/` キーで検索欄へ、日本語の同義語も可）、グループ・値の型・tutorials での使用・point の絞り込み。右側に雛形（Copy ボタン付き）、ヘッダの記述例、パラメータ表、継承元、tutorials での使用状況（場・パッチの役割）
- **Typical per field タブ**: 場（U、p、p_rgh、k …）を選ぶと、入口・出口・壁などの役割ごとに tutorials でよく使われる型を表示します。recipe の役割の一覧もあります。
- **項目を直接開く**: `catalog.html#totalPressure`、`catalog.html#point:solidBodyMotionDisplacement`

## Claude Code スキル

`install.sh` で `~/.claude/skills/openfoam-boundaryconditions` にリンクされます。
「入口は 5 m/s、出口は大気開放、壁は滑りなしにしたい」のように頼むと、`foamBC check` でケース（パッチの型、p か p_rgh か、乱流モデルの場）を確認し、`recipe` と `show` の内容に基づいて全場の境界条件を書きます。名前で呼ぶ場合は `/openfoam-boundaryconditions` です。

## OpenFOAM の既存機能

```bash
foamSearch $FOAM_TUTORIALS boundaryField.inlet.type U      # tutorials の inlet パッチの U の type を集計
foamDictionary 0/U -entry boundaryField -expand               # 構文確認（#include、$ を展開）
simpleFoam -listScalarBCs / -listVectorBCs                    # ソルバで使える境界条件名の一覧（説明なし）
cat $WM_PROJECT_DIR/etc/caseDicts/setConstraintTypes          # constraint 型を一括設定する include ファイル
```
- `foamSearch` はパッチ名を指定する必要があり、`-listScalarBCs` は名前しか表示しません。foamBC は説明・必須項目・使用例・組み合わせまで表示します。
- `codedFixedValue` のコードの雛形は `etc/codeTemplates/dynamicCode/fixedValueFvPatchFieldTemplate.{C,H}` にあります。

## 制限と注意
- 対応しているのは **ESI 版（openfoam.com, vYYMM）** です。Foundation 版（openfoam.org）の境界条件（`flowRateInletVelocity` の `profile` など）は対象外です。
- finite-area（faPatchField）と面の場（fvsPatchField）は対象外です。
- パッチの役割（inlet/outlet/wall …）の集計は、tutorials のパッチ名から推定したものです。
- check は静的な解析です。`$変数` や `#include` を含むエントリは、展開できる範囲で確認します。式や `#codeStream` の中身は評価しません。
- ソルバ固有の境界条件のうち、ソースツリー外のライブラリ（`libs` で読み込むユーザーのライブラリ）は DB にありません。check では「unknown」と表示されます。
- 動作確認の範囲:
  - DB の生成: v2606（ソース）、v2512、v2012（パッケージ）
  - tutorials の全ケース（602）に check を実行し、残るエラーが実在しない型名（`compressible::kqRWallFunction` など旧名称）だけであることを確認
  - recipe → `--insert` → 実行: v2512 の simpleFoam/pitzDaily（velocityInlet + pressureOutlet）、interFoam/damBreak（openAtmosphere、setFields 後）

## ライセンス
GPL-3.0-or-later（[LICENSE](LICENSE)）。
同梱の `bcDatabase.json` と `catalog.html` には、OpenFOAM のソース（GPL-3.0-or-later、© OpenCFD Ltd. ほか）の doc コメントの文章が含まれます。

OPENFOAM® is a registered trade mark of OpenCFD Limited, producer and distributor of the OpenFOAM software via www.openfoam.com. This offering is not approved or endorsed by OpenCFD Limited.
