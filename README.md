# figma-alt-suggest-lite — Figma画像のALTテキスト提案（Figma Design Agent 用軽量版）

Figmaの FRAME / SECTION / COMPONENT / COMPONENT_SET を選択して実行すると、選択内のすべての画像（`IMAGE`フィルを持つノード）の実際の見た目を確認し、具体的な日本語ALTテキストを提案する — あるいは装飾画像として説明文不要と判断する — Figma の Design Agent のカスタムスキル機能に登録して利用するスキルです。

これは有料講座「[KMRVID Figma Skills](https://kmrvid-claude-skills.gaspanik.workers.dev/figma/)」で提供されているフル版`figma-alt-suggest`（提案の確認後、Dev Modeのアノテーションとして書き込む機能つき）から、書き込み機能を取り除いたトリムダウン版です。画像の検出と提案レポートの表示までを行い、Figmaファイルへの書き込みは一切行わない「診断＆レポートのみ」の軽量版です。**画像を1枚ずつ実際に見てALT文を考えるとどんな提案が出るか、まず試してみたい方**に向いています。

> **フル版が気になる方へ**: この軽量版は画像の検出・分類・提案ロジック単体を体験いただくものです。フル版では提案の確認後、Figmaの Dev Mode → Annotations パネルに実際に書き込むところまで行えます。同じFigma Design Agent版の`figma-annotate`や[`figma-layer-rename-lite`](https://github.com/gaspanik/figma-layer-rename-lite-skill)と組み合わせて使うことも想定しています。`figma-annotate`は、フレーム内のデザインをもとに開発用のアノテーションを付与し、エンジニアとの間でのやりとり、またAIエージェントを使った実装精度の向上に寄与します。 `figma-contrast-check`（色のコントラストチェック）もFigma Skills講座内にFigma Design Agent版として収録されており、本スキルと同じ環境で組み合わせて使えます — ただし無料公開している[`figma-contrast-check-lite`](https://github.com/gaspanik/figma-contrast-check-lite-skill)はClaude Code + Figma MCP版で提供されているため、Lite版同士を試す場合は動作環境が異なる点にご注意ください（下記スコープ参照）。全スキルカタログはこちら: [KMRVID Figma Skills README](https://fragrant-edam-563.notion.site/KMRVID-Figma-Skills-README-md-3ad5dae25dd280d69c12c69b277d90b6)

> **有料講座のプレビュー**: 講座内の各種スキルを実際に動かしている様子は、こちらの[YouTube](https://www.youtube.com/@kmrvid/videos)や講座の[無料プレビュー](https://kmrvid.com/apps/ldt-course/course/6922c0ad336ff5545bbff61d/6a6b17c49dc4e0183716f605?locale=ja)（アカウント作成、ログイン不要）に掲載しております。

---

## これは何か

ALTテキストは実装時、コードを書く人が後から考えることが多く、その時にはファイル名やレイヤー名から推測するしかありません（`hero.jpg` → 「hero」というALTでは意味がありません）。このスキルはその判断をデザインレビューの段階に前倒しします — 画像を実際に見られるのは、コードになる前のFigmaファイルの中だからです。

```
キャンバス上でFRAME/SECTION/COMPONENT/COMPONENT_SETを選択
        │
        ▼
Walk tree, don't skip INSTANCE subtrees（IMAGEフィルを持つノードを収集）
        │
        ▼
同一画像（imageHash）をグルーピング — 重複は1枚だけ精査
        │
        ▼
screenshot()で実際の見た目をキャプチャし、周辺のTEXTノードも確認
        │
        ▼
装飾 / 要説明文を判定し、要説明文には具体的な日本語ALT文を提案
        │
        ▼
提案レポートを出力（書き込みはしない）
```

`INSTANCE`のサブツリーはスキップしません — カード/商品コンポーネントのインスタンス内に置かれた写真も、実際にレンダリングされて読まれる画像なので、チェック対象から外しません。

---

## スコープ

- **対象:** 選択内のすべての可視な`IMAGE`フィル（`Rectangle`や`Image 1`のような自動生成名でも対象、名前でのフィルタは行いません）。装飾（アイコン+同義ラベル、背景テクスチャ）か要説明文かを判定し、後者には具体的な内容を書いたALT文を提案します
- **非対応:** Figmaファイルへの書き込み（アノテーション追加を含む）、ベクターアイコン/イラスト（`IMAGE`フィルを持たない純粋な`VECTOR`/`BOOLEAN_OPERATION`）、ALT以外のアクセシビリティ注釈（ランドマーク・ARIAパターン）、色のコントラスト判定。これらが必要な場合はフル版`figma-alt-suggest`、あるいは`figma-annotate`・`figma-contrast-check`をご利用ください
- `figma-contrast-check`（色のコントラストチェック）と併用すると、「配色は読めるか」と「画像は説明されているか」というアクセシビリティの異なる軸を両方カバーできます。フル版はFigma Skills講座内にFigma Design Agent版として収録されており、本スキルと同じ環境（Figmaのカスタムスキル登録）で動作します。無料公開している[`figma-contrast-check-lite`](https://github.com/gaspanik/figma-contrast-check-lite-skill)はClaude Code + Figma MCPサーバー版のため、Lite版同士を併用したい場合は別途Claude Code側のセットアップが必要です

---

## 必要要件

- **Figma（Design Agent / カスタムスキル登録機能）** — このスキルはFigma内蔵のPlugin APIスクリプト実行ツール（`evaluate_script`等）のみを使用します。読み取り専用で、ファイルへの書き込みは行いません
- 外部のMCPサーバーやAPIキーは不要です。SKILL.md単体をFigma Design Agentのカスタムスキルとして登録するだけで動作します

---

## リポジトリ構成

```
skills/
  figma-alt-suggest-lite/
    SKILL.md          — Figmaのカスタムスキル登録機能に登録するスキル定義
    LICENSE
```

---

## はじめかた

**1. このリポジトリをclone、もしくはZIPダウンロードして解凍**

```bash
git clone https://github.com/gaspanik/figma-alt-suggest-lite-skill
```

**2. Figmaのカスタムスキル登録機能で `skills/figma-alt-suggest-lite/SKILL.md` を登録**

**3. キャンバス上で対象のFRAME/SECTION/COMPONENT/COMPONENT_SETを選択**

**4. チャットからスキルを実行**

画像を含むフレームを選択後、チャットで以下のように入力してスキルを実行します。
画像枚数が多い場合はその分処理に時間がかかります。セクション単位でフレームを選択して実行するのが推奨です。

```
/figma-alt-suggest-lite
```

```
このフレーム内の画像にどんなALTテキストが適切か提案して
```

```
Suggest alt text for the images in this frame
```

---

## 実行時の流れ

1. **対象の確認** — キャンバス上の選択がFRAME/SECTION/COMPONENT/COMPONENT_SETかどうかを確認
2. **画像ノードの検出** — `IMAGE`フィルを持つノードをツリー全体（インスタンス内含む）から収集し、同一画像をグルーピング
3. **キャプチャと分類** — `screenshot()`で実際の見た目を取得し、周辺のテキストとの関係から装飾/要説明文を判定、具体的なALT文を提案
4. **レポート出力** — 分類件数のサマリーと提案の一覧表（詳細は「詳細を見せて」で表示）

---

## さらに先へ

この軽量版は画像の検出・分類・提案ロジック単体を体験できるものです。提案の確認後にDev Modeのアノテーションとして実際に書き込めるフル版`figma-alt-suggest`、および他のFigma連携スキルを含む全スキルカタログは講座に含まれています: [KMRVID Figma Skills README](https://fragrant-edam-563.notion.site/KMRVID-Figma-Skills-README-md-3ad5dae25dd280d69c12c69b277d90b6)

---

Built by Masaaki Komori - [@cipher](https://x.com/cipher) · Skill for [Figma](https://www.figma.com/)
