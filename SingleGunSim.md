# Single-particle gun から SiEmcalTopoTree を作る

この文書は single-particle generator から detector simulation、reconstruction、`SiEmcalTopoTree` 作成までを説明します。Machine Learning の学習・評価は [`ElectronSeparation_MachineLearning/README.ja.md`](../ElectronSeparation_MachineLearning/README.ja.md) に分離しています。

## 先に結論

現行の `run_singlev2.sh` と `run_single.sh` は `Fun4All_singleParticle_SiliconPositionv4.C` を呼びます。一回の実行で generation、Geant4、reconstruction、DST、analyzer ROOT、flat tree、ML feature 追加まで行います。

## ファイルの役割

### Single gun

| 種類 | ファイル | 役割 |
|---|---|---|
| electron wrapper | [`macros/run_singlev2.sh`](macros/run_singlev2.sh#L1) | particle を `e-` に固定して Position v4 を実行 |
| pion wrapper | [`macros/run_single.sh`](macros/run_single.sh#L1) | particle を `pi-` に固定して Position v4 を実行 |
| Condor submit | [`macros/submit_single.job`](macros/submit_single.job#L1) | electron wrapper を submit |
| 現行 source | [`Fun4All_singleParticle_SiliconPositionv4.C`](macros/Fun4All_singleParticle_SiliconPositionv4.C#L57) | generation から DST、analyzer、flat tree まで一括実行 |
| centroid v2 | [`Fun4All_singleParticle_Siliconv2.C`](macros/Fun4All_singleParticle_Siliconv2.C#L1) | 104 cm centroid matching、`make...v3` + feature 追加 |
| first-touch v3 | [`Fun4All_singleParticle_Siliconv3.C`](macros/Fun4All_singleParticle_Siliconv3.C#L1) | 93.5 cm first-touch matching |

`macros/run.sh` はさらに古い `Fun4All_singleParticle_Silicon.C` を呼び、現在の silicon--calorimeter matching chain とは異なるので非推奨です。

## 実行方法

### Single gun

wrapper を使う場合:

```bash
cd /sphenix/u/tomoki/work_tomoki/SiCalo/macros

# e-, process 0。NEVENTS の既定値は 10
NEVENTS=10 ./run_singlev2.sh 0

# pi-, process 0
NEVENTS=10 ./run_single.sh 0
```

particle を明示して直接実行する方が、設定が分かりやすく再利用しやすいです。

```bash
cd /sphenix/u/tomoki/work_tomoki/SiCalo/macros
root -l -b -q 'Fun4All_singleParticle_SiliconPositionv4.C(0,10,"/sphenix/u/tomoki/work_tomoki/SiCalo/output","e-")'
```

対応 particle は `e-`, `e+`, `pi-`, `pi+`, `kaon-`, `kaon+`, `proton`, `anti_proton`, `mu-`, `mu+` です（[`Positionv4.C` L74--87](macros/Fun4All_singleParticle_SiliconPositionv4.C#L74)）。

single gun では一度の実行で次の三種類を書きます。

```text
output/DST_<directory-tag>_sicalo_position_v4/
  DST_SiliconOnly_single_<particle-tag>_sicalo_position_v4_<process>.root

output/ana_<directory-tag>_sicalo_position_v4/
  topo_ana_<process>_v2.root
  SiEmcalTopoTree_<Geant4-particle-name>_<process-6桁>_v4.root
```

例えば `e-`, process 0 なら `directory-tag` と `particle-tag` は `eminus`、flat ROOT は `SiEmcalTopoTree_e-_000000_v4.root` です。`pi-` だけは過去の命名との互換性のため directory が `ana_0803piminus_sicalo_position_v4`、particle tag が `piminus` になります（[`Positionv4.C` L74--102](macros/Fun4All_singleParticle_SiliconPositionv4.C#L74)）。

Pythia と違い、single-gun Position v4 は DST output manager も登録しています（[`Positionv4.C` L261--266](macros/Fun4All_singleParticle_SiliconPositionv4.C#L261)）。

## Single-gun simulation chain

single gun は input DST を読むのではなく、`PHG4SimpleEventGenerator` から event を生成します（[`Positionv4.C` L121--131](macros/Fun4All_singleParticle_SiliconPositionv4.C#L121)）。

- 1 event に 1 particle
- vertex width 0
- `-1 < eta < 1`
- full azimuth
- `0.3 < pT < 3 GeV/c`

その後、MVTX、INTT、TPC、micromegas、CEMC、HCAL、magnet などを有効にして Geant4 を実行し、cell/tower/cluster を再構成します（[`Positionv4.C` L141--202](macros/Fun4All_singleParticle_SiliconPositionv4.C#L141)）。

`CLUSTERINFO_CEMC` は `RawClusterBuilderTemplate` で作られます。入力は `TOWERINFO_CALIB_CEMC`、tower threshold は 0.070 GeV、profile は `CEMCprof_Thresh30MeV.root`、TowerInfo と alternate-z vertex を使用します（L182--197）。その後の silicon seeding、vertex、projection、matching は Pythia と同じ構成です。

## Silicon--EMCal matching の詳細

実装本体は [`SiliconSeedAna/SiliconCaloMatching.cc`](SiliconSeedAna/SiliconCaloMatching.cc#L1) です。

### 1. 入力 node と設定

マクロが主に設定する値は次です。

| 項目 | centroid v2/Position v4/v5 | first-touch v3 |
|---|---:|---:|
| EMCal cluster node | `CLUSTERINFO_CEMC` | `CLUSTERINFO_CEMC` |
| cluster minimum energy | 0.2 GeV | 0.2 GeV |
| projection radius | 104 cm | 93.5 cm |
| inner/outer interpolation fraction | 1.0 | 1.0 |
| first-touch tower | false | true |
| `setMC(true)` | false | true |
| update track pT | true | true |

class の既定値は radius 93.5 cm、fraction 0.5、cluster threshold 0.18 GeV、track pT threshold 0.20 GeV ですが、上記は各 macro で上書きされます（[`SiliconCaloMatching.h` L80--94](SiliconSeedAna/SiliconCaloMatching.h#L80)）。

### 2. vertex と track の選択

各 event で crossing 0 の vertex を最初に採用します（[`SiliconCaloMatching.cc` L105--121](SiliconSeedAna/SiliconCaloMatching.cc#L105)）。track loop では次を行います。

- data mode、すなわち `setMC(true)` でない場合は crossing が 0 でない track を捨てる（L125--133）。
- pT が threshold 未満の track を捨てる（L136--138）。
- 指定半径と outer radius の `SvtxTrackState` を取得する（L160--180）。
- 二つの state の `phi` と `z` を fraction で補間する（L181--184）。fraction 1.0 では指定半径側の state をそのまま使う。

### 3. closest EMCal cluster

energy threshold を通る全 EMCal cluster に対し、track projection との差を

```text
distance = sqrt((delta_phi * R / 3)^2 + delta_z^2)
```

で評価し、最小の cluster を選びます（[`SiliconCaloMatching.cc` L202--230](SiliconSeedAna/SiliconCaloMatching.cc#L202)）。

ここで重要なのは、**この段階では最大 `delta_phi` や最大 `delta_z` の hard cut がない**ことです。energy cut を通る cluster があれば最も近いものを選び、距離そのものは tree に残します。

### 4. calorimeter bending から pT を再評価

silicon seed が 3 cluster 以上あることを要求し、layer 順に並べた外側 2 cluster を使って silicon 側の飛行方向を求めます（[`SiliconCaloMatching.cc` L318--359](SiliconSeedAna/SiliconCaloMatching.cc#L318)）。

centroid mode では、外側 silicon cluster から EMCal cluster centroid への方向と silicon 方向の `delta_phi` を使います（L399--407）。first-touch mode では constituent tower を調べ、charge を含む符号条件

```text
-charge * delta_phi > 0
```

を満たす tower のうち、曲がり角が最小のものを入口 tower として選びます（L366--398）。

最終的な経験式は

```text
pT = 0.21 * signed_bending_angle^(-0.986)
```

です（[`SiliconCaloMatching.cc` L409--410](SiliconSeedAna/SiliconCaloMatching.cc#L409)）。`setUpdateTrackPt(true)` の場合は、この pT と元の track phi を使って `px`, `py` を上書きします（L245--254）。

結果は `SiliconCaloTrackMap` に track ID、calo ID、再評価 pT、phi、eta、`delta_phi`、`delta_z` として保存されます（L257--268）。

## `SiliconSeedsAna` が analyzer ROOT に保存するもの

実装は [`SiliconSeedAna/SiliconSeedsAna.cc`](SiliconSeedAna/SiliconSeedsAna.cc#L1) です。event ごとに truth、vertex、EMCal cluster、track、silicon cluster、all-calorimeter topo cluster の順で処理します（[`SiliconSeedsAna.cc` L392--440](SiliconSeedAna/SiliconSeedsAna.cc#L392)）。

主な TTree は次です。

| TTree | 内容 | branch 定義 |
|---|---|---|
| `trackTree` | track state、charge、quality、truth ID、Si--Calo matching | [L171--223](SiliconSeedAna/SiliconSeedsAna.cc#L171) |
| `caloTree` | CEMC cluster と constituent tower | [L227--250](SiliconSeedAna/SiliconSeedsAna.cc#L227) |
| `SiClusTree` | silicon cluster position/layer | [L272--280](SiliconSeedAna/SiliconSeedsAna.cc#L272) |
| `evtTree` | event-level multiplicity、vertex など | [L294--317](SiliconSeedAna/SiliconSeedsAna.cc#L294) |
| `truthTree` | truth particle と CEMC entrance position | [L322--352](SiliconSeedAna/SiliconSeedsAna.cc#L322) |
| `TopoClusTree` | all-calorimeter topo cluster と tower | [L356--386](SiliconSeedAna/SiliconSeedsAna.cc#L356) |

## Truth情報を持ってくる処理の詳細

single gunでは、`PHG4SimpleEventGenerator`で作ったparticleを`G4Setup()`へ渡すため、`G4TruthInfo`と`G4HIT_CEMC`は同じjob内のGeant4 simulationから作られます（[`Positionv4.C` L121--131](macros/Fun4All_singleParticle_SiliconPositionv4.C#L121)、[L171--173](macros/Fun4All_singleParticle_SiliconPositionv4.C#L171)）。外部truth DSTは読みません。

```text
simple generator -> Geant4 -> G4TruthInfo ---------------- truth_id
                          -> tracker truth association -> track_id_truth
                          -> G4HIT_CEMC ------------------- same G4 track ID
                                                              |
make_SiEmcalTopoTree_v4.C: truth_id == track_id_truth でjoin --+
```

`build_truthreco_tables()`をanalyzerより先に登録し（[`Positionv4.C` L249](macros/Fun4All_singleParticle_SiliconPositionv4.C#L249)）、`SiliconSeedsAna::setMC(true)`でtruth treeと`SvtxEvalStack`を有効にします（L251--259）。`setMC(true)`の第2引数`doEval`は既定でtrueです（[`SiliconSeedsAna.h` L95](SiliconSeedAna/SiliconSeedsAna.h#L95)）。

### 1. `G4TruthInfo`から保存するparticle

各eventで`G4TruthInfo` nodeを取得します。nodeがなければ、そのeventでは`truthTree->Fill()`まで到達しません（[`SiliconSeedsAna.cc` L446--453](SiliconSeedAna/SiliconSeedsAna.cc#L446)）。

`GetPrimaryParticleRange()`ではなく **`GetParticleRange()`** を使うので、gunで投入した1個のprimaryだけでなく、Geant4が作ったsecondaryも対象です（[L533--538](SiliconSeedAna/SiliconSeedsAna.cc#L533)）。現在のcutはproduction vertexの`r <= 5 cm`だけで、PID、pT、eta、primary/secondaryによる選別はありません（L540--547）。

| branch | 出所・意味 |
|---|---|
| `truth_id` | Geant4 track ID。全joinのkey |
| `truth_pid` | PDG particle ID |
| `truth_px/py/pz/e`, `truth_pt/eta/phi` | particleの生成時momentum |
| `truth_x/y/z`, `truth_vtxid` | particleのproduction vertex |
| `truth_primary_id` | 元primaryのID |
| `truth_parent_id` | 直接のparent track ID |

これらのvectorはindexごとに同じtruth particleを表します（[`SiliconSeedsAna.cc` L567--601](SiliconSeedAna/SiliconSeedsAna.cc#L567)）。`truth_vtx_x/y/z`だけはparticle-alignedではなく、`GetPrimaryVtxRange()`のprimary vertex一覧です（L521--530）。

### 2. Reconstructed trackをtruth particleへ対応づける方法

`setMC(true)`により`InitRun()`で`SvtxEvalStack`を作り（[`SiliconSeedsAna.cc` L254--263](SiliconSeedAna/SiliconSeedsAna.cc#L254)）、eventごとに`SvtxTrackEval`を更新します（L647--653）。各reco trackについて、

1. `max_truth_particle_by_nclusters(track)`で寄与tracker cluster数が最大のtruth particleを選ぶ。
2. `best_track_from(truth)`で逆引きしたreco track IDが現在のtrack IDと一致することを確認する。

という二段階を通った場合だけ`track_id_truth`と`track_truth_pid`を保存します（[`SiliconSeedsAna.cc` L807--841](SiliconSeedAna/SiliconSeedsAna.cc#L807)）。position最近接ではなく、tracker cluster truth associationに基づくmatchingです。

未対応時はanalyzer ROOTで`track_id_truth = std::numeric_limits<int>::max()`、`track_truth_pid = 0`です。flat converterは`track_truth_pid`を直接使わず、`track_id_truth`から`truthTree`をjoinし直します。

### 3. Truth CEMC entrance position

`truth_cemc_*`は、reco EMCal clusterの位置分解能とCEMC到達時momentumに対するpT resolutionを測るために追加したreferenceです。

`G4HIT_CEMC`を全走査して`hit->get_trkid()`ごとのmapを作り、同じtrack IDのhit数を数えます（[`SiliconSeedsAna.cc` L468--486](SiliconSeedAna/SiliconSeedsAna.cc#L468)）。各hitのendpoint 0/1を両方調べ、有限値の中で **`r = hypot(x,y)`が最小のendpoint** をentranceとして採用します（L488--511）。

これは最小time、最大energy deposit、reco clusterのtruth contributor、またはhelix projectionではありません。選択endpointの`x/y/z/r/t`、`px/py/pz`を保存し、`phi=atan2(y,x)`、幾何学的eta=`asinh(z/r)`を計算します。

particleの`truth_id`とhitの`trkid`は完全一致でjoinします（[L585--601](SiliconSeedAna/SiliconSeedsAna.cc#L585)）。有効endpointを持つ一致hitがあれば`truth_cemc_valid=1`です。同じtrack IDのhitがなければ`valid=0`, `nhits=0`で値は`NaN`です。hitは存在しても両endpointが非有限値なら、まれに`valid=0`かつ`nhits>0`になり得ます。daughter hitをprimaryへ集約しないため、gun electronがCEMC前でsecondaryを作った場合、そのdaughter hitはprimary electronの`truth_cemc_*`には入りません。

位置分解能には`calo_phi - truth_cemc_phi`、`calo_z - truth_cemc_z`などを使えます。pT基準は二種類を区別します。

- `truth_pt`: particleの生成時pT。
- `truth_cemc_pt = hypot(truth_cemc_px, truth_cemc_py)`: upstream material中のenergy lossやscattering後、CEMCへ入った瞬間のpT。

したがってdetector入口でのpT reconstruction性能を見る場合は`truth_cemc_pt`、generator-level全体との比較には`truth_pt`を使います。

#### 既存のresolution plotでの使い方

- [`plot_EMCalPhiResolution_v4.C` L55--153](analysiscode_bymyself/plot_EMCalPhiResolution_v4.C#L55) は `emcal_truth_dphi` の分布幅をposition resolutionとして測り、pT binには生成時`truth_pt`を使います。
- [`plot_SiEmcalPtResolution_withTheory_v3.C` L189--230](analysiscode_bymyself/plot_SiEmcalPtResolution_withTheory_v3.C#L189) のpT residualは `(track_pt - truth_pt)/truth_pt` と `(calo_pt - truth_pt)/truth_pt` です。
- 同macroは`emcal_truth_dphi`のsigmaを `sigma(pT)/pT ~= 0.986 * sigma(phi) / bending_angle` に伝播し、position termによるpT-resolution予測を作ります（[L345--380](analysiscode_bymyself/plot_SiEmcalPtResolution_withTheory_v3.C#L345)）。
- `truth_cemc_pt`はCEMC入口pTを直接referenceにする別の比較に使えますが、現在の上記plotの分母は生成時`truth_pt`です。

### 4. Flat treeでの最終join

`make_SiEmcalTopoTree_v4.C`はeventごとに`truth_id -> vector index` mapを作り（[`make...v4.C` L500--503](macros/make_SiEmcalTopoTree_v4.C#L500)）、各trackの`track_id_truth`で検索します（[L626--669](macros/make_SiEmcalTopoTree_v4.C#L626)）。truth particleがproduction-vertex cutで除かれている場合も未対応になります。

未対応値は`-9999`、`truth_cemc_valid=0`、`truth_cemc_nhits=0`です。validな場合は次を追加します（[`make...v4.C` L614--678](macros/make_SiEmcalTopoTree_v4.C#L614)）。

```text
truth_cemc_pt                 = hypot(truth_cemc_px, truth_cemc_py)
emcal_truth_dphi              = wrap(calo_phi - truth_cemc_phi)
emcal_truth_ds_at_truth_r     = truth_cemc_r * emcal_truth_dphi
emcal_truth_ds_at_cluster_r   = 104 cm * emcal_truth_dphi
```

v4 converterは必要な`truth_cemc_*` branchが不足しているanalyzer ROOTを拒否します（[`make...v4.C` L319--359](macros/make_SiEmcalTopoTree_v4.C#L319)）。

### EMCal cluster

`CLUSTERINFO_CEMC` から cluster energy、position、chi2、probability を取得し、constituent tower ごとに geometry center、energy、time を保存します（[`SiliconSeedsAna.cc` L1204--1376](SiliconSeedAna/SiliconSeedsAna.cc#L1204)）。

### All-calorimeter topo cluster

`TOPOCLUSTER_ALLCALO` を読み、0.18 GeV 以上の cluster について constituent tower を CEMC、IHCAL、OHCAL に分類して energy を合計します（[`SiliconSeedsAna.cc` L1517--1728](SiliconSeedAna/SiliconSeedsAna.cc#L1517)）。tower の geometry と time も保存します。

## `make_SiEmcalTopoTree` の flat-tree 化

### `make_SiEmcalTopoTree_v3.C`

[`make_SiEmcalTopoTree_v3.C`](macros/make_SiEmcalTopoTree_v3.C#L207) は analyzer ROOT の複数 TTree を、ML で読みやすい **1 entry = 1 track--EMCal--topo match** の tree にまとめます。

処理は次の通りです。

1. `trackTree`, `caloTree`, `SiClusTree`, `TopoClusTree`, `truthTree` を開く（[L218--230](macros/make_SiEmcalTopoTree_v3.C#L218)）。
2. Topo tree に一定の event-number offset がある場合を検出して同期する（[L394--433](macros/make_SiEmcalTopoTree_v3.C#L394)）。
3. truth ID map と calo--topo map を event ごとに構築する（L444--452）。
4. track state が半径 104 cm 近傍にあること、および acceptance 内であることを確認する（[L466--490](macros/make_SiEmcalTopoTree_v3.C#L466)）。
5. 0.2 GeV 以上の CEMC cluster から、matcher と同じ距離式で closest cluster を再選択する（L493--510）。ここでも `delta_phi`, `delta_z` hard cut は意図的に適用しない（L512--513）。
6. CEMC constituent tower の共有数から topo cluster を割り当てる（下記）。
7. OHCAL tower が 5 個以上の topo cluster を除外する（L514--543）。
8. `track_id_truth` と `truth_id` を結合し、truth PID と pT を保存する（L558--571）。
9. track、truth、CEMC、Si--Calo residual、topo cluster/tower vector を一つの entry に保存する。

calo cluster と topo cluster の対応は、CEMC tower の `x/y/z/energy` が各々 `1e-3` 以内で一致する tower 数を数えます（[`make...v3.C` L97--135](macros/make_SiEmcalTopoTree_v3.C#L97)）。共有 tower 数が最大の組を順に選ぶ greedy one-to-one assignment で、最低 1 tower の共有を要求します（L137--202）。

出力 branch は event/index、track、truth、calo、matching residual、pT diagnostic、shared-tower 数、topo cluster vector、全 tower vector です（[`make...v3.C` L347--392](macros/make_SiEmcalTopoTree_v3.C#L347)）。topo/tower は選択 object だけに切らず、元の vector と `topo_index` を保存します。

### `make_SiEmcalTopoTree_firsttouch.C`

[`make_SiEmcalTopoTree_firsttouch.C`](macros/make_SiEmcalTopoTree_firsttouch.C#L1) は基本 schema は v3 と同じですが、半径 93.5 cm を使い、EMCal centroid の代わりに charge と曲がる向きが一致する最初の constituent tower を使います。対応する first-touch tower が見つからない track は除外されます。

### `make_SiEmcalTopoTree_v4.C`

[`make_SiEmcalTopoTree_v4.C`](macros/make_SiEmcalTopoTree_v4.C#L1) は v3 の 104 cm centroid chain を基礎に、`truth_cemc_x/y/z/t` を読み、次を追加します。

- truth CEMC entrance の位置と時刻
- truth CEMC entrance での pT
- reconstructed EMCal cluster と truth entrance の `delta_phi`
- truth radius および cluster radius での arc-length residual

これにより、reco cluster centroid と Geant4 truth の実際の CEMC 到達位置を直接比較できます。

## ML feature の追加

[`add_SiEmcalTopoTreeFeatures_v4.C`](macros/add_SiEmcalTopoTreeFeatures_v4.C#L1) は flat ROOT を `UPDATE` mode で開き（[L115--120](macros/add_SiEmcalTopoTreeFeatures_v4.C#L115)）、analyzer ROOT の情報から ML 用 branch を同じ tree に追加します。

追加する主な変数は以下です（[L137--168](macros/add_SiEmcalTopoTreeFeatures_v4.C#L137)）。

- `track_p`, `E/p`
- IHCAL/CEMC、OHCAL/CEMC、全 HCAL/CEMC energy ratio
- track と cluster の incidence angle
- shower の energy-weighted `sigma_eta`, `sigma_phi`
- maximum tower energy fraction
- maximum tower を中心とする 3x3 core energy fraction
- EM tower 数
- selected topo cluster の energy、ET、eta、phi、tower 数
- CEMC/IHCAL/OHCAL 別 energy と cluster probability
- track の `phi0`, `z0`

`track_p = pT cosh(eta)` と各 energy ratio は [L185--196](macros/add_SiEmcalTopoTreeFeatures_v4.C#L185)、incidence は [L255--271](macros/add_SiEmcalTopoTreeFeatures_v4.C#L255)、shower shape と 3x3 core は [L274--322](macros/add_SiEmcalTopoTreeFeatures_v4.C#L274) で計算します。

## Single-gun version の違い

| Macro | EMCal position | converter | flat output |
|---|---|---|---|
| `Siliconv2.C` | 104 cm centroid | v3 + feature v4 | `_v4.root` |
| `Siliconv3.C` | 93.5 cm first-touch tower | first-touch | `_v3.root` |
| `SiliconPositionv4.C` | 104 cm centroid + truth CEMC entrance | v4 + feature v4 | `_v4.root` |

## 「どれを使うべきか」

- 通常の electron/pion single-gun production: `Fun4All_singleParticle_SiliconPositionv4.C`
- first-touch tower 法との比較: `Fun4All_singleParticle_Siliconv3.C`
- centroid と first-touch は pT 定義が異なるため、同じ training sample には混ぜません。

## 注意点

- closest EMCal cluster 選択には residual の hard cut がありません。必要なら学習前または解析時に `sicalo_dr`, `sicalo_dphi`, `sicalo_dz` で quality cut を行います。
- first-touch と centroid では pT の定義が異なるため、同じ branch 名でも production を混ぜない方が安全です。
- flat converter は analyzer ROOT が正常に完成した後に走ります。途中で Fun4All が失敗すれば `SiEmcalTopoTree` は作られません。
