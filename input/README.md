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

```

```
