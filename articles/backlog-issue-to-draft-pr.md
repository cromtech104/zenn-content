---
title: "Backlogの課題を投げると、調査ログ付きのDraft PRが返ってくる仕組み"
emoji: "🔧"
type: "tech"
topics: ["backlog", "claude", "githubapp", "個人開発", "ai"]
published: true
---

Backlogでバグのチケットを受け取ると調査から始まるが、どのファイルか、なぜこうなってるかを調べるだけで30分以上かかる場合があります。実装より調査の方が長い日も多々発生しますね。

そこで、課題本文を読ませて関連コードを自動で調べさせ、何を根拠にどう直したかをPR本文に残したDraft PRまで作るようにしています。自動マージはしないので、少し丁寧なレビューをするだけという感覚で使っています。

この記事では、Claude CodeとBacklog MCPでそれを手元で組む手順を、動くコード付きで書いております。

## 作るものの全体像

流れはシンプルで、4ステップです。

1. Backlogから課題（タイトルと本文）を取る
2. Claudeに「まず調査だけ」させて、どのファイルが怪しいか・なぜかを出させる（＝調査ログ）
3. その調査をもとに、修正内容をJSONで出させる
4. ブランチを切って変更をコミットし、調査ログをPR本文に貼った **Draft PR** を開く

ポイントは2つだけ意識しています。**いきなり修正を書かせず、先に調査させること**。そして**Draftで止めること**です。AIの修正は外すこともあるので、根拠を見て人間が判断する前提にしています。

必要なものは、BacklogのAPIキーとスペース、AnthropicのAPIキー、GitHubのトークン（後述のとおりGitHub Appが楽です）、あとはPythonくらいです。

## ① Backlogから課題を取る

タイトルにMCPと書きましたが、中身はBacklogのREST APIです。Backlog MCPを使うとこの取得が「ツール呼び出し」になるだけで、叩いているものは同じなので、ここでは分かりやすさのため直接書きます。

```python
import os
import requests

BACKLOG_SPACE = os.environ["BACKLOG_SPACE"]      # 例: example.backlog.com
BACKLOG_API_KEY = os.environ["BACKLOG_API_KEY"]

def get_issue(issue_key: str) -> dict:
    url = f"https://{BACKLOG_SPACE}/api/v2/issues/{issue_key}"
    r = requests.get(url, params={"apiKey": BACKLOG_API_KEY})
    r.raise_for_status()
    return r.json()
```

BacklogのAPIキーはクエリパラメータで渡す方式です。取れる`summary`（タイトル）と`description`（本文）を、このあとそのままClaudeに渡します。

## ② まず「調査だけ」させる

ここが一番効きます。最初から「直して」と言うと、的外れなファイルをいじったり、関係ないところまで書き換えたりします。なので最初のターンは**コードを書かせず、どこを見るべきかと、その理由だけ**を出させます。これがそのまま調査ログになります。

```python
import anthropic

client = anthropic.Anthropic()  # ANTHROPIC_API_KEY を読む

def investigate(issue: dict, repo_tree: str) -> str:
    prompt = f"""次のBacklog課題を直すために、関連しそうなファイルと原因を調べてください。
この段階ではコードを書かないでください。どのファイルを見るべきか、なぜそう考えたかだけを箇条書きで出してください。

# 課題
{issue['summary']}
{issue.get('description', '')}

# リポジトリのファイル一覧
{repo_tree}
"""
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2000,
        messages=[{"role": "user", "content": prompt}],
    )
    return msg.content[0].text
```

`repo_tree`はリポジトリのファイル一覧です。GitHubのTree APIで取るか、ローカルなら`git ls-files`の結果で十分です。全ファイルの中身まで渡すとトークンが一気に膨らむので、この段階では**一覧だけ**にしています。

返ってくるのは「`app/services/cart.py`の割引計算が怪しい。理由は〜」のような文章です。これをあとでPR本文に貼ります。

## ③ 修正をJSONで出させる

調査でファイルが絞れたら、そのファイルの中身だけを渡して修正を作らせます。出力はJSONに固定します。説明文を地の文で混ぜられるとパースが面倒なので、理由もJSONの中に入れてもらいます。

```python
import json

def generate_fix(issue: dict, investigation: str, file_contents: str) -> dict:
    prompt = f"""さきほどの調査をもとに、修正を作ってください。
出力は次の形のJSONだけにしてください。前後に説明文やコードフェンスを付けないでください。

{{"reason": "なぜこう直すか", "changes": [{{"path": "対象パス", "new_content": "修正後の全文"}}]}}

# 課題
{issue['summary']}

# 調査結果
{investigation}

# 対象ファイルの中身
{file_contents}
"""
    msg = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=4000,
        messages=[{"role": "user", "content": prompt}],
    )
    return json.loads(_strip_to_json(msg.content[0].text))
```

「JSONだけ」と書いても、たまに```json ... ```で囲んで返してきます。そこを素直に剥がす関数を1個挟んでおくと安定します。

```python
def _strip_to_json(text: str) -> str:
    text = text.strip()
    if text.startswith("```"):
        # 最初と最後のコードフェンス行を落とす
        text = "\n".join(text.split("\n")[1:-1])
    return text
```

この「2ステップに分ける」「JSONで受けて崩れを直す」あたりは、前に別記事で詳しく書いたので、そちらも置いておきます（[AIにコード修正を生成させる2ステップprompting](https://zenn.dev/repocarta/articles/ai-two-step-code-generation)）。

## ④ 調査ログ付きのDraft PRを開く

最後に、ブランチを切って変更をコミットし、Draft PRを開きます。ここで**PR本文に②の調査ログと③の理由を貼る**のが、この仕組みのいちばん大事なところです。レビューする側は、何を根拠に直したのかがそのまま読めます。

```python
from github import Github

gh = Github(os.environ["GITHUB_TOKEN"])
repo = gh.get_repo("owner/name")

def open_draft_pr(issue_key: str, fix: dict, investigation: str, base: str = "main") -> str:
    branch = f"auto/{issue_key}"
    base_ref = repo.get_branch(base)
    repo.create_git_ref(ref=f"refs/heads/{branch}", sha=base_ref.commit.sha)

    for ch in fix["changes"]:
        current = repo.get_contents(ch["path"], ref=branch)
        repo.update_file(
            ch["path"],
            message=f"fix: {issue_key}",
            content=ch["new_content"],
            sha=current.sha,
            branch=branch,
        )

    body = f"""## このPRの変更理由
{fix['reason']}

## 調査ログ
{investigation}

---
自動生成のDraftです。マージ前に必ず内容を確認してください。
"""
    pr = repo.create_pull(
        title=f"{issue_key} の自動修正案",
        body=body,
        head=branch,
        base=base,
        draft=True,  # ここがDraft
    )
    return pr.html_url
```

`draft=True`でDraftになります。これで、レビュー前に勝手にマージされる事故は起きません。

トークンを使うので、GitHubの認証はGitHub Appのインストールトークンにしておくと、リポジトリ単位で権限を絞れて楽です。Appの認証はそこそこ罠があったので、これも別記事に分けました（[GitHub Appの認証、思ったより罠が多かった](https://zenn.dev/repocarta/articles/github-app-auth-pitfalls)）。

## ハマったところ

- **ファイルを絞らないとトークンが膨らむ**。②の調査で対象を絞って、③では本当に必要なファイルの中身だけ渡すようにしました。最初は全部渡して、毎回それなりの金額になっていました。
- **`new_content`で全文を返す方式は、大きいファイルだと高い**。差分だけ返させる方法も試しましたが、行ズレで壊れやすかったので、いまは全文方式で割り切っています。ここは好みだと思います。
- **JSONはたまに崩れる**。③の剥がす処理を入れてからは安定しました。
- **自動マージは入れていない**。一度入れかけましたが、AIの修正は外すこともあるので、根拠を見て人が判断する前提のほうが結局速い、という結論になりました。Draftで止めているのはそのためです。

## おわりに

ここまでが手元版です。実際に毎日回すとなると、Backlogのwebhookで課題作成を拾って自動で起動する、トークン上限や対象外フォルダの設定を入れる、失敗したときの通知を出す、といった運用の手当てが要ります。そのあたりまで含めて面倒を見るものを作っていて、[keros](https://keros.repocarta.jp/)というサービスにしていますが、仕組み自体はこの記事のとおりなので、まずは手元で動かしてみるのが一番理解が早いと思います。
