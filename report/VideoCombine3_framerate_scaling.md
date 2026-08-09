# VideoCombine3 フレームレート変換に伴う音声時間圧縮 仕様検討レポート

作成日: 2026-08-10
対象ブランチ: `develop_malloc`
対象実装: `VideoCombine3` (`videohelpersuite/nodes.py`)
関連: [VideoCombine3_spec.md](VideoCombine3_spec.md)

## 1. 要件

入力フレーム列と入力音声が「ある想定フレームレート」で作られている前提のとき、
それより高い（あるいは低い）`frame_rate` で保存すると、映像は再生速度が変わる。
このとき音声も同じ比率で時間圧縮／伸長し、同期を保ちたい。

例: 想定 24fps の素材を `frame_rate=48` で保存 → 映像は 2 倍速 → 音声も長さを半分にする。

## 2. 追加パラメータ

| 名前 | 型 | 既定 | 範囲 | 意味 |
|---|---|---|---|---|
| `source_frame_rate` | FLOAT | 0.0 | 0–1e4, step 0.01 | 入力素材が想定しているフレームレート。0 = 無効（`frame_rate` と同じとみなし、従来動作） |

既定値に「`frame_rate` と同じ値」を書くことはできない（ウィジェット既定値は他ウィジェットを参照できない）ため、
**0 を「無効」のセンチネル**として扱う。`source_frame_rate == frame_rate` の場合も比率 1.0 となり実質無効。

速度比を次で定義する:

```
speed_ratio = frame_rate / source_frame_rate
```

`speed_ratio > 1` が早回し（音声を短く）、`< 1` がスローモーション（音声を長く）。

## 3. 映像側

**変更不要。** 現行実装は N 枚のフレームを `-r frame_rate` で raw video として流し込んでいるだけなので、
`frame_rate` を上げれば出力動画は自動的に短く（＝早回しに）なる。
`source_frame_rate` は映像のエンコードには一切関与しない。

## 4. 音声側

### 4.1 既存のトリム計算の修正（必須）

現在のコードは音声の切り出しに `frame_rate` を使っている:

```python
start_sample = min(round(skip_first_frames / frame_rate * sample_rate), total_samples)
out_samples  = round(trimmed_frame_count / frame_rate * sample_rate)
```

`skip_first_frames` / `trimmed_frame_count` は**入力フレームの枚数**であり、
入力音声は **`source_frame_rate` の時間軸**に乗っている。したがって分母は `source_frame_rate` でなければならない:

```python
start_sample = min(round(skip_first_frames    / source_fps * sample_rate), total_samples)
out_samples  = round(trimmed_frame_count / source_fps * sample_rate)
```

ここで `source_fps = source_frame_rate if source_frame_rate > 0 else frame_rate`。

これを直さないと、24fps 素材を 48fps で保存したときに音声の切り出し位置・長さが半分になり、
その後の時間圧縮でさらに半分になって二重に短くなる。**これが本対応で最も間違えやすい箇所。**

修正後の各時間軸は次の通り:

| 量 | 時間軸 | 式 |
|---|---|---|
| 切り出した音声長 | ソース時間 | `(end_sample - start_sample) / sample_rate` |
| 圧縮後の音声長 | 出力時間 | 上記 ÷ `speed_ratio` |
| 出力動画長 | 出力時間 | `total_frames_output / frame_rate` |

トリムが正しければ「切り出した音声長 ÷ speed_ratio」＝「トリム後フレーム数 ÷ frame_rate」となり、
映像と一致する。

### 4.2 時間圧縮の方式

2 案あり、聴感がまったく異なるため**ユーザーが選べるようにするのが妥当**。

#### 案A: `atempo`（ピッチ維持 / タイムストレッチ）

```
atempo=<speed_ratio>
```

位相ボコーダ的な処理で再生速度だけ変える。声や音楽が「早口になるが声の高さは変わらない」。
一般的な用途ではこちらが自然。

- 実測（1kHz 正弦波 4 秒 → `atempo=2`）: 長さ 2.002 秒、ピーク周波数 1000Hz（ピッチ維持）
- **長さがわずかに不正確**（4.000 → 2.002 秒、約 0.1%）。`-shortest` があるため出力尺には影響しないが、
  末尾がごく僅かに余る。
- **有効範囲に制限あり**。今回の検証環境（ffmpeg N-123837）では `0.5–100` だが、
  古いビルドでは `0.5–2.0` が上限。VHS は同梱 ffmpeg ではなくユーザー環境の ffmpeg / imageio-ffmpeg を使うため、
  **移植性を考えて 0.5–2.0 の範囲に収まるよう分解して連鎖させる**べき。
  - 実測: `atempo=0.25` は単体でエラー（`Value 0.250000 for parameter 'tempo' out of range`）、
    `atempo=0.5,atempo=0.5` は 4 秒 → 15.943 秒で正常動作。

分解アルゴリズム:

```python
def atempo_chain(ratio):
    parts = []
    while ratio > 2.0:
        parts.append(2.0)
        ratio /= 2.0
    while ratio < 0.5:
        parts.append(0.5)
        ratio /= 0.5
    if abs(ratio - 1.0) > 1e-6:
        parts.append(ratio)
    return [f"atempo={p:.6f}" for p in parts]
```

#### 案B: `asetrate` + `aresample`（ピッチも変わる / テープ早回し）

```
asetrate=<sample_rate * speed_ratio>,aresample=<sample_rate>
```

サンプリングレートを偽って読み替えるだけなので、**長さが数学的に正確**で、
音質劣化もリサンプリング由来のものだけ。ただしピッチが速度と同じ比率で変わる。

- 実測（同条件、`asetrate=88200,aresample=44100`）: 長さ 2.000 秒（正確）、ピーク周波数 2000Hz（1 オクターブ上）
- 効果音やアニメーション的な演出ではこちらが望ましい場合がある。
- 範囲制限なし。連鎖不要。

なお案B は「raw f32le 入力の `-ar` に `sample_rate * speed_ratio` を渡す」だけでも同等のことができるが、
その場合出力音声のサンプルレートが非標準値（例 88200）のままエンコーダに渡り、
コーデックによっては非対応となる。`aresample` で元のレートに戻す明示的な形の方が安全。

#### 案C: `rubberband`（参考）

検証環境では利用可能だったが、ffmpeg のビルドオプション依存（`--enable-librubberband`）で
存在しない環境が多い。品質は atempo より良いが、可搬性が低いため既定では採用せず、今回は見送る。

#### 推奨

`audio_speed_mode` を追加し、`atempo`（既定）/ `resample` から選べるようにする。

| 名前 | 型 | 既定 | 意味 |
|---|---|---|---|
| `audio_speed_mode` | COMBO | `"atempo"` | `"atempo"` = ピッチ維持、`"resample"` = ピッチも変化 |

### 4.3 フィルタの適用順序（重要）

現在の `-af` チェーンは `afade(in) → afade(out) → apad`。ここに速度変更を挿入する。

```
atempo/asetrate → afade(in) → afade(out) → apad
```

**速度変更を afade より前に置く**理由:

- そうしないと、指定した「フェード 0.5 秒」が圧縮後に 0.25 秒になってしまう。
  ユーザーが指定するフェード秒数は、聴く側の時間＝**出力時間軸**で解釈されるべき。
- `apad` は最後のまま。`whole_dur` は `total_frames_output / frame_rate + 1` と既に出力時間軸なので変更不要。

これに伴い、フェードアウトの開始位置に使っている `audio_dur` を**出力時間軸**に直す必要がある:

```python
audio_dur_src = (end_sample - start_sample) / sample_rate   # ソース時間
audio_dur     = audio_dur_src / speed_ratio                 # 出力時間 ← afade はこちらを使う
```

フェード長のクランプ（`fade_in + fade_out > audio_dur` のとき比率縮小）も出力時間軸で行う。

チェーンは既存の `merge_filter_args(mux_args, '-af')` が `-af` を出現順に連結するので、
`mux_args` 内で `速度変更 → afade → apad` の順に並べれば意図通りになる。

### 4.4 ログ

比率が 1.0 でないときに、判断材料が残るよう出力する:

```
VideoCombine3: audio speed x2.000 (source_frame_rate=24 -> frame_rate=48), mode=atempo,
               2.000s -> 1.000s
```

## 5. エッジケース

| 状況 | 挙動 |
|---|---|
| `source_frame_rate == 0` | 無効。`speed_ratio = 1.0` とし、フィルタを一切追加しない（従来動作を完全に維持） |
| `source_frame_rate == frame_rate` | `speed_ratio == 1.0` として同上（浮動小数の比較は `abs(r - 1.0) < 1e-6` で判定） |
| `source_frame_rate < 0` | ウィジェットの `min:0` で防ぐが、コード側でも `max(0.0, v)` でクランプ |
| 極端な比率（例 0.01 や 100） | `atempo` は分解して連鎖するため動作するが、音質は大きく劣化する。比率が 0.25 未満または 4 超のとき `logger.warn` を出す |
| 音声が動画より長い | 4.1 の修正後は「ソース時間で切り出し → 圧縮」の順になるため、切り出し時点で既に必要長になっており、圧縮後も動画長と一致する |
| 音声が動画より短い | 圧縮後さらに短くなる。従来どおり `apad` が無音で埋める |
| フェード秒数 0 | 従来どおり `afade` は付かない。速度変更フィルタのみが入る |
| Pillow 経路（gif/webp）/ `gifski_pass` | 音声自体を扱わないため無関係。`source_frame_rate` 指定時は `logger.warn` で無視を明示 |
| `loop_count` / `pingpong` | 音声長の基準は従来どおり「トリム後・展開前」のフレーム数。ループ分は無音パディングされる（既存仕様どおり） |

## 6. パラメータ全体像（追加後）

VideoCombine3 の optional 入力は次のようになる:

```
audio, meta_batch, vae, thumbnail_type, filename_counter,
skip_first_frames, skip_last_frames,
audio_fade_in_seconds,  audio_fade_in_start_level,
audio_fade_out_seconds, audio_fade_out_end_level,
source_frame_rate, audio_speed_mode          ← 今回の追加
```

## 7. 変更箇所

| ファイル | 内容 |
|---|---|
| `videohelpersuite/nodes.py` | `VideoCombine3.INPUT_TYPES` に 2 パラメータ追加／`combine_video` シグネチャ追加／音声トリムの分母を `source_fps` に修正／速度変更フィルタの生成と `mux_args` への挿入／`audio_dur` を出力時間軸へ／`atempo_chain()` ヘルパ追加 |
| `web/js/VHS.core.js` | 変更不要（ウィジェットは自動生成） |
| `report/VideoCombine3_spec.md` | 入力表と 4 章に本仕様を反映 |

## 8. 検証計画

1. **速度比 2.0 / ピッチ維持**: 24fps 想定・48fps 出力・スキップなし。
   出力動画長がフレーム数 ÷ 48 と一致し、音声も同じ長さで、ピーク周波数が入力と同じことを確認。
2. **速度比 2.0 / ピッチ変化**: `audio_speed_mode="resample"` で同条件。ピーク周波数が 2 倍になることを確認。
3. **速度比 0.5**: 48fps 想定・24fps 出力。音声が 2 倍に伸びることを確認。
4. **スキップ併用**: 24fps 想定・48fps 出力・`skip_first_frames=5` / `skip_last_frames=5`。
   切り出しサンプル範囲がソース時間で計算され（`5/24` 秒起点）、圧縮後に映像と一致することを確認。
   ここが 4.1 の修正の回帰テストになる。
5. **フェード併用**: 上記に `audio_fade_out_seconds=0.5`。
   出力音声の末尾 0.5 秒でフェードしていること（＝フェードが圧縮の影響を受けていないこと）を RMS で確認。
6. **`source_frame_rate=0`**: 既存の全テストが同じ結果になること（回帰確認）。

## 9. 実測データ（方式選定の根拠）

環境: ffmpeg N-123837-g235d5fd30a、1kHz 正弦波 4.000 秒 / 44100Hz。

| フィルタ | 出力長 | ピーク周波数 | 備考 |
|---|---|---|---|
| `atempo=2` | 2.002 秒 | 1000 Hz | ピッチ維持。長さに約 0.1% の誤差 |
| `asetrate=88200,aresample=44100` | 2.000 秒 | 2000 Hz | 長さ正確。1 オクターブ上がる |
| `atempo=0.25` | — | — | エラー（範囲 0.5–100 外） |
| `atempo=0.5,atempo=0.5` | 15.943 秒 | 1000 Hz | 連鎖で範囲外の比率にも対応可 |

## 10. 確定した判断

1. **圧縮方式** — 切り替えウィジェット `audio_speed_mode` を用意し、`atempo`（既定）/ `resample` から選択。
2. **既定値** — `atempo`（ピッチ維持）。
3. **パラメータ名** — `source_frame_rate` / `audio_speed_mode` を採用。

## 11. 検証結果（実装後）

8 章の検証計画に沿って実行。入力 48 フレーム、素材想定 24fps（＝ソース時間 2.0 秒）、
音声は 1kHz 正弦波 44100Hz。ffmpeg N-123837。

| # | 条件 | 出力動画長 | 出力音声長 | ピーク周波数 | 判定 |
|---|---|---|---|---|---|
| 1 | 24→48fps, `atempo` | 1.000s（期待 1.000s） | 0.998s | 1000 Hz（維持） | PASS |
| 2 | 24→48fps, `resample` | 1.000s（期待 1.000s） | 0.998s | 2000 Hz（2 倍） | PASS |
| 3 | 48→24fps, `atempo`（伸長） | 2.000s（期待 2.000s） | 1.997s | 1000 Hz | PASS |
| 4 | 24→48fps + skip 5/5, `atempo` | 0.792s（期待 0.792s） | 0.789s | 1000 Hz | PASS |
| 5 | #4 + `audio_fade_out_seconds=0.2` | 0.792s | 0.789s | 1000 Hz | PASS |
| 6 | `source_frame_rate=0`（無効） | 2.000s（期待 2.000s） | 1.997s | 1000 Hz | PASS |

音声長の 2〜3ms の不足は AAC のフレーム量子化と `atempo` の誤差によるもので、`-shortest` により出力尺には影響しない。

**#4 が 4.1 の修正の回帰テストになっている**。ログ上の切り出し範囲は
`trimming audio to samples [9188, 79013) of 88200` で、`9188 ≈ 5/24 × 44100`。
分母がソース時間（24fps）で計算されており、出力の 48fps ではないことを確認できる。
修正前の実装ならここが `5/48 × 44100 = 4594` となり、二重に短くなっていた。

**#5** のフェードは出力時間軸で効いていることを RMS で確認:
中央 50ms = 0.4969 に対し末尾 50ms = 0.0772（約 1/6）。
`atempo` の後に `afade` を置いているため、指定した 0.2 秒が圧縮で 0.1 秒に潰れていない。

`atempo_chain()` の単体確認（係数の積が比率と一致し、各係数が 0.5–2.0 に収まること）:

| 比率 | 生成されるフィルタ |
|---|---|
| 1.0 | （なし） |
| 2.0 | `atempo=2.000000` |
| 3.0 | `atempo=2.000000,atempo=1.500000` |
| 4.0 | `atempo=2.000000,atempo=2.000000` |
| 0.5 | `atempo=0.500000` |
| 0.25 | `atempo=0.500000,atempo=0.500000` |
| 0.125 | `atempo=0.500000,atempo=0.500000,atempo=0.500000` |

既存のテスト（トリム・フェード・中間ファイル削除）も再実行し、結果に変化がないことを確認済み。
