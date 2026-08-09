# VideoCombine3 仕様検討レポート

作成日: 2026-08-10
対象ブランチ: `develop_malloc`
ベース実装: `VideoCombine2` (`videohelpersuite/nodes.py:683-1120`)

## 1. 位置づけ

VideoCombine2 のコードをコピーして `class VideoCombine3` を新設する。

- ノード名: `VHS_VideoCombine3`
- 表示名: `Video Combine 3 🎥🅥🅗🅢`

VideoCombine2 は既存ワークフロー互換のためそのまま残す。出力
(`Filenames`, `image_thumbnail`, `filename`, `filebasename`) と既存ウィジェットは全て踏襲する。

## 2. 追加入力

`optional` に追加する（既存ワークフローのロードを壊さないため `required` には入れない）。

| 名前 | 型 | 既定 | 範囲 | 意味 |
|---|---|---|---|---|
| `skip_first_frames` | INT | 0 | 0–1e6, step 1 | 先頭から捨てるフレーム数 |
| `skip_last_frames` | INT | 0 | 0–1e6, step 1 | 末尾から捨てるフレーム数 |
| `audio_fade_in_seconds` | FLOAT | 0.0 | 0–1e4, step 0.01 | 出力音声の先頭フェードイン長 |
| `audio_fade_in_start_level` | FLOAT | 0.0 | 0–1.0, step 0.01 | フェードイン開始時の音量（1.0 = 原音量） |
| `audio_fade_out_seconds` | FLOAT | 0.0 | 0–1e4, step 0.01 | 出力音声の終端フェードアウト長 |
| `audio_fade_out_end_level` | FLOAT | 0.0 | 0–1.0, step 0.01 | フェードアウト終了時の音量（1.0 = 原音量） |

JS 側 `web/js/VHS.core.js` の `VHS_VideoCombine3` エントリを `VideoCombine2` と同じ内容で追加する
（ウィジェット順序リスト・`save_output→save_image` マップ・`onNodeCreated` の追加出力・プレビュー系ハンドラ）。

## 3. フレームスキップの仕様

### 3.1 適用順序

現行パイプラインは
`images → (VAE decode) → first_image取得 → pingpong → pad → loop → ffmpeg`
の順。トリムは **VAE デコード直後・`first_image` 取得の前** に挿入する。

理由:

- サムネイル（`image_thumbnail` 出力および PNG/WebP サムネイルファイル）が
  「実際に保存された動画の先頭フレーム」であるべきなので、トリム後の先頭を `first_image` にする必要がある。
- `pingpong` はトリム後の区間を折り返すのが自然（トリム前に折り返すと捨てたフレームが末尾に復活する）。
- `loop_count` の `loop=size=` に渡す `num_frames` もトリム後の値でなければループが破綻する。

### 3.2 実装方式（遅延評価を維持）

`images` はテンソルの場合もジェネレータ（VAE デコード時）の場合もあるので、両対応のトリム関数を 1 つ用意する。

- 先頭スキップ: `itertools.islice(images, skip_first, None)`
- 末尾スキップ: `collections.deque(maxlen=skip_last)` による遅延バッファ。
  `skip_last` 枚だけメモリに保持しつつ、それを超えたものから順次 yield する（全フレームをメモリに載せない）。
  `skip_last == 0` の時はバッファ処理を完全にバイパスする。

テンソル入力の場合はスライス `images[skip_first : len(images) - skip_last]` の方が高速なので、
`torch.Tensor` かつ VAE 不使用のケースは分岐してスライスで処理する。

### 3.3 フレーム数とプログレスバー

```
num_frames_out = num_frames - skip_first_frames - skip_last_frames
```

- `pbar = ProgressBar(num_frames_out)` とする（`pbar` 生成をトリム後の値算出の後ろに移動）。
- `pingpong` 時の `num_frames += num_frames - 2` 補正もトリム後の値に対して行う。

### 3.4 バリデーション / エラー処理

| 状況 | 挙動 |
|---|---|
| `skip_first + skip_last >= num_frames` | 明示的な `Exception` を送出（メッセージにフレーム数と指定値を含める）。無音で空ファイルを吐くより原因が分かりやすい |
| 負値 | ウィジェットの `min:0` で防ぐが、コード側でも `max(0, v)` でクランプ |
| `num_frames_out == 1` かつ png 系フォーマット | 既存の `%03d` 置換ロジックがそのまま動くよう `num_frames` 相当の変数を出力後フレーム数に統一 |
| `meta_batch` 併用 | バッチ分割実行だと「全体の先頭/末尾」が判定できない。`meta_batch is not None` かつスキップ指定ありなら `Exception` を送出（`pingpong` が非対応なのと同じ扱い） |
| `format` が `image/gif` `image/webp`（Pillow 経路） | トリムは VAE デコード直後に入れるので Pillow 経路にもそのまま効く。音声は元々非対応なのでフェード指定は無視（`logger.warn`） |
| `gifski_pass` 経路 | 既存実装で `audio = None` にされるため、フェード指定は無視 |

## 4. 音声の仕様

### 4.1 切り出し

`audio['waveform']` は `[batch, channels, samples]`。ffmpeg の `-ss/-t` ではなくサンプル単位のスライスで切る
（フレームとサンプルの対応を厳密にできる、かつ既存の f32le パイプ入力にそのまま乗る）。

```
sr           = audio['sample_rate']
start_sample = round(skip_first_frames / frame_rate * sr)
out_samples  = round(num_frames_out    / frame_rate * sr)
end_sample   = min(start_sample + out_samples, total_samples)
waveform_out = waveform[:, :, start_sample:end_sample]
```

- `start_sample >= total_samples`（音声が動画より短く、スキップで全部消える）→
  音声なし扱いにフォールバックし `logger.warn`。mux しない。
- 音声が短くて `end_sample` が届かない場合は既存の `apad=whole_dur=` がそのまま無音パディングするので追加処理は不要。
- 動画側は既に `total_frames_output` ベースで `min_audio_dur` を計算しているため、
  `-shortest` の挙動は現行のまま維持できる。

### 4.2 フェード

ffmpeg の `afade` フィルタを `-af` として `mux_args` に追加する。
既存の `apad` も `-af` で渡され、`merge_filter_args(mux_args, '-af')`
(`videohelpersuite/utils.py:404`) が同一 `-af` をカンマ連結してくれるので、フィルタチェーンとして共存できる。

```
audio_dur = (end_sample - start_sample) / sr   # 実際に流し込む音声長
filters = []
if audio_fade_in_seconds  > 0:
    filters.append(f"afade=t=in:st=0:d={fade_in}:silence={fade_in_level}")
if audio_fade_out_seconds > 0:
    filters.append(f"afade=t=out:st={audio_dur - fade_out}:d={fade_out}:silence={fade_out_level}")
```

設計判断:

- **フェードの到達音量**: `afade` の `silence` オプションを使う。これは「フェードの静か側のゲイン」であり、
  フェードインでは開始時、フェードアウトでは終了時の音量に相当する（既定 0 = 無音）。
  例えば `audio_fade_out_end_level=0.5` なら音量は原音の 50% まで下がって終わる。
  対になる `unity`（大きい側のゲイン、既定 1.0）は原音量固定で良いため公開しない。
  値は 0.0–1.0 にクランプする。1.0 を指定するとフェードが実質無効になる。

- **カーブ**: `curve=tri`（線形）を既定とする。`afade` の既定が `tri` なので明示指定は省略可。
  将来的にカーブ選択ウィジェットを足す余地は残す。
- **フェードアウトの起点**: `apad` によるパディングより **前** にフェードを適用する必要がある
  （パディング後だと無音部分に対してフェードがかかり、実音声の末尾が切れずに残る）。
  したがってフィルタ順を `afade(in) → afade(out) → apad` にする。
  `merge_filter_args` は出現順に連結するので、**`mux_args` 内で afade を apad より先に置く**こと。
- **長さのクランプ**: `fade_in + fade_out > audio_dur` の場合は、両者を比率配分して合計が
  `audio_dur` に収まるよう縮める（クロスして無音になるのを防ぐ）。縮めた旨を `logger.warn`。
- `audio_dur` は「スキップ適用後の実音声長」であり、動画長ではない。
  音声が動画より短い場合、フェードアウトは音声の実末尾に掛かる。
  これが望ましくない（動画末尾に合わせたい）なら `min(audio_dur, num_frames_out / frame_rate)` を使う選択もあるが、
  **実音声末尾基準**を採用する方が「音がフェードして消える」という意図に合致する。

### 4.3 中間ファイル（音声なし動画）の削除

音声を mux する際、ffmpeg は「先に生成した音声なし動画」を入力として
`-c:v copy` で音声を足した別ファイルを作る。つまり音声なし版は **mux のための中間ファイル**であり、
音声あり版が生成できた時点で不要になる。

VideoCombine3 では mux 成功後に音声なし版を削除し、`output_files`（`VHS_FILENAMES` 出力）からも取り除く。

- 削除に失敗しても（ファイルロック等）例外にはせず `logger.warn` に留め、処理は継続する。
- 音声が無い場合・Pillow 経路・`gifski_pass` 経路では mux 自体が走らないので、動画本体は当然残る。
- 既存の `VHS_KeepIntermediate` オプションによる削除処理より前段で行うため、両者は競合しない。

### 4.4 `-af` が既に指定されているフォーマットとの競合

`video_format` 側が `audio_pass` に独自の `-af` を持つケースは現状の同梱フォーマットには無いが、
`merge_filter_args` が全部連結するので破綻はしない。

## 5. ログ / 出力

- 既存の「実行コマンドをログ出力」(`logger.info(f"Executing audio mux: ...")`) はそのまま活かす。
  フィルタ連結後のコマンドが出るので、フェード指定の検証が容易。
- 追加で、トリム適用時に
  `logger.info(f"VideoCombine3: trim frames {skip_first}..{skip_last}, {num_frames} -> {num_frames_out}")`
  相当を出す。

## 6. 想定される副作用・注意点

1. **`loop_count` との組み合わせ**: ループはトリム後の区間が繰り返される。
   音声はループ分伸びないので `apad` で無音が入る（現行 VideoCombine2 と同じ挙動）。
2. **`frame_rate` が float の場合**: `floatOrInt` なので 29.97 等が入りうる。
   サンプル数計算は float 演算 → `round` で丸めるため、最大 1 サンプルの誤差に留まる。
3. **VAE 経路の末尾スキップ**: `deque` に `skip_last` 枚のデコード済みフレームを保持するため、
   `skip_last` が大きいと（例: 4K で数百フレーム）メモリを食う。
   実用上の値なら問題ないが、極端な値のときに警告を出すことは検討可。
4. **サムネイル PNG のメタデータ**: トリム後の先頭フレームに prompt/workflow メタが載る。挙動としては正しい。

## 7. 変更ファイル

| ファイル | 内容 |
|---|---|
| `videohelpersuite/nodes.py` | `VideoCombine3` クラス追加、`NODE_CLASS_MAPPINGS` / `NODE_DISPLAY_NAME_MAPPINGS` に登録 |
| `web/js/VHS.core.js` | 36 行目付近のウィジェット順序、43 行目付近の名前マップ、2065 行目の `onNodeCreated`/プレビュー登録分岐に `VHS_VideoCombine3` を追加 |

## 8. 確定した判断

当初の未決事項は以下の通り決定し、実装済み:

1. **4.2 フェードアウトの起点** — 実音声末尾基準（`apad` によるパディングより前に適用）。
2. **3.4 meta_batch 併用時** — スキップ指定があれば `Exception`（`pingpong` と同様の扱い）。

## 9. 検証結果

30 フレーム / 10fps / 3 秒（各フレームの赤成分にインデックスを埋め込み）+ 3 秒 1kHz 正弦波ステレオ音声、
`skip_first_frames=5` / `skip_last_frames=5` / フェード各 0.5 秒 / `video/h264-mp4` で実行。

| 項目 | 結果 |
|---|---|
| 出力フレーム数 | 20（期待値どおり） |
| 出力動画長 | 2.000 秒（期待値どおり） |
| 音声切り出し範囲 | サンプル `[22050, 110250)` = 0.5〜2.5 秒（映像区間と一致） |
| 出力音声長 | 1.997 秒（AAC のフレーム量子化による誤差） |
| フィルタチェーン | `afade=t=in:st=0:d=0.5, afade=t=out:st=1.5:d=0.5, apad=whole_dur=3.0`（afade が apad より前） |
| 音量 RMS | 先頭 50ms=0.029 / 中央 50ms=0.503 / 末尾 50ms=0.032 |
| 中間ファイル | 音声なし版は削除され、`output_files` にも残らない |

さらに `audio_fade_in_start_level=0.25` / `audio_fade_out_end_level=0.5` で再実行:

| 区間 | 実測 RMS | 理論値 |
|---|---|---|
| 先頭 50ms | 0.1437 | 0.503 × (0.25 + 0.75×0.05 の平均ゲイン 0.2875) ≈ 0.145 |
| 中央 50ms | 0.5032 | 原音量 |
| 末尾 50ms | 0.2643 | 0.503 × (0.55→0.50 の平均ゲイン 0.525) ≈ 0.264 |

いずれも理論値と一致し、指定音量まで下がって（上がって）フェードが止まることを確認。

補足: 埋め込みインデックスがデコード時に 1 ずれるのは h264 の RGB→YUV420→RGB 往復による量子化で、
スキップ 0 のベースライン実行でも同じずれが出るためトリム処理の問題ではない。
