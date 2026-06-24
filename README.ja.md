# claude-resume-all

[Claude Code](https://claude.com/claude-code) の**あるプロジェクトの全セッション**を、`/resume` で1つずつ選ぶ代わりに、**一括で**(それぞれ別ターミナルに)再開するツールです。

> English: [README.md](README.md)

## なぜ必要か

Claude Code の `/resume`(および `claude --resume`)は、セッションを**1つずつ**しか復帰できません。エディタやターミナルが落ちたとき、そのプロジェクトで走らせていた複数の Claude スレッドを1つずつ開き直すのは面倒です。

`claude-resume-all` は、現在のプロジェクトに保存された**全セッション**を見つけ、1コマンドで別々のターミナルに展開します。

```console
$ claude-resume-all
Project : /Users/me/git/app
Profile : personal  (/Users/me/.claude-personal)
Launcher: claude
Sessions: 7
  ed861b4f  地名に紐づける写真アップロード機能の追加
  cb5cafea  ファクトチェック完了後に publish:true にする
  ...
Open 7 terminal window(s)? [y/N] y
Launched 7 session(s) via claude.
```

## 仕組み

Claude Code は各セッションを `*.jsonl`(ファイル名=セッションID)として、次の場所に保存します。

```
<config-dir>/projects/<エンコードされたプロジェクトパス>/<session-id>.jsonl
```

`<config-dir>` の既定は `~/.claude`。`<エンコードされたプロジェクトパス>` は、プロジェクトの絶対パスの英数字以外をすべて `-` に置換したもの(例: `/Users/me/git/app` → `-Users-me-git-app`)。

`claude-resume-all` はこのディレクトリを発見し、プロジェクトのセッションを一覧し、各 ID に対して新しいターミナルで `claude --resume <id>` を起動します。

## インストール

**zsh**(macOS 既定)が必要です。`python3` は任意ですが推奨(セッションの要約ラベル表示と `vscode` バックエンドで使用)。

```sh
git clone https://github.com/nakamura196/claude-resume-all.git
ln -s "$PWD/claude-resume-all/bin/claude-resume-all" ~/.local/bin/claude-resume-all
# ~/.local/bin が PATH に入っていることを確認
```

(または `bin/claude-resume-all` を PATH 上の好きな場所にコピー)

## 使い方

```sh
claude-resume-all                 # $PWD の全セッション。プロファイルは自動検出
claude-resume-all /path/to/proj   # 別のプロジェクトディレクトリ
claude-resume-all -p work         # プロファイルを明示指定
claude-resume-all --config-dir ~/.claude-x   # config dir を直接指定
claude-resume-all --recover       # クラッシュで閉じたセッションだけ(後述)
claude-resume-all --recent 3h     # 直近3時間にアクティブだったものだけ
claude-resume-all --since 14:30   # 本日その時刻以降にアクティブだったものだけ
claude-resume-all --order old     # 古い順に開く(最後に触ったものが最前面に)
claude-resume-all --recursive     # サブディレクトリから起動したセッションも拾う
claude-resume-all --list          # 一覧のみ(開かない)
claude-resume-all --app iterm     # terminal | iterm | vscode | print
claude-resume-all -y              # 確認プロンプトをスキップ
claude-resume-all --help
```

### クラッシュ復元 — 強制終了したものだけ再開(`--recover`)

強制終了の後に欲しいのは、たいてい**落ちた瞬間に開いていた**セッションです。これまでに作った全部でもなければ、自分でクリーン終了したものでもありません。

セッション実行中、Claude Code は `<config-dir>/sessions/<pid>.json`(`sessionId` と `cwd` を保持)を書きます。クリーン終了時はこのファイルを削除し、クラッシュ/強制終了では残します。つまり **「ファイルが残っているのに PID が死んでいる」= 落ちた瞬間に開いていたセッション**。`--recover` はまさにそれだけを再開します(クリーン終了済みも起動中も除外)。記録された `cwd` を使うので、サブディレクトリ起動セッションを取り違えるエンコード曖昧性も回避します。

```sh
claude-resume-all --recover                 # 現在のプロジェクトのクラッシュ分
claude-resume-all --recover --all-projects  # 強制終了した全プロジェクト分
```

> ⚠️ クラッシュ後は**早めに**実行してください。Claude Code は古いレジストリを定期的に掃除します。消えた後は `--recent` / `--since`(各セッションファイルの最終更新時刻でフィルタ。これは消えない)を保険に使ってください。

### フィルタと並び順

| フラグ | 効果 |
|------|--------|
| `--recent <N>` | `90m` / `3h` / `1d` 以内にアクティブだったものだけ |
| `--since <時刻>` | `HH:MM`(本日)または `YYYY-MM-DD HH:MM` 以降にアクティブだったものだけ |
| `--order recent` | 新しい順に開く(**既定**) |
| `--order old` | 古い順に開く。最後に触ったセッションが最後に開き、最前面に来る |
| `--recursive` / `-r` | プロジェクト配下のサブディレクトリから起動したセッションも含める |
| `--all-projects` | (`--recover` と併用)プロジェクトフィルタを無視 |

### ターミナルのバックエンド(`--app`)

| 値         | 動作 |
|------------|------|
| `terminal` | macOS Terminal.app。セッションごとに1ウインドウ(**既定**) |
| `iterm`    | iTerm2。セッションごとに1ウインドウ |
| `vscode`   | **一時的な** `.vscode/tasks.json` を生成し、VS Code 内で生成済みの *Resume ALL* タスクを1回実行(後述) |
| `print`    | resume コマンドを表示するだけ。tmux / WezTerm / 独自ランチャーへパイプ |

`terminal` / `iterm` / `vscode` は `osascript` を使うため **macOS 専用**。`print` はどの OS でも動きます。

### VS Code の統合ターミナルで開く

外部 CLI から VS Code の統合ターミナルを直接起動することは**できず**、VS Code が実行できるタスクは `<project>/.vscode/tasks.json` のものだけです。そこで `--app vscode` は:

1. セッションごとの専用ターミナルタスクと、複合タスク **▶ Resume ALL (<profile>)** を含む `.vscode/tasks.json` を生成(既存のタスクは保持)。
2. そのタスクを1回実行するよう案内
   (`Cmd+Shift+P` → *タスク: タスクの実行* → *▶ Resume ALL …*)。
3. 待機します — ターミナルが開いたら **Enter** を押すと `tasks.json` を**削除/復元**します。ターミナルは起動後も動き続けるため、リポジトリには何も残りません。`--keep` で残すことも可能。

## プロファイル

**プロファイル**とは要するに Claude の config dir です。`~/.claude` と `~/.claude-*` から自動検出し、basename をプロファイル名にします(`~/.claude` → `default`、`~/.claude-work` → `work`)。

1つのプロジェクトが複数プロファイルにセッションを持つ場合は、`-p` で選ぶよう促します。プロファイル名やパスは一切ハードコードされておらず、**実行時に発見**されます。

### 任意の設定ファイル

`${XDG_CONFIG_HOME:-~/.config}/claude-resume-all/config`(すべて任意):

```ini
# 既定のターミナルバックエンド
app=terminal
# 既定で実行する実行ファイル
launcher=claude
# 非標準の場所にある config dir を追加プロファイルとして登録
profile.foo=~/some/where/.claude-foo
# プロファイル別の launcher(自分のシェルエイリアスなど)
launcher.foo=claude-foo
```

環境変数での上書き: `CLAUDE_RA_APP`, `CLAUDE_RA_LAUNCHER`, `CLAUDE_CONFIG_DIR`, `CLAUDE_BIN`。

## 注意点

- GUI バックエンドは macOS 専用。`--app print` は移植可能。
- 初回は、ターミナルが Terminal.app / iTerm を制御する許可(オートメーション権限)を macOS が求めることがあります。
- これは Claude Code のディスク上のセッション配置に依存します。これは**ドキュメント化された安定 API ではない**ため、将来の Claude Code で変わる可能性があります。

## ライセンス

[MIT](LICENSE)
