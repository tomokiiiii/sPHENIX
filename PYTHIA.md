# Pythia pp simulation から SiEmcalTopoTree を作る

この文書は MDC2 Pythia pp DST から `SiEmcalTopoTree` を作るまでを説明します。Machine Learning の学習・評価は [`ElectronSeparation_MachineLearning/README.ja.md`](../ElectronSeparation_MachineLearning/README.ja.md) に分離しています。

## 先に結論

現行の `dstmacro/sim/run.sh` は `Fun4All_PYTHIA_SiliconRecoPositionv5.C` を呼び、Fun4All analysis、`ana_*.root`、flat-tree conversion、ML feature 追加までを一回で実行します。ElectronClassifier の pp v4 input は別世代の `Fun4All_PYTHIA_SiliconRecov2.C` で作られています。

## ファイルの役割

### Pythia pp

| 種類 | ファイル | 役割 |
|---|---|---|
| 実行 wrapper | [`dstmacro/sim/run.sh`](dstmacro/sim/run.sh#L1) | process ID を受け取り、環境を設定して Position v5 マクロを実行 |
| Condor submit | [`dstmacro/sim/submit.job`](dstmacro/sim/submit.job#L1) | `run.sh` を 10,000 process submit |
| 現行 source | [`Fun4All_PYTHIA_SiliconRecoPositionv5.C`](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L1) | 現在 `run.sh` から呼ばれる Fun4All マクロ |
| 同等 source | [`Fun4All_PYTHIA_SiliconRecoPositionv4.C`](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv4.C#L1) | v5 と実質同じ。関数名と include guard のみ異なる |
| ML v4 production | [`Fun4All_PYTHIA_SiliconRecov2.C`](dstmacro/sim/Fun4All_PYTHIA_SiliconRecov2.C#L1) | `ana_pythia_sicalo_v2` に ElectronClassifier 用 `_v4.root` を作る |
| first-touch study | [`Fun4All_PYTHIA_SiliconRecov3.C`](dstmacro/sim/Fun4All_PYTHIA_SiliconRecov3.C#L1) | EMCal first-touch tower と 93.5 cm projection を使う旧比較版 |
| 初期版 | [`Fun4All_PYTHIA_SiliconReco.C`](dstmacro/sim/Fun4All_PYTHIA_SiliconReco.C#L1) | analyzer ROOT まで。flat tree の自動作成なし |

`run_pythia.csh`、`run_condor.job`、`run_job.csh` は古いユーザー領域を参照する legacy 経路です。新しい production には `run.sh` と `submit.job` を使用します。

## 実行方法

### Pythia: 現行 Position v5

```bash
cd /sphenix/u/tomoki/work_tomoki/SiCalo/dstmacro/sim

# process 0、10 event のテスト
NEVENTS=10 ./run.sh 0

# production submit
condor_submit submit.job
```

`run.sh` は process ID を 6 桁に変換し、既存の `_v5.root` があれば skip します（[`run.sh` L4--12](dstmacro/sim/run.sh#L4)）。実際の ROOT 呼び出しは [`run.sh` L27](dstmacro/sim/run.sh#L27) です。

現行の主出力は次です。

```text
output/ana_pythia_sicalo_position_v5/
  ana_0000000029-<process>_v5.root
  SiEmcalTopoTree_0000000029-<process>_v5.root
```

Pythia マクロ内には `DST_SiliconOnly...root` の名前も定義されていますが、output manager の登録がコメントアウトされているため、現行 Position v5 は DST を書きません（[`Positionv5.C` L337--362](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L337)）。

### Pythia: ElectronClassifier で使っている pp v4 production

`ElectronClassifier/trainv4.py` などで指定している

```text
output/ana_pythia_sicalo_v2/SiEmcalTopoTree_0000000029-<process>_v4.root
```

は `Fun4All_PYTHIA_SiliconRecov2.C` が作る系列です。1 process を直接実行する例は次です。

```bash
cd /sphenix/u/tomoki/work_tomoki/SiCalo/dstmacro/sim
root -l -b -q 'Fun4All_PYTHIA_SiliconRecov2.C(0,10)'
```

## Pythia input と Fun4All chain

Pythia マクロは Pythia event をその場で generate して full simulation するのではなく、MDC2 の `js_pp200_signal` で既に作られた run 29 の DST stream を読みます（[`Positionv5.C` L125--152](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L125)）。

読み込む主要 stream は以下です。

- G4 hits / truth
- tracker hits
- truth information
- tracker clusters
- track seeds
- calorimeter clusters

TRACKS と GLOBAL の filename も組み立てますが、その input manager はコメントアウトされています。したがって、完成 track を読むのではなく silicon seed からこの macro 内で track map を作り直します（[`Positionv5.C` L222--258](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L222)）。

主な順序は次です。

1. CDB を MDC2、run 29 に設定し `TrackingInit()` を実行（[`Positionv5.C` L198--220](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L198)）。
2. 各 DST stream を `Fun4AllDstInputManager` に登録（L222--249）。
3. `TopoClusterReco()` を実行し `TOPOCLUSTER_ALLCALO` を準備（[`Positionv5.C` L270](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L270)）。
4. `TrackSeedTrackMapConverter` で `SiliconTrackSeedContainer` から `SvtxTrackMap` を作る（[`Positionv5.C` L291--297](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L291)）。
5. `PHSimpleVertexFinder` で vertex を再構成（L301--312）。主な cut は DCA 0.075 cm、track pT 0.1 GeV、beam-line cut 1 cm、quality 300、MVTX cluster 数 3。
6. charge を再計算し、global vertex node を作る（L314--322）。
7. `PHActsTrackProjection` で track state を EMCal 半径 104 cm に外挿（L325--328）。
8. truth/reco association table を作る（[`Positionv5.C` L333](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L333)）。
9. `SiliconCaloMatching` を登録（L371--379）。
10. `SiliconSeedsAna` を登録して analyzer ROOT を作る（L382--392）。
11. `se->run()`、`End()` 後に flat-tree converter と feature adder を順に実行（[`Positionv5.C` L397--435](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L397)）。

calorimeter waveform node は現在の DST schema と合わず、この解析でも不要なため無効化しています（[`Positionv5.C` L242--249](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L242)）。

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

truthは一度に一か所から取得しているのではなく、次の複数の情報源を最後にtrack IDでjoinしています。

```text
G4TruthInfo -> 全truth particleの生成時情報 --------- truth_id
                                                       |
tracker hit/cluster truth association -> reco track -> track_id_truth
                                                       |
G4HIT_CEMC -> 同じG4 track IDの最内周endpoint -------+
                                                       |
make_SiEmcalTopoTree_v4.C: truth_id == track_id_truth でjoin
```

### 1. Pythia DSTのどのstreamから読むか

Pythia版ではtruth関係だけでも二種類のinputを使います。

- `g4hits/.../G4Hits_pythia8_Detroit-*.root`: `G4TruthInfo`、`G4HIT_CEMC`などのGeant4 particle/hit nodeを供給。
- `nopileup/trkrhit/.../DST_TRUTH_pythia8_Detroit-*.root`とtracker hit/cluster stream: reco clusterとGeant4 hitのassociationを供給。

filenameの組み立ては [`Positionv5.C` L125--152](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L125)、input managerへの登録は [L222--249](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L222) です。`build_truthreco_tables()` をanalyzerより先に登録し、truth/reco evaluatorが必要とする対応表を準備します（[L331--334](dstmacro/sim/Fun4All_PYTHIA_SiliconRecoPositionv5.C#L331)）。

`SiliconSeedsAna::setMC(true)` は既定で `doEval=true` も設定します（[`SiliconSeedsAna.h` L95](SiliconSeedAna/SiliconSeedsAna.h#L95)）。したがって `InitRun()` で `SvtxEvalStack` が作られ、`truthTree`とtrack truth matchingの両方が有効になります（[`SiliconSeedsAna.cc` L254--263](SiliconSeedAna/SiliconSeedsAna.cc#L254)）。

### 2. `G4TruthInfo`から保存するparticle

各eventの最初に `findNode::getClass<PHG4TruthInfoContainer>(topNode, "G4TruthInfo")` でcontainerを取得します（[`SiliconSeedsAna.cc` L446--453](SiliconSeedAna/SiliconSeedsAna.cc#L446)）。nodeがなければ、そのeventでは`truthTree->Fill()`まで到達せずtruth entryを作りません。

particle loopは`GetPrimaryParticleRange()`ではなく **`GetParticleRange()`** を使うため、primaryだけでなくsecondaryも対象です（[L533--538](SiliconSeedAna/SiliconSeedsAna.cc#L533)）。現在の選別はproduction vertexの横方向半径

```text
vertex_r = sqrt(vertex_x^2 + vertex_y^2) <= 5 cm
```

だけです（L540--547）。PID、pT、eta、primary/secondaryによるcutはありません。保存するvectorは次の通りで、同じindexが同じtruth particleを表します（[L567--601](SiliconSeedAna/SiliconSeedsAna.cc#L567)）。

| branch | 出所・意味 |
|---|---|
| `truth_id` | `PHG4Particle::get_track_id()`。以降のjoin key |
| `truth_pid` | PDG particle ID |
| `truth_px/py/pz/e` | particleに保存された生成時4-momentum |
| `truth_pt/eta/phi` | 上の4-momentumから計算 |
| `truth_x/y/z`, `truth_vtxid` | `get_vtx_id()`から`GetVtx()`で取得した、このparticleのproduction vertex |
| `truth_primary_id` | Geant4 recordのprimary ID |
| `truth_parent_id` | Geant4 recordのparent track ID |

`truth_vtx_x/y/z`は上記particle-aligned vectorとは別です。`GetPrimaryVtxRange()`を走査してprimary vertexだけを順に保存しますが、対応するvertex ID branchはありません（[L521--530](SiliconSeedAna/SiliconSeedsAna.cc#L521)）。particleごとのvertexには`truth_x/y/z`と`truth_vtxid`を使います。

### 3. Reconstructed trackをtruth particleへ対応づける方法

`processTrackMap()`でeventごとに`SvtxEvalStack::next_event()`を呼び、`SvtxTrackEval`を取得します（[`SiliconSeedsAna.cc` L647--653](SiliconSeedAna/SiliconSeedsAna.cc#L647)）。各reco trackに対して次の二段階を要求します。

1. `max_truth_particle_by_nclusters(track)`で、そのreco trackに最も多くtracker clusterを寄与したtruth particleを選ぶ。
2. `best_track_from(g4particle_match)`でtruth側から最良reco trackを逆引きし、そのtrack IDが現在処理中のreco track IDと一致することを確認する。

両方を通った場合だけ、`get_track_id()`を`track_id_truth`、`get_pid()`を`track_truth_pid`に保存します（[`SiliconSeedsAna.cc` L807--841](SiliconSeedAna/SiliconSeedsAna.cc#L807)）。つまり単なる最近接position matchingではなく、tracker cluster truth associationに基づく相互確認付きmatchingです。

対応が見つからない場合、analyzer ROOTでは`track_id_truth = std::numeric_limits<int>::max()`、`track_truth_pid = 0`です。flat converterは`track_truth_pid`をそのまま採用せず、`track_id_truth`を使って`truthTree`を引き直します。

### 4. Truth CEMC entrance position

`truth_cemc_*`は、reco EMCal clusterの位置分解能とCEMC到達時momentumに対するpT resolutionを測るために追加したreferenceです。

`G4HIT_CEMC`を全走査し、`PHG4Hit::get_trkid()`をkeyとするmapを先に作ります（[`SiliconSeedsAna.cc` L468--485](SiliconSeedAna/SiliconSeedsAna.cc#L468)）。同じG4 track IDに属するhit数を`truth_cemc_nhits`として数えます。

各hitについてendpoint 0と1の両方を調べ、有限値の候補のうち **円筒半径 `r = hypot(x,y)` が最小のendpoint** をentranceとして採用します（L488--511）。したがって、これは次のいずれでもありません。

- 最小timeのhit
- 最大energy depositのhit
- reco EMCal clusterのtruth contributor
- helixをEMCal面へ外挿したposition

選ばれたendpointから`x/y/z/r/t`と`px/py/pz`を保存し、`phi = atan2(y,x)`、position etaを`asinh(z/r)`で計算します。この`truth_cemc_eta`はmomentum etaではなく、原点からendpointを見た幾何学的etaです。位置分解能には`calo_phi - truth_cemc_phi`、`calo_z - truth_cemc_z`などを使えます。`truth_pt`が生成時pTなのに対し、`truth_cemc_pt = hypot(truth_cemc_px, truth_cemc_py)`はmaterial中のenergy lossやscattering後、CEMCへ入った瞬間のpTです。CEMC入口基準のpT resolutionには後者を使います。

#### 既存のresolution plotでの使い方

- [`plot_EMCalPhiResolution_v4.C` L55--153](analysiscode_bymyself/plot_EMCalPhiResolution_v4.C#L55) は `truth_cemc_valid` を要求し、`emcal_truth_dphi = calo_phi - truth_cemc_phi` の分布幅をposition resolutionとして測ります。pT binは現在、生成時`truth_pt`で切っています。
- [`plot_SiEmcalPtResolution_withTheory_v3.C` L189--230](analysiscode_bymyself/plot_SiEmcalPtResolution_withTheory_v3.C#L189) の直接的なpT residualは `(track_pt - truth_pt)/truth_pt` と `(calo_pt - truth_pt)/truth_pt` です。
- 同macroは別途`emcal_truth_dphi`のsigmaを測り、経験式のbending angleを使って `sigma(pT)/pT ~= 0.986 * sigma(phi) / bending_angle` に伝播します（[L345--380](analysiscode_bymyself/plot_SiEmcalPtResolution_withTheory_v3.C#L345)）。つまり現在のplotでは`truth_cemc`位置を、position termがpT resolutionへ与える寄与の予測に使っています。
- `truth_cemc_pt` branchはCEMC入口pTを直接referenceにする比較にも使えますが、上記の既存pT plotの分母は`truth_cemc_pt`ではなく`truth_pt`です。

その後、`G4TruthInfo`で保持した各particleの`truth_id`とCEMC hitの`trkid`を **完全一致** でjoinします（[L585--601](SiliconSeedAna/SiliconSeedsAna.cc#L585)）。有効endpointを持つ一致hitがあれば`truth_cemc_valid=1`です。同じtrack IDのhit自体がなければ`valid=0`, `nhits=0`でposition/momentum/timeは`NaN`です。hitは存在しても両endpointが非有限値なら、まれに`valid=0`かつ`nhits>0`になり得ます。parentや`primary_id`への繰り上げはしないため、primary electronがCEMC到達前にsecondaryを作り、そのdaughterだけがCEMCへ入った場合、そのhitはprimary electronには加算されません。

### 5. Flat treeでの最終joinと欠損値

`make_SiEmcalTopoTree_v4.C`はeventごとに`truth_id -> truthTree vector index`のmapを作り（[`make...v4.C` L500--503](macros/make_SiEmcalTopoTree_v4.C#L500)）、各reco trackの`track_id_truth`をkeyに検索します（[L626--669](macros/make_SiEmcalTopoTree_v4.C#L626)）。このため、track evaluatorでmatchしていても、そのtruth particleが`vertex_r > 5 cm`で`truthTree`から除かれていればflat treeではtruth未対応になります。

未対応時のflat-tree値は`truth_pid/truth_pt/truth_cemc_* = -9999`、`truth_cemc_valid=0`、`truth_cemc_nhits=0`です（[`make...v4.C` L15](macros/make_SiEmcalTopoTree_v4.C#L15)、[L614--625](macros/make_SiEmcalTopoTree_v4.C#L614)）。`truth_cemc_valid=1`の場合だけ、次を計算します（[L670--678](macros/make_SiEmcalTopoTree_v4.C#L670)）。

```text
truth_cemc_pt                 = hypot(truth_cemc_px, truth_cemc_py)
emcal_truth_dphi              = wrap(calo_phi - truth_cemc_phi)
emcal_truth_ds_at_truth_r     = truth_cemc_r * emcal_truth_dphi
emcal_truth_ds_at_cluster_r   = 104 cm * emcal_truth_dphi
```

v4 converterは`truth_cemc_*` branchが一つでも不足すると変換を中止し、更新済み`SiliconSeedsAna`でanalyzer ROOTを再生成するようerrorを出します（[`make...v4.C` L319--359](macros/make_SiEmcalTopoTree_v4.C#L319)）。

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

## 各 Pythia version の違い

| Macro | EMCal position/matching | Truth CEMC | flat converter | 出力 |
|---|---|---|---|---|
| `SiliconReco.C` | projection の明示半径なし、初期 centroid | truth table 無効 | 自動実行なし | `ana_pythia3/ana_*.root` |
| `SiliconRecov2.C` | 104 cm、cluster centroid | truth table あり、flat schema は v3 相当 | `make...v3` + feature v4 | `ana_pythia_sicalo_v2/*_v4.root` |
| `SiliconRecov3.C` | 93.5 cm、first-touch tower | truth table 無効 | `make...firsttouch` | `ana_pythia_sicalo_v3/*_v3.root` |
| `Positionv4.C` | 104 cm、cluster centroid | CEMC entrance position あり | `make...v4` + feature v4 | position-v5 dir の `*_v5.root` |
| `Positionv5.C` | 104 cm、cluster centroid | CEMC entrance position あり | `make...v4` + feature v4 | position-v5 dir の `*_v5.root` |

`Positionv4.C` と `Positionv5.C` の処理内容は現状同じで、差は include guard と関数名だけです。さらに、どちらも output directory と filename は `position_v5` / `_v5.root` です。このため、**マクロ名の v4/v5、converter の v4、出力 tree の v5 は同じ意味の version 番号ではありません**。

## 「どれを使うべきか」

- 既存の ElectronClassifier pp v4 学習を再現する: `Fun4All_PYTHIA_SiliconRecov2.C`
- truth CEMC entrance position を含む最新比較を作る: `run.sh` → `Fun4All_PYTHIA_SiliconRecoPositionv5.C`
- first-touch tower 法との比較を再現する: `Fun4All_PYTHIA_SiliconRecov3.C`

production の再現性を保つには、ROOT filename の `_v4` / `_v5` だけでなく、使用した Fun4All macro、converter、projection radius、centroid/first-touch の別を一緒に記録してください。

## 注意点

- closest EMCal cluster 選択には residual の hard cut がありません。必要なら学習前または解析時に `sicalo_dr`, `sicalo_dphi`, `sicalo_dz` で quality cut を行います。
- first-touch と centroid では pT の定義が異なるため、同じ branch 名でも production を混ぜない方が安全です。
- Pythia Position v5 の `DST_SiliconOnly...root` は名前だけ定義され、現在は出力されません。
- flat converter は analyzer ROOT が正常に完成した後に走ります。途中で Fun4All が失敗すれば `SiEmcalTopoTree` は作られません。
- `run.sh` は最終 flat ROOT の存在で process 全体を skip するため、analyzer だけを作り直したい場合や schema を変更した場合は output 管理に注意が必要です。
- `macros/run_ai.sh` は既存 Pythia analyzer ROOT に `make...v3` と feature adder を後から適用する backfill 用で、simulation を走らせる script ではありません。
