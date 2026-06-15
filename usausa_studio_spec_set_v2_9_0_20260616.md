# うさうさ フィエスタ学習工房 ―― 設計仕様書 一式

- **バージョン**: v2.9.0 PRO
- **作成日**: 2026-06-16
- **作成**: うさうさ研修工房（サブ講師）
- **区分**: 社内・教育用
- **対象成果物**: 単一HTML学習ハブ（フル版 / アーティファクト版）
- **収録**: 要件定義 ／ アーキテクチャ仕様 ／ 詳細設計 ／ 実装仕様 ／ テスト仕様 ／ 付録

---

## 目次

1. 第1部 要件定義書
2. 第2部 アーキテクチャ仕様書
3. 第3部 詳細設計書
4. 第4部 実装仕様書
5. 第5部 テスト仕様書
6. 付録A 変更履歴

---

# 第1部 要件定義書

## 1.1 目的・背景

新人IT研修において、作図・資料作成・調べもの・学習の入口が分散し、受講者・講師双方の手が止まる課題があった。本成果物は、これらの機能を**単一のHTMLファイル**に集約し、つまずきを個人の努力ではなく**仕組み**で低減することを目的とする。

配布性・安定性を重視し、原則オフラインで動作する単一ファイルとして実装する。AI連携・音声読み上げのみ例外的にネットワークを使用する。

## 1.2 用語

| 用語 | 説明 |
|---|---|
| フル版 | 4ライブラリを同梱した完全オフライン版（AI/TTS除く）。 |
| アーティファクト版 | claude.ai上での描画を軽くするため、図ライブラリ(mermaid)のみCDNから読み込む軽量版。 |
| タブ | 画面上部で切り替える機能単位（15種）。 |
| ガード | 保存・出力・送信の前に機密/実名を検知し警告する仕組み。 |

## 1.3 機能要件（FR）

| ID | 機能 | 優先度 |
|---|---|---|
| FR-01 | 作図（フロー/ER/クラス/シーケンス/状態/データ設計）と即時プレビュー | 高 |
| FR-02 | Markdownメモの作成・プレビュー・一時保存 | 高 |
| FR-03 | メモからの出力（PPTX/Word/PDF/Markdown/CSV/PNG） | 高 |
| FR-04 | AI学習サポート（3口調）と音声読み上げ | 中 |
| FR-05 | 勉強会準備・MECEサポート・迷ったらマップ | 中 |
| FR-06 | DB設計支援（定義書/CREATE TABLE/ER図生成） | 中 |
| FR-07 | ツールボックス（chmod/正規表現/JSON/Base64/cron） | 中 |
| FR-08 | 設計書レビュー（セキュリティ観点の自動チェック＋AI任意） | 中 |
| FR-09 | 学習（クイズ/フラッシュ/ダッシュボード/自作登録） | 中 |
| FR-10 | 横断検索・掲示板・参考URL・困ったら・使い方 | 低 |
| FR-11 | データのバックアップ/復元（APIキーは除外） | 中 |

## 1.4 非機能要件（NFR）

| ID | 区分 | 要件 |
|---|---|---|
| NFR-01 | 可搬性 | 単一HTMLファイルで動作。ダブルクリックで起動。 |
| NFR-02 | オフライン | AI/TTS以外はネット接続なしで完結（フル版）。 |
| NFR-03 | 互換性 | ES5互換JS。オプショナルチェーン/Null合体/後読み正規表現を使用しない。 |
| NFR-04 | セキュリティ | 全innerHTML経路をエスケープ（XSS対策）。機密/実名スキャン。CSPで外部送信を制限。 |
| NFR-05 | プライバシー | APIキーは端末内(localStorage)のみ。出力ファイルに書き出さない。 |
| NFR-06 | アクセシビリティ | タブの矢印キー移動、モーダルのEsc/フォーカストラップ、文字サイズ切替。 |
| NFR-07 | サイズ | アーティファクト版は描画負荷を抑えるため軽量（実測 約879KB）。 |

## 1.5 制約条件

- フル版は外部CDN・外部フォントを使用しない（ライブラリはインライン同梱）。
- 出力ファイル名はASCII＋日付（必要に応じ時刻）。
- 成果物に実名を記載しない（役職名で代替）。
- 事実と所感を分離して記述する。捏造した統計・出典を用いない。

## 1.6 配布形態の要件

用途に応じて2系統を提供する。機能・コードは同一とし、差異は図ライブラリの供給方法とCSP・刻印に限定する。

- **フル版（同梱・完全オフライン）**: 教室配布・オフライン確実動作向け。
- **アーティファクト版（軽量）**: claude.ai上で軽快に開く用途向け。

---

# 第2部 アーキテクチャ仕様書

## 2.1 全体構成

単一HTML内に、CSS（`<style>`）・アプリJS（`<script>`）・ライブラリ（`<script>`）をインラインで内包する。外部リソースへの依存を最小化し、状態はブラウザのlocalStorageに保持する（不可時はメモリにフォールバック）。

- プレゼンテーション層: 15タブのパネル＋モーダル＋トースト＋マスコット。
- アプリ層: タブ制御、各機能モジュール、セキュリティガード、ストア抽象。
- ライブラリ層: mermaid / marked / html2canvas / pptxgenjs。
- 永続層: localStorage（キー名前空間 `usausa_*`）。

## 2.2 ライブラリ構成

| ライブラリ | 用途 | 備考 |
|---|---|---|
| mermaid 11.15.0 | 作図（テキスト→SVG） | 最大容量。アーティファクト版はCDN化。 |
| marked 12 | Markdown→HTML（プレビュー/AI整形） | 出力はsanitizeHtmlで無害化。 |
| html2canvas 1 | プレビューのPNG画像化 | ダウンロードはプレビュー方式。 |
| pptxgenjs | PowerPoint(.pptx)生成 | 表紙/目次/テーマ/図形に対応。 |

## 2.3 データ層（永続キー）

`store`オブジェクト（get/set/getJSON/setJSON）がlocalStorageを抽象化し、利用不可時はメモリ実装に切替える。主なキーを示す。

| キー | 内容 |
|---|---|
| `usausa_memo` / `usausa_board` / `usausa_refs` | メモ・掲示板・参考URL |
| `usausa_dia_codes` | 作図タイプ別の編集中コード |
| `usausa_settings` | 設定（モード/サウンド/TTS/ガード/文字サイズ） |
| `usausa_stats` | 学習統計（正答率/連続日数/学習日数） |
| `usausa_userquiz` / `usausa_userflash` | 自作クイズ/フラッシュ |
| `usausa_errlog` / `usausa_helplinks` | エラーログ/参考URL一覧 |
| `usausa_api_key` | APIキー（バックアップ・出力には含めない） |

## 2.4 セキュリティアーキテクチャ

- **出力経路の無害化**: `escapeHtml`/`e()` を全innerHTML経路に適用。AI/メモのMarkdownは `marked`→`sanitizeHtml`（script/on*/javascript:除去、リンクに rel/target 付与）。
- **機密・実名ガード**: `secretScan`（鍵/トークン/秘密鍵/パスワード/メール/IP）＋`nameScan`（敬称＋氏名、役職名は許可）を `nameGuard` に統合。保存・出力・AI送信・掲示の前に確認。
- **CSP**: `connect-src` を Anthropic API（＋アーティファクト版はcdnjs）に限定し、データの外部持ち出しを構造的に防止。`base-uri`/`form-action`/`object-src` も制限。
- **鍵の非永続化**: APIキーは `usausa_api_key` にのみ保持し、バックアップJSON・出力物に含めない。

## 2.5 配布2系統の差分（最重要）

| 観点 | フル版（同梱） | アーティファクト版（軽量） |
|---|---|---|
| mermaid供給 | インライン同梱 | cdnjs から `<script src>` で取得 |
| 他ライブラリ | インライン同梱 | インライン同梱（同じ） |
| 実測サイズ | 約4.02MB | 約879KB |
| CSP connect-src | `api.anthropic.com` のみ | `api.anthropic.com` ＋ cdnjs |
| CSP script-src | `'self' 'unsafe-inline' 'unsafe-eval' blob:` | 左記＋ `https://cdnjs.cloudflare.com` |
| オフライン作図 | 可能 | 不可（mermaid取得にネット要） |
| 刻印 | v2.9.0 PRO | v2.9.0 PRO 軽量（タイトル/バッジ/フッター） |
| mermaid未読込時 | 発生しない | ガードで作図タブにメッセージ表示（他機能は継続） |

> 機能・アプリコードは同一。差異は上表に限定される。

## 2.6 画面構成（タブ一覧）

| # | タブ | 主な機能 |
|---|---|---|
| 1 | 作図 | 6種の図を即時プレビュー |
| 2 | メモ | Markdown編集・安全チェック |
| 3 | 出力センター | PPTX/Word/PDF/MD/CSV/PNG |
| 4 | AIうさうさ | 学習サポート・読み上げ |
| 5 | 勉強会 | 準備支援 |
| 6 | MECEサポート | 目的→図＋手順 |
| 7 | DB設計 | 定義書/DDL/ER生成 |
| 8 | ツールボックス | chmod/正規表現/JSON/Base64/cron |
| 9 | 設計書レビュー | セキュリティ観点チェック |
| 10 | 掲示板 | 共有メモ |
| 11 | 参考URL | 正規ポータル/自分の記事 |
| 12 | 迷ったらマップ | 困りごと→3ステップ |
| 13 | 困ったら | エラー/Tips/セキュリティ |
| 14 | 暇つぶし学習 | クイズ/フラッシュ/ダッシュボード |
| 15 | 使い方 | ヘルプ/変更履歴 |

---

# 第3部 詳細設計書

## 3.1 タブ／パネル制御

- `MAIN_TABS` 配列からタブとパネルを生成。`switchMain(key)` で表示切替し、`aria-selected` と番号バッジを更新。
- アクセシビリティ: `maintabs` の ArrowLeft/Right/Home/End でフォーカス移動＋切替。

## 3.2 作図モジュール

- `DIAGRAMS` にタイプ別の templates / intro / cheatsheet を保持（システム構成図(subgraph)・並行(par)テンプレを含む）。
- `diaRender()`: 入力をデバウンスし `mermaid.render`→SVG挿入。構文エラーは詳細を `details` で表示。
- ガード: mermaid 未定義時は初期化・描画をスキップし、作図タブに案内を表示（アーティファクト版のCDN失敗対策）。

## 3.3 メモ／出力

- メモ: `marked` でプレビュー、スニペット挿入、手動保存、🔐安全チェック（実名＋機密）。
- 出力: `expMD/expWord/expPDF/expPNG/expCSV/expPPTX`。各関数の冒頭で `nameGuard(本文)` を実行。
- PPTX: `mdToSlides()` で見出し→スライド。テーマ(fiesta/plain/night)・表紙・目次を切替。図形(`addShape`)でアクセント。

## 3.4 AIモジュール

- `AI_MODES`(gentle/spartan/nekketsu) のシステムプロンプトで口調を切替。
- `callClaude(system,messages)`: `https://api.anthropic.com/v1/messages`、model=`claude-sonnet-4-6`。鍵あり時は `x-api-key` を付与。
- 送信前に `nameGuard`。応答は `sanitizeHtml` で描画し、TTS読み上げに対応。

## 3.5 セキュリティモジュール

- `secretScan(text)`: sk-ant/sk-/AWS(AKIA)/GitHub/Slack/JWT/Bearer/資格情報代入/メール/IP/秘密鍵 を検出し、値はマスク表示。
- `nameScan(text)`: 敬称＋氏名を検出、役職名(`ROLE_OK`)は除外、要注意の姓(`SURNAMES`)も検出。
- `nameGuard(text)`: 設定ON時、機密または実名を検出したら `confirm` で続行可否（falseで中断）。

## 3.6 ツールボックス

- chmod: r/w/x チェック→8進数・記号・コマンド（既定644）。
- 正規表現: `new RegExp` を try/catch、一致をハイライト、無限ループ防止のガード付き。
- JSON整形/検証、Base64(UTF-8往復)、cron(5フィールドの日本語簡易解説)。

## 3.7 DB設計 / MECE / マップ / 困ったら

- DB設計: 列ビルダー→定義書MD/CREATE TABLE/ER図コード生成、作図タブへ受け渡し。
- MECE: 目的→図タイプ＋3ステップ、チェックリストをメモへ。
- 迷ったらマップ: 困りごと→3ステップ。困ったら: エラー/Tips/基本/🛡️セキュリティ(OWASP Top10)/正規ポータル。

## 3.8 設計書レビュー

- **①自動チェック(送信なし)**: `secretScan`/`nameScan`＋`RV_TOPICS`(11観点)のキーワード被覆を判定し、未言及を🔴🟡🟢で提示。出力はtextContent。
- **②チェックリスト(MECE)**: 機能/セキュリティ/保守の項目→`genReviewReport`でMarkdownレポート(事実/所感分離)を生成、メモ/コピー。
- **③AIレビュー(任意/外部送信)**: 確認＋`nameGuard`後、セキュリティレビュー用システムプロンプトで`callClaude`。攻撃コードは出力しない方針。結果は`sanitizeHtml`。

## 3.9 学習機能

- `QUIZ`/`FLASH` の出題、`answerQuiz`→`recordQuiz` で統計更新。
- ダッシュボード: 正答率/連続日数/挑戦数/学習日数。`bumpStreak` で日付差から連続を算定。
- 自作: `openAddQuiz`/`openAddFlash` で登録(escape済保存)、`openManageCustom`で削除。`usausa_userquiz`/`userflash` に保持。

## 3.10 横断検索 / バックアップ / アクセシビリティ / サウンド

- 横断検索: `buildSearchIndex`(タブ/図/学習/MECE/DB/ツール) を遅延生成。Ctrl/⌘+K でモーダル、結果からタブ遷移。
- バックアップ: `exportAll`/`importAll`(`usausa_api_key`除外)。`reloadFromStore`で一部を即時反映。
- アクセシビリティ: `openModal`にフォーカストラップ/Esc/復帰、`applyFontSize`(小/標準/大)。
- サウンド: Web Audioのオフライン合成（メキシコ風BGM/効果音）。

---

# 第4部 実装仕様書

## 4.1 コーディング規約

- ES5互換。`?.`（オプショナルチェーン）・`??`（Null合体）・`(?<...)`（後読み）を使用しない。
- innerHTML へ渡す動的文字列は必ず `escapeHtml`/`e()` を経由。
- `[hidden]{display:none!important}` を全体に適用。
- 出力ファイル名はASCII＋日付（必要に応じ時刻）。

## 4.2 ビルド手順（共通）

1. テンプレートHTML内の4つのライブラリ用プレースホルダを、各ライブラリの本文で置換する。
2. 置換時、ライブラリ内の `</script>` を `<\/script>` へエスケープする。
3. 検証: プレースホルダ残存0、終了タグ `</script>` が5（4ライブラリ＋アプリ）、末尾が `</html>`、`connect-src` を含む。
4. アプリ`<script>`を抽出し `node --check` で構文確認。`?.`/`??` が0であることを確認。

```
置換 → エスケープ </script> → 検証(プレースホルダ0/終了タグ5/末尾</html>) → 構文チェック
```

## 4.3 バージョン刻印規約

現行バージョン文字列はファイル内で**5箇所一致**させる: `<title>`、Versionコメント、バッジ(`.ver`)、フッター、変更履歴(先頭)。`__ver` も同値。

## 4.4 フル版ビルド

- 4ライブラリすべてをインライン同梱。
- CSP: `connect-src` は `https://api.anthropic.com` のみ。
- 実測 約4.02MB。完全オフライン（AI/TTS除く）。

## 4.5 アーティファクト版ビルド（差分）

| 手順 | 内容 |
|---|---|
| mermaid | `<script>プレースホルダ</script>` を cdnjs の `<script src>` に置換（11.15.0固定） |
| 他3種 | インライン同梱（フル版と同じ） |
| CSP script-src | `https://cdnjs.cloudflare.com` を追加 |
| CSP connect-src | `https://cdnjs.cloudflare.com` を追加 |
| 刻印 | タイトル/バッジ/フッター/コメントに「軽量」を付与 |
| ファイル名 | `usausa_fiesta_studio_ARTIFACT_LIGHT_<日付_時刻>.html` |

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/11.15.0/mermaid.min.js"></script>
```

## 4.6 CSP仕様

| ディレクティブ | 値（フル版） | アーティファクト版での追加 |
|---|---|---|
| default-src | `'self' data: blob:` | — |
| script-src | `'self' 'unsafe-inline' 'unsafe-eval' blob:` | + cdnjs |
| style-src | `'self' 'unsafe-inline'` | — |
| img-src | `'self' data: blob:` | — |
| connect-src | `https://api.anthropic.com` | + cdnjs |
| base-uri / form-action / object-src | `'none'` | — |

> CSPの要点は `connect-src` による外部送信の制限。インライン処理と同梱ライブラリのため script/style は `'unsafe-inline'` を許可している。

## 4.7 主要関数・エンドポイント

| 関数 | 仕様 |
|---|---|
| callClaude | `POST https://api.anthropic.com/v1/messages`, model=`claude-sonnet-4-6`。鍵あり=`x-api-key`/2048トークン、鍵なし=1000トークン。 |
| downloadBlob | アンカー＋`window.open`フォールバックで保存。`stamp()`で日付時刻ファイル名。 |
| openModal | Esc/フォーカストラップ/フォーカス復帰つきのダイアログ。 |

---

# 第5部 テスト仕様書

## 5.1 方針

- jsdom 上でテンプレートを読み込み、商用品質のテストキャンペーンを実行。
- ゼロバグ（自作ロジックのFAIL=0、起動時未捕捉エラー=0）を出荷ゲートとする。
- 失敗は「製品の不具合」と「テストコード/スタブの不備」を切り分けて報告する。

## 5.2 テスト観点

- 単体: 個別関数（store/escape/secretScan 等）。
- 結合: モジュール間連携（メモ→出力、DB→作図 等）。
- 総合: 起動〜主要フロー。
- ホワイトボックス: 分岐・境界（chmod/cron/正規表現 等）。
- ブラックボックス: 入出力仕様。
- モンキー: ランダム操作後の健全性。
- バージョン別: v2.3.0〜v2.8.0 の追加機能を回帰確認。

## 5.3 件数（最終商用テスト結果）

| カテゴリ | PASS | FAIL |
|---|---|---|
| 単体 | 16 | 0 |
| 結合 | 18 | 0 |
| 総合 | 6 | 0 |
| ホワイトボックス | 13 | 0 |
| ブラックボックス | 18 | 0 |
| v2.3.0 | 20 | 0 |
| DB設計 | 13 | 0 |
| v2.5.0 | 15 | 0 |
| v2.6.0 | 20 | 0 |
| v2.7.0 | 15 | 0 |
| v2.8.0 | 10 | 0 |
| モンキー | 4 | 0 |
| **合計** | **168** | **0** |

> PPTXは実pptxgenjsで生成し、python-pptxで開いて妥当性を確認（別検証）。

## 5.4 スタブ戦略

- ライブラリ: mermaid/marked/html2canvas をスタブ、PptxGenJS は `ShapeType`/`addShape`/`write`(Blob) を備えたスタブ。
- 環境: AudioContext/speechSynthesis/clipboard/execCommand/URL.createObjectURL（生成Blob捕捉）/fetch（呼び出し捕捉）。
- JSDOM は url を指定し `runScripts:'dangerously'` でアプリを実行。

## 5.5 実行手順

```
node test_campaign.js
```

結果はカテゴリ別のPASS/FAILと、合計・起動時未捕捉エラー数を表示する。

## 5.6 合否基準・出荷判定

- 自作ロジックのFAIL=0 かつ 起動時未捕捉エラー=0 で「バージョンアップ可」。
- 判定後にバージョン刻印（5箇所一致）→ ライブラリ注入 → 構文/整合検証 → 出荷。

## 5.7 既知の限界

- CSPはjsdomで強制されないため、実機ブラウザでの描画・通信は初回に目視確認することを推奨。
- 機密・実名スキャンはキーワード/パターン判定であり、誤検知・見落としがあり得る（最終確認は人）。
- アーティファクト版の作図は cdnjs への到達性に依存する。

---

# 付録A 変更履歴（概要）

| 版 | 主な内容 |
|---|---|
| v2.3.0 | 出力センター(PPTX等)/MECE/データ設計作画/常時BGM/タブ番号 |
| v2.4.0 | DB設計タブ |
| v2.5.0 | 実名アラート(nameGuard)/バックアップ・復元 |
| v2.6.0 | ツールボックス/学習ダッシュボード/横断検索/PPT表紙・目次・テーマ/作図テンプレ追加 |
| v2.7.0 | 機密情報スキャナ/セキュリティ・チートシート/アクセシビリティ/自作クイズ・フラッシュ |
| v2.8.0 | 設計書レビュー(セキュリティ観点)/CSP(外部送信制限) |
| v2.9.0 | 最終商用テスト合格のリリース版（168 PASS/0 FAIL）。フル版＋アーティファクト(軽量)版を提供。 |

---

面白きこともなき世を面白く。（高杉晋作）
