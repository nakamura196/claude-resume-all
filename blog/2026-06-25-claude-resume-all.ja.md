# Claude Code の全セッションを一括で再開する

*2026-06-25*

[Claude Code](https://claude.com/claude-code) には `/resume` があり、過去のセッションに戻れます。便利なのですが——エディタが落ちて、「あれ、このプロジェクトで**7つ**もセッション走らせてた」と気づいたときに困ります。`/resume` は1つずつしか開けず、「全部 resume」する組み込み機能はありません。

そこで [`claude-resume-all`](https://github.com/nakamura196/claude-resume-all) を作りました。プロジェクトの全セッションを見つけ、それぞれ別ターミナルに展開する1コマンドです。本記事では仕組みと、途中でハマった macOS 由来の落とし穴を紹介します。

## Claude Code はセッションをどこに保存しているか

出発点は「セッションはどこにあるのか」。少し掘ると、こういう構造でした。

```
<config-dir>/projects/<エンコードされたプロジェクトパス>/<session-id>.jsonl
```

- `<config-dir>` の既定は `~/.claude`。`CLAUDE_CONFIG_DIR` を設定するとそちらを見ます(「個人」「仕事」で履歴を分けるのに便利)。
- `<エンコードされたプロジェクトパス>` は、プロジェクトの絶対パスの英数字以外をすべて `-` に置換したもの。`/Users/me/git/app` → `-Users-me-git-app`。
- 各 `*.jsonl` が1セッションで、**ファイル名がそのまま** `claude --resume <id>` に渡すセッション ID。

これがツールの全鍵です。プロジェクトのエンコード済みパス配下の `.jsonl` を列挙すれば、再開可能な全セッションが手に入ります。

```sh
ENC=$(print -r -- "$PWD" | sed 's/[^a-zA-Z0-9]/-/g')
ls -t "$HOME/.claude/projects/$ENC"/*.jsonl
```

一覧表示を見やすくするなら、`.jsonl` は JSON オブジェクトの並びなので、最初の `type:"user"` 行を取れば ID の横に出すラベルになります。

## ターミナルへ展開する

macOS で「N 個のウインドウを開いて各々でコマンドを実行」する最も確実な方法は、`osascript`(AppleScript)です。

```sh
osascript -e 'tell application "Terminal" to do script "claude --resume <id>"'
```

これをセッション ID 分ループすれば、1セッション=1ウインドウ。iTerm2 にも `create window ... write text` という同等の書き方があります。

### 落とし穴 #1: ウインドウは開いている。見えていないだけ

最初にユーザーが実行したとき、`Launched 7 session(s)` と誇らしげに出たのにウインドウが現れませんでした。開いていない?

```sh
$ osascript -e 'tell application "Terminal" to count windows'
19
```

ちゃんと開いていました。原因は、`do script` はウインドウを作るが Terminal を前面化しないこと。そしてユーザーは**全画面の VS Code(独自の macOS Space)**にいた。ウインドウは別 Space に全部あって見えなかったのです。1行で解決:

```sh
osascript -e 'tell application "Terminal" to activate'
```

### 落とし穴 #2: `command not found: claude`

最初に試した VS Code 経路(後述)で、`claude --resume …` を実行するタスクが失敗しました。

```
zsh:1: command not found: claude
```

VS Code はタスクのコマンドを**非対話のログインシェル**(`/bin/zsh -l -c '…'`)で実行します。ログインシェルは `.zprofile`/`.zlogin` は読みますが、**`.zshrc` は読みません**。多くの人が `PATH` 追加やエイリアスを書くのは `.zshrc`。だから `~/.local/bin` の `claude` も、個人の `claude-personal` エイリアスも、そのシェルには存在しませんでした。

対策は、spawn するコマンドで対話シェルの状態に頼らないこと。`command -v claude` で実体パスを解決し、config dir はエイリアスではなく環境変数で明示的に渡します。

```sh
CLAUDE_CONFIG_DIR=/Users/me/.claude-personal /Users/me/.local/bin/claude --resume <id>
```

(Terminal.app のウインドウは対話シェルなのでエイリアスも効きますが、エイリアス非依存にすればどの経路でも同じ挙動になります。)

## VS Code 問題(と、きれいな小技)

私を含め多くの人は VS Code の中で生活していて、別アプリではなく**統合**ターミナルにセッションを開きたい。ここで壁にぶつかります。

> 外部 CLI から「コマンドを実行する VS Code 統合ターミナル」を起動する公式手段は無い。VS Code が実行できるタスクは `<project>/.vscode/tasks.json` のものだけ。

つまり唯一の道は、セッションごとの専用ターミナルタスク+それらを並列実行する複合タスクを `tasks.json` に書き、VS Code 内で1回だけ実行(`タスク: タスクの実行`)すること。

ですがこのファイルはコミット対象のプロジェクト設定。生成した `tasks.json` を置きっぱなしにするとリポジトリが汚れます。そこで小さな気づき:

> タスクが統合ターミナルを起動し `claude` が走り出した後は、`tasks.json` を削除してもターミナルは動き続ける。

なのでツールはこのファイルを**一時的**に扱います。既存 `tasks.json` を退避 → 生成版を書き出し → タスク実行を待ち → **Enter 一押し**で元に戻す(自分で作った場合はファイルと `.vscode/` を削除)。ターミナルは開いたまま、リポジトリはきれいなまま。`--keep` で残すことも可能です。

生成は上書きではなく**マージ**で、既存タスクは保持し、ツールが管理するタスク(マーカー接頭辞で識別)だけを置き換えます。再実行しても冪等です。

## 個人のパスを埋め込まない

私のマシンには config dir が3つ(`~/.claude`, `~/.claude-personal`, `~/.claude-work`)あり、シェルエイリアスで切り替えています。これらを「プロファイル」としてハードコードしたくなりますが、それは*私の*環境であって、これは公開ツールです。

代わりに、実行時に `~/.claude` と `~/.claude-*` を glob して**発見**し、各ディレクトリの basename をプロファイル名にします。私のマシンでは personal/work が自動で見つかり、あなたのマシンではあなたのものが見つかる——ソースには個人のパスがゼロ。1つのプロジェクトが複数プロファイルにセッションを持つ場合は `-p` で選ばせます。任意の設定ファイルで、変な場所の config dir を登録したり、プロファイルを自分のエイリアスに対応づけたりもできます。

## おまけの落とし穴: 古典的 awk `sub()` の罠

`--help` の出力が空になりました。犯人は、先頭コメントブロックを表示するつもりのワンライナー。

```awk
NR>1 && /^#/{sub(/^# ?/,"");print}  NR>1 && !/^#/{exit}
```

awk は各行で**両方**のルールを評価します。1つ目の `sub()` が `$0` から `#` を消し、2つ目のルールが**書き換え後**の `$0` を見る——もう `#` で始まらないので、最初の行で `exit` してしまう。`next` で短絡させれば直ります。

```awk
NR==1{next}  /^#/{sub(/^# ?/,"");print;next}  {exit}
```

## まとめ

- Claude Code のセッション配置は単純でスクリプト化しやすい。ただしドキュメント化されていないディスク形式なので、変わり得る前提で扱う。
- macOS では `osascript` が Terminal.app / iTerm を操作するレバー。`activate` を忘れずに。
- spawn するコマンドは対話シェルの状態(PATH・エイリアス)に依存させない。
- VS Code の統合ターミナルは外部から操作できないが、一時的で自己後始末する `tasks.json` できれいな等価が得られる。

ツールは GitHub にあります:
[nakamura196/claude-resume-all](https://github.com/nakamura196/claude-resume-all)。
