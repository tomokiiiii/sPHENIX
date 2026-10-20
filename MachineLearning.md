# Electron separation with machine learning

このディレクトリは、sPHENIX simulation から作成した `SiEmcalTopoTree` を入力として、
track に対応する calorimeter topological cluster が electron/positron かどうかを
DeepSets model で識別するための学習・評価コードです。

現在の推奨 workflow は `ElectronClassifier` の **v4** entry point です。用途に応じて、
次の 3 種類の学習を使い分けます。

| Workflow | 学習 sample | 出力 class | Entry point |
| --- | --- | --- | --- |
| pp only | Pythia pp simulation | non-electron / electron | `train_v4_binary_pp_only.py` |
| single-gun → pp | single-particle gun で pretrain 後、Pythia pp で fine-tune | non-electron / electron | `train_v4_binary_singlegun_then_pp.py` |
| single-gun only | 選択した single-particle gun | 2–5 particle class | `train_v4_singlegun_5class_only.py` |

binary model では `abs(truth_pid) == 11` を electron、その他を background とします。
multi-class model は charge をまとめた e±、π±、K±、p/anti-p、μ± の最大 5 class を扱います。

> [!NOTE]
> この README の command は
> `/sphenix/u/tomoki/work_tomoki/ElectronSeparation_MachineLearning` を current directory
> として実行する想定です。ROOT input はリポジトリに同梱されておらず、例では
> `/sphenix/u/tomoki/work_tomoki/SiCalo/output` 以下の生成済み sample を使用します。

## 処理の概要

1. `SiCalo` で single-gun または Pythia pp simulation を処理し、
   `SiEmcalTopoTree` を持つ `*_v4.root` file を作成します。
2. `electron/datav4.py` が track–cluster matching と品質 cut を適用し、scalar feature と
   cluster 内 tower feature を読み込みます。
3. `electron/model.py` の DeepSets model が可変長の tower 集合と scalar feature を統合し、
   particle class probability を出力します。
4. 学習済み checkpoint を held-out test event に適用して CSV を作成し、ROC、score 分布、
   truth PID 別分布を plot します。
5. 必要なら `p_electron` threshold を適用し、選択 entry と score を新しい ROOT file へ保存します。

## ディレクトリ構成

```text
ElectronClassifier/
├── electron/                         dataset、model、metrics などの共通 module
├── train_v4_binary_pp_only.py        Pythia pp のみの binary 学習
├── train_v4_binary_singlegun_then_pp.py
│                                      single-gun pretraining + pp fine-tuning
├── train_v4_singlegun_5class_only.py  single-gun の 2–5 class 学習
├── evaluate_shards_vscodexv4.py       binary model の pp test split 評価
├── evaluate_shardsv4_singlegun_5class_only.py
│                                      multi-class model の single-gun test split 評価
├── filter_shards_vscodexv4.py         pp ROOT shard の score cut
├── filter_shardsv4_singlegun_5class_only.py
│                                      single-gun ROOT file の score cut
├── plot_result.py                      prediction CSV の plot
├── requirements.txt
└── output/                             checkpoint、metrics、prediction、plot
```

`train.py`、`trainv2.py`、`trainv3*.py`、`train_vscodex*.py` などは過去の開発・再現用です。
新しい v4 学習は上表の 3 entry point から開始してください。また、現状の `export_onnx.py`
と `cpp/` は pi0/direct-gamma model 用の interface が残っているため、この electron-ID v4
model の deployment 手順としては使用しません。

## Python 環境

```bash
cd /sphenix/u/tomoki/work_tomoki/ElectronSeparation_MachineLearning
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -r ElectronClassifier/requirements.txt
```

主な依存 package は PyTorch、uproot、Awkward Array、NumPy、scikit-learn、Matplotlib です。
ROOT file の filtering では PyROOT も必要なため、その場合は互換性のある sPHENIX/ROOT
環境を先に setup してください。`--device auto` は CUDA が利用可能なら GPU、それ以外は
CPU を選択します。

## v4 input と event selection

すべての推奨 workflow は ROOT tree `SiEmcalTopoTree` を読みます。truth 情報の
`truth_pid` と `truth_pt` は label・metadata としてのみ使い、model input には含めません。

読み込み時に、有限で有効な feature を持つ entry に対して次の cut を適用します。

```text
abs(match_dz)   < 10
abs(match_dphi) < 0.06
track_chi2      < 5000
-100 < calo_chi2 < 48
calo_pt > 0, track_pt > 0, track_p > 0, n_emcal_towers > 0
```

Model input は次の 2 部分です。

- global feature: cluster/track kinematics、E/p、matching residual、HCal/EMCal ratio、
  incidence、EMCal shower shape、tower 数、calorimeter energy、track parameter など 27 変数
- point feature: cluster 中の各 tower に対する Δη、Δφ、EMCal/iHCal/oHCal energy、
  log energy、cluster energy fraction の 7 変数

各 tower を共通 network で埋め込み、sum/mean/max pooling した表現と global feature を
結合して分類します。規格化係数は training split だけから計算され、checkpoint に保存されます。

## Quickstart 1: Pythia pp のみで binary e± ID を学習

Pythia pp shard だけから model を初期化して学習します。実在する file の範囲に合わせて
`--first-process` と `--last-process` を変更してください。欠損 file は既定で skip され、
`--strict-missing` を追加すると欠損時に停止します。

```bash
python3 ElectronClassifier/train_v4_binary_pp_only.py \
  --pp-pattern "/sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_pythia_sicalo_v2/SiEmcalTopoTree_0000000029-{process:06d}_v4.root" \
  --first-process 0 \
  --last-process 9999 \
  --num-of-files 50 \
  --epochs-per-file 1 \
  --batch-size 256 \
  --class-weight balanced \
  --learning-rate 1e-4 \
  --output-dir ElectronClassifier/output/pponly_v4 \
  --device auto
```

`--num-of-files 50` は全入力 file 数ではなく、50 shard を 1 個の streaming training unit
としてまとめる指定です。`--epochs-per-file` という歴史的な option 名も、実際にはこの
multi-file unit ごとの epoch 数を表します。既定の split は train/validation/test =
80%/10%/10% です。

## Quickstart 2: single-gun で pretrain して Pythia pp で fine-tune

single-gun は粒子種を制御した pretraining、Pythia pp は実際の粒子組成に近い fine-tuning
として使用します。binary label なので、下の 10 sample は e± 対 non-electron にまとめられます。

```bash
python3 ElectronClassifier/train_v4_binary_singlegun_then_pp.py \
  --eminus-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_eminus_sicalo_v2/eminus_v4.root \
  --piminus-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_0803piminus_sicalo_v2/piminus_v4.root \
  --single-gun-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_eplus_sicalo_v2/eplus_v4.root \
  --single-gun-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_piplus_sicalo_v2/piplus_v4.root \
  --single-gun-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_kminus_sicalo_v2/kminus_v4.root \
  --single-gun-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_kplus_sicalo_v2/kplus_v4.root \
  --single-gun-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_proton_sicalo_v2/proton_v4.root \
  --single-gun-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_antiproton_sicalo_v2/antiproton_v4.root \
  --single-gun-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_muminus_sicalo_v2/muminus_v4.root \
  --single-gun-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_muplus_sicalo_v2/muplus_v4.root \
  --pp-pattern "/sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_pythia_sicalo_v2/SiEmcalTopoTree_0000000029-{process:06d}_v4.root" \
  --first-process 0 \
  --last-process 9999 \
  --num-of-files 50 \
  --pretrain-epochs 10 \
  --epochs-per-file 1 \
  --batch-size 256 \
  --class-weight balanced \
  --learning-rate 1e-4 \
  --output-dir ElectronClassifier/output/binary_singlegun_then_pp_v4 \
  --device auto
```

single-gun training split から計算した規格化を pp fine-tuning にも使用します。中断後は
同じ設定に `--resume-checkpoint ElectronClassifier/output/binary_singlegun_then_pp_v4/latest_model.pt`
を追加すると、model、optimizer、および処理済み pp unit の情報を復元できます。

## Quickstart 3: single-gun のみで 2–5 class PID を学習

指定した粒子 class だけで学習します。全 10 charge-separated sample を与えると、次の
charge-inclusive 5 class になります。

| Class index | Particle | CSV probability |
| --- | --- | --- |
| 0 | e± | `P_e` |
| 1 | π± | `P_pi` |
| 2 | K± | `P_K` |
| 3 | p / anti-p | `P_p` |
| 4 | μ± | `P_mu` |

```bash
python3 ElectronClassifier/train_v4_singlegun_5class_only.py \
  --eminus-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_eminus_sicalo_v2/eminus_v4.root \
  --eplus-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_eplus_sicalo_v2/eplus_v4.root \
  --piminus-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_0803piminus_sicalo_v2/piminus_v4.root \
  --piplus-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_piplus_sicalo_v2/piplus_v4.root \
  --kminus-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_kminus_sicalo_v2/kminus_v4.root \
  --kplus-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_kplus_sicalo_v2/kplus_v4.root \
  --proton-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_proton_sicalo_v2/proton_v4.root \
  --antiproton-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_antiproton_sicalo_v2/antiproton_v4.root \
  --muminus-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_muminus_sicalo_v2/muminus_v4.root \
  --muplus-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_muplus_sicalo_v2/muplus_v4.root \
  --epochs 50 \
  --batch-size 256 \
  --class-weight balanced \
  --learning-rate 1e-4 \
  --output-dir ElectronClassifier/output/singlegun_5class_v4 \
  --device auto
```

例えば e− と π− の 2 file だけを指定すると electron/pion の 2 class model になります。
最低 2 particle class が必要で、各 train/validation/test split に全 active class が含まれない
場合は停止します。この workflow に pp sample は入りません。

## 学習結果

Binary pp workflow の output directory には次が保存されます。

| File | 内容 |
| --- | --- |
| `latest_model.pt` | model、optimizer、規格化、split、処理済み pp unit を含む checkpoint |
| `training_protocol.json` | input pattern、split、学習計画 |
| `training_history.jsonl` | epoch/unit ごとの loss、accuracy、AUC |

Single-gun multi-class workflow では `best_model.pt`、`training_history.json`、
`test_metrics.json`、`test_predictions.csv`、`dataset_summary.json` が保存されます。
`best_model.pt` は validation loss が最小の checkpoint です。

## Binary pp model の評価と plot

学習時と同じ `--num-of-files`、process range、pp pattern を使い、各 unit の held-out test
event を評価します。学習時に欠損 shard を skip した場合は評価にも `--skip-missing` を付けます。

```bash
python3 ElectronClassifier/evaluate_shards_vscodexv4.py \
  --checkpoint ElectronClassifier/output/pponly_v4/latest_model.pt \
  --pp-pattern "/sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_pythia_sicalo_v2/SiEmcalTopoTree_0000000029-{process:06d}_v4.root" \
  --first-process 0 \
  --last-process 9999 \
  --num-of-files 50 \
  --skip-missing \
  --output ElectronClassifier/output/pponly_v4/test_predictions.csv \
  --device auto

python3 ElectronClassifier/plot_result.py \
  --input-dir ElectronClassifier/output/pponly_v4 \
  --track-pt-bins 0.3,1,2,3,4,5,6 \
  --formats png,pdf \
  --dpi 200
```

既定では plot を `INPUT_DIR/plots/` 以下へ PNG と PDF で保存します。主な出力は inclusive
ROC、`p_electron` 分布、truth PID 別 background 分布、track-pT bin 別 ROC です。

## Single-gun multi-class model の再評価と plot

Checkpoint に保存された input file と event split を再利用します。既定では test entry 数を
file ごとの最小数に down-sample して sample 間を balance します。

```bash
python3 ElectronClassifier/evaluate_shardsv4_singlegun_5class_only.py \
  --checkpoint ElectronClassifier/output/singlegun_5class_v4/best_model.pt \
  --output ElectronClassifier/output/singlegun_5class_v4/test_predictions.csv \
  --test-balance per-file \
  --device auto

python3 ElectronClassifier/plot_result.py \
  --input-dir ElectronClassifier/output/singlegun_5class_v4 \
  --track-pt-bins 0.3,1,2,3,4,5,6 \
  --formats png,pdf \
  --dpi 200
```

## Electron score で ROOT shard を選別

次の例は binary checkpoint を pp shard に適用し、`p_electron >= 0.8` の entry だけを
clone します。出力 tree には元 branch に加えて `p_electron` branch が追加されます。

```bash
python3 ElectronClassifier/filter_shards_vscodexv4.py \
  --checkpoint ElectronClassifier/output/pponly_v4/latest_model.pt \
  --pp-pattern "/sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_pythia_sicalo_v2/SiEmcalTopoTree_0000000029-{process:06d}_v4.root" \
  --output-pattern "/sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_pythia_sicalo_v2/afterML_pponly/SiEmcalTopoTree_0000000029-{process:06d}_v4after.root" \
  --first-process 0 \
  --last-process 9999 \
  --score-threshold 0.6 \
  --skip-missing \
  --device auto
```

既存 output は既定で skip されます。置き換える場合だけ `--overwrite` を指定してください。
Single-gun multi-class checkpoint には
`filter_shardsv4_singlegun_5class_only.py` を使います。

## 重要な option

- `--class-weight balanced`: 現在の training split の class 数の逆数に基づく weight を使用
- `--device auto`: CUDA が利用可能なら CUDA、なければ CPU
- `--num-workers`: PyTorch DataLoader worker 数。既定は 0
- `--seed`: data split と学習の random seed。既定は 12345
- `--tree-name`: 入力 TTree 名。既定は `SiEmcalTopoTree`
- `--pooling`: tower embedding の集約方法。`sum`、`mean`、`max` から選択
- `--strict-missing`: training で要求した pp shard が欠けていたら停止
- `--skip-missing`: evaluation/filtering で欠損 pp shard を skip

全 option は各 entry point の `--help` で確認できます。

```bash
python3 ElectronClassifier/train_v4_binary_pp_only.py --help
python3 ElectronClassifier/train_v4_binary_singlegun_then_pp.py --help
python3 ElectronClassifier/train_v4_singlegun_5class_only.py --help
```

## 再現性と注意点

- train/validation/test split を再現するには、checkpoint と同じ process range、
  `--num-of-files`、file の存在状態、seed を使用してください。
- pp streaming では file の grouping が変わると unit 内 event split も変わります。
- `truth_pid` が不明な実 data へそのまま適用する reader は現在用意されていません。
  v4 dataset loader は truth branch を label/metadata として要求します。
- score cut 後の ROOT file は元 file と同じ場所へ上書きせず、別 directory に保存してください。
- `output/` 以下には大きな checkpoint、prediction、plot が生成されるため、Git への追加前に
  file size と内容を確認してください。

## 実行例: e−/π− single-gun

次は e− と π−だけで 2 class model を学習し、学習時 prediction の退避、held-out test
split の再評価、plot、`p_electron >= 0.6` の ROOT filtering まで行う一連の command です。
`evaluate_shardsv4_singlegun_5class_only.py` は既定で `test_predictions.csv` を書くため、
学習直後の CSV を比較用に残す場合は評価前にコピーします。

```bash
python3 ElectronClassifier/train_v4_singlegun_5class_only.py \
  --eminus-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_eminus_sicalo_v2/eminus_v4.root \
  --piminus-file /sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_0803piminus_sicalo_v2/piminus_v4.root \
  --epochs 50 \
  --batch-size 256 \
  --class-weight balanced \
  --learning-rate 1e-4 \
  --output-dir ElectronClassifier/output/singlegun_e_vs_pi_v4 \
  --device auto

cp \
  ElectronClassifier/output/singlegun_e_vs_pi_v4/test_predictions.csv \
  ElectronClassifier/output/singlegun_e_vs_pi_v4/test_predictions_singlegun.csv

python3 ElectronClassifier/evaluate_shardsv4_singlegun_5class_only.py \
  --checkpoint ElectronClassifier/output/singlegun_e_vs_pi_v4/best_model.pt \
  --batch-size 256 \
  --device auto

python3 ElectronClassifier/plot_result.py \
  --input-dir ElectronClassifier/output/singlegun_e_vs_pi_v4 \
  --track-pt-bins 0.2,0.5,1,1.5,2,2.5,3,4,5 \
  --formats png,pdf \
  --dpi 200

python3 ElectronClassifier/filter_shardsv4_singlegun_5class_only.py \
  --checkpoint ElectronClassifier/output/singlegun_e_vs_pi_v4/best_model.pt \
  --output-dir /sphenix/u/tomoki/work_tomoki/SiCalo/output/singlegun_e_vs_pi_afterML \
  --score-threshold 0.6 \
  --batch-size 256 \
  --device auto
```

## 実行例: Pythia pp only

`train_v4_binary_pp_only.py` は下記の `trainv4.py --pp-only` の専用 entry point です。
ここでは実際に使用した command と同じ形を示します。plot command の
`--track-pt-bins` 行末にも継続用の `\` が必要です。

```bash
python3 ElectronClassifier/trainv4.py \
  --pp-only \
  --pp-pattern "/sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_pythia_sicalo_v2/SiEmcalTopoTree_0000000029-{process:06d}_v4.root" \
  --first-process 0 \
  --last-process 9999 \
  --num-of-files 50 \
  --epochs-per-file 1 \
  --batch-size 256 \
  --class-weight balanced \
  --learning-rate 1e-4 \
  --output-dir ElectronClassifier/output/pponly_v4 \
  --device auto

python3 ElectronClassifier/evaluate_shards_vscodexv4.py \
  --checkpoint ElectronClassifier/output/pponly_v4/latest_model.pt \
  --pp-pattern "/sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_pythia_sicalo_v2/SiEmcalTopoTree_0000000029-{process:06d}_v4.root" \
  --first-process 0 \
  --last-process 9999 \
  --num-of-files 50 \
  --output ElectronClassifier/output/pponly_v4/test_predictions.csv \
  --batch-size 256 \
  --skip-missing \
  --device auto

python3 ElectronClassifier/plot_result.py \
  --input-dir ElectronClassifier/output/pponly_v4 \
  --track-pt-bins 0.3,1,2,3,4,5,6 \
  --formats png,pdf \
  --dpi 200

python3 ElectronClassifier/filter_shards_vscodexv4.py \
  --checkpoint ElectronClassifier/output/pponly_v4/latest_model.pt \
  --pp-pattern "/sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_pythia_sicalo_v2/SiEmcalTopoTree_0000000029-{process:06d}_v4.root" \
  --output-pattern "/sphenix/u/tomoki/work_tomoki/SiCalo/output/ana_pythia_sicalo_v2/afterML_pponly/SiEmcalTopoTree_0000000029-{process:06d}_v4after.root" \
  --first-process 0 \
  --last-process 9999 \
  --score-threshold 0.6 \
  --batch-size 256 \
  --skip-missing \
  --device auto
```
