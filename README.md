# Claude Code を活用した動画編集のサンプルリポジトリ

## 事前準備

以下の環境が用意されていることが前提です：

- Homebrew
- ffmpeg
- Node.js (v18以上)
- Python
- uv / poetry

## 環境構築

以下のプロンプトを、Claude Code で実行してください。

```markdown
以下の手順を最初から最後まで全部自動で実行してください。
途中で止まらず、全て完了してから報告してください。日本語で応答してください。

## 重要ルール

- どのステップでエラーが出ても、自分で原因を分析して修正し、そのステップをやり直してから次に進んでください。絶対に途中で止まらないでください。
- 全ステップ完了後に「環境構築が完了しました」と報告してください。途中報告は不要です。

---

## 1. Python 環境の作成とパッケージのインストール

uv / poetry を使用する。
以下のライブラリをインストール。

- openai-whisper
- torch
- torchaudio
- torchcodec（torchaudio 2.9以降で必須）
- silero-vad
- budoux

---

## 2. Whisper モデルのダウンロード

large-v3 のモデルをダウンロードします。

---

## 3. Remotion プロジェクトのセットアップ

以下を実行し、Remotion プロジェクトを作成してください。
対話形式で質問が出ます:

- プロジェクト名を聞かれたら「remotion-project」と入力
- テンプレートを聞かれたら「Blank」を選択
- パッケージマネージャーを聞かれたら「npm」を選択
- その他の質問はデフォルト（Enter）で進めてください
  \`\`\`bash
  npx create-video@latest
  \`\`\`

作成後、ライブラリをインストールします。
\`\`\`bash
cd <project_name>
npm install budoux @remotion/google-fonts
npx skills add remotion-dev/skills
cd ..
\`\`\`

作業完了後、Remotion プロジェクトをルートディレクトリに展開してください。

## 4. 最終確認

以下をすべて確認してください。

- [ ] Python 環境が存在する
- [ ] Whisper モデルがダウンロード済み
- [ ] Remotion プロジェクトが存在する

全て問題なければ「環境構築が完了しました」と報告してください。
問題がある場合は自分で修正してから報告してください。
```

## 動画生成

```markdown
以下の手順を最初から最後まで全部自動で実行してください。
途中で止まらず、全て完了してから報告してください。日本語で応答してください。

---

## 重要ルール

- どのステップでエラーが出ても、自分で原因を分析して修正し、そのステップをやり直してから次に進んでください。絶対に途中で止まらないでください。
- Python を実行するときは必ず uv / poetry を使ってください。
- 全ステップ完了後に「完了しました」と報告してください。途中報告は不要です。

入力動画: input/<movie_file>

---

## Step 1: 音声抽出

ffmpegで入力動画から音声を抽出してください。
\`\`\`bash
mkdir -p temp output
ffmpeg -y -i "input/<movie_file>" -ar 16000 -ac 1 temp/audio_16k.wav
ffmpeg -y -i "input/<movie_file>" -ar 44100 -ac 1 temp/voice_audio.wav
\`\`\`

---

## Step 2: 無音カット（ジェットカット）

Pythonスクリプトを作成・実行してください。

Silero VADで音声区間を検出:
\`\`\`python
import torch; torch.set_num_threads(1)
from silero_vad import load_silero_vad, read_audio, get_speech_timestamps
model = load_silero_vad()
wav = read_audio('temp/audio_16k.wav', sampling_rate=16000)
speech_timestamps = get_speech_timestamps(wav, model, threshold=0.5, min_silence_duration_ms=400, speech_pad_ms=150, min_speech_duration_ms=250, sampling_rate=16000)
\`\`\`

検出した音声区間をffmpegのfilter_complexで結合して、無音部分をカットした動画を作ってください。
注意: concatフィルターの入力順序は [v0][a0][v1][a1]... のようにビデオとオーディオを交互に指定すること（[v0][v1]...[a0][a1]...ではない）。

出力: temp/cut_video.mp4

セグメントが多い場合（50個以上）はチャンク方式（10個ずつ結合→最終結合）にフォールバックしてください。
カット前後の秒数とカット率をログに出力してください。

---

## Step 3: カット済み動画から音声を再抽出

\`\`\`bash
ffmpeg -y -i temp/cut_video.mp4 -ar 44100 -ac 1 temp/voice_audio.wav
\`\`\`

---

## Step 4: Whisper文字起こし

python で実行してください。
Whisper large-v3で文字起こし（メモリ不足時はmediumにフォールバック）。
word_timestamps=Trueで単語レベルのタイムスタンプを取得。

\`\`\`python
import whisper
model = whisper.load_model("large-v3")
result = model.transcribe("temp/voice_audio.wav", language="ja", word_timestamps=True)
\`\`\`

結果をtemp/whisper_result.jsonに保存。文字数と単語数をログに出力。

---

## Step 5: BudouXテロップ分割

python で実行してください。
Whisperのセグメントを、BudouXで日本語の文節境界に基づいて分割してください。
BudouXのAPI: `parser = budoux.load_default_japanese_parser()` でパーサーを取得し、`parser.parse(text)` で文節分割する。

重要ルール（厳守）:

- 1つのテロップに表示する文字数は最大30文字。30文字を超えるセグメントは、読点や文節境界で複数のテロップに分割すること
- 分割後の各テロップについて: 18文字以下は1行表示、19〜30文字はBudouXで文節分割→均等2行（最大2行厳守）
- フォントサイズは最長行の文字数で自動決定: 8文字以下→72px / 9〜12→64px / 13〜18→56px / 19〜24→48px / 25以上→42px
- 分割時、各テロップのstart/endは元セグメントの時間を文字数比で按分する

処理手順:

1. Whisperの各セグメントに対して、テキストが30文字を超えている場合はBudouXで文節分割し、30文字以下の複数テロップに分ける
2. 各テロップが19文字以上の場合はBudouXで均等2行に分割してlinesに入れる
3. 最終チェック: 全テロップのlines内の各行が30文字以下であることを確認する。超えていたら再分割する

結果をtemp/subtitles.jsonに保存。フォーマット:
[{"id":0,"start":0.5,"end":2.3,"lines":["1行目","2行目"],"fontSize":56,"color":"main"}]
テロップの件数をログに出力。

---

## Step 6: 動画情報の取得と変換

ffprobeでtemp/cut_video.mp4の解像度・コーデック・ビット深度・フレームレートを確認してください。
以下のいずれかに該当する場合は、H.264(8bit) yuv420p 30fpsに変換してtemp/cut_video.mp4を上書きしてください：

- HEVC(H.265)またはAV1コーデック
- 10bitの色深度
- 30fps以外のフレームレート（60fpsなど）
  変換コマンド: ffmpeg -y -i temp/cut_video.mp4 -c:v libx264 -pix_fmt yuv420p -r 30 -preset fast -crf 18 -c:a aac -b:a 192k temp/cut_video_converted.mp4 && /bin/cp -f temp/cut_video_converted.mp4 temp/cut_video.mp4 && rm temp/cut_video_converted.mp4
  注意: `mv` はエイリアスで対話確認が入る場合があるため `/bin/cp -f` + `rm` を使う。
  H.264で8bitで30fpsなら変換不要です。

---

## Step 7: Remotionコンポーネント自動生成

src/ に以下のファイルを自動生成してください。
コードは以下の仕様に厳密に従ってください。

### Root.tsx

\`\`\`tsx
import React from "react";
import { Composition } from "remotion";
import { MainVideo } from "./MainVideo";

export const RemotionRoot: React.FC = () => {
return (
<Composition
id="MainVideo"
component={MainVideo}
durationInFrames={/_ カット済み動画の秒数 × 30 _/}
fps={30}
width={/_ Step 6で取得したwidth _/}
height={/_ Step 6で取得したheight _/}
/>
);
};
\`\`\`

### Subtitle.tsx

propsは lines: string[] と fontSize: number。 位置は必ず bottom: 40 で画面最下部に固定。topやtranslateYは絶対に使わないこと。
\`\`\`tsx
import React from "react";
import { useCurrentFrame, interpolate } from "remotion";
import { loadFont } from "@remotion/google-fonts/NotoSansJP";

const { fontFamily } = loadFont();

interface SubtitleProps {
lines: string[];
fontSize: number;
}

export const Subtitle: React.FC<SubtitleProps> = ({ lines, fontSize }) => {
const frame = useCurrentFrame();
const opacity = interpolate(frame, [0, 3], [0, 1], { extrapolateRight: "clamp" });

return (

<div style={{
      position: "absolute",
      bottom: 40,
      left: 0,
      right: 0,
      display: "flex",
      flexDirection: "column",
      alignItems: "center",
      pointerEvents: "none",
      opacity,
    }}>
{lines.map((line, i) => (
<div key={i} style={{
          color: "#FFFFFF",
          fontSize,
          fontWeight: 900,
          fontFamily,
          textShadow: [
            "3px 0 0 #6B21A8", "-3px 0 0 #6B21A8",
            "0 3px 0 #6B21A8", "0 -3px 0 #6B21A8",
            "3px 3px 0 #6B21A8", "-3px -3px 0 #6B21A8",
            "3px -3px 0 #6B21A8", "-3px 3px 0 #6B21A8",
            "2px 2px 4px rgba(0,0,0,0.8)",
          ].join(", "),
          lineHeight: 1.4,
          whiteSpace: "nowrap",
          textAlign: "center",
        }}>
{line}
</div>
))}
</div>
);
};
\`\`\`

### MainVideo.tsx

動画はOffthreadVideoを使うこと（Videoではなく）。テロップデータはrequireで同期読み込みすること（fetchは使わないこと）。
\`\`\`tsx
import React from "react";
import { AbsoluteFill, Sequence, Audio, OffthreadVideo, staticFile } from "remotion";
import { Subtitle } from "./Subtitle";

const subtitles = require("../public/subtitles.json");

export const MainVideo: React.FC = () => {
return (
<AbsoluteFill style={{ backgroundColor: "#000" }}>
<Audio src={staticFile("voice*audio.wav")} volume={1} />
<OffthreadVideo
src={staticFile("cut_video.mp4")}
style={{ width: "100%", height: "100%", objectFit: "contain" }}
muted
/>
{subtitles.map((sub: any, idx: number) => {
const from = Math.round(sub.start * 30);
const dur = Math.max(1, Math.round((sub.end - sub.start) \_ 30));
return (
<Sequence key={idx} from={from} durationInFrames={dur}>
<Subtitle lines={sub.lines} fontSize={sub.fontSize} />
</Sequence>
);
})}
</AbsoluteFill>
);
};
\`\`\`

上記3ファイルをそのまま使ってください。durationInFrames、width、heightの数値だけ実際の値に置き換えてください。

---

## Step 8: 静的ファイルを配置

\`\`\`bash
cp temp/cut_video.mp4 public/
cp temp/voice_audio.wav public/
cp temp/subtitles.json public/
\`\`\`

---

## Step 9: 字幕テロップのブラッシュアップ

public/subtitles.json に対して、下記の3点の修正を行う:

- フォントサイズを統一（1行用、2行用どちらも同じフォントサイズに合わせる）
- 誤字の修正、専門用語を正しい表記への校正
- 字幕テロップとして、区切り位置を適切な形に変更

上記以外の変更は禁止。（内容は変えない）

---

## Step 10: Remotionレンダリング

\`\`\`bash
npx remotion render MainVideo output/final.mp4 --timeout=120000
\`\`\`

レンダリングでエラーが出た場合は、エラー内容を分析してソースコードを修正し、再度レンダリングしてください。タイムアウトの場合は npx remotion upgrade を実行してから再度レンダリングしてください。

---

## Step 11: 完成ファイルを開く

\`\`\`bash
open output/final.mp4
\`\`\`

全ステップを自動実行して、最後に以下を報告してください:
カット前後の秒数とカット率
テロップの件数
出力ファイルのパス
```
