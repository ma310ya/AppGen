# AI App Generator

ブラウザーで操作できるAI開発パイプラインです。Web画面はPython標準ライブラリのHTTPサーバーで提供します。AIプロバイダーはGeminiとGitHub Copilotに対応しています。

## このアプリでできること

作りたいものや変更内容を入力し、AIに単一ファイルのPythonコードを作成・改修させて、選択したGitHubリポジトリへ反映するアプリです。ブラウザー画面で要件、保存先のリポジトリと `.py` ファイル名、仕様書をまとめるAIモデル、コード生成AIモデルを指定できます。要件定義ファイル（`.md` / `.txt`）を読み込ませることもできます。既存ファイルの内容があれば、それをもとに改修します。

実行時は、まず要件から仕様を作成し、その仕様に沿ってPythonコードを生成します。生成コードは `py_compile` で構文チェックされ、構文エラーがあればエラー内容をAIに渡して最大3回まで再生成します。成功すると対象ファイルを選択したGitHubリポジトリのデフォルトブランチへ自動でコミット・プッシュします。進行ログや仕様、生成コード、エラー詳細、およびコードから抽出した追加Pythonパッケージの候補は画面で確認できます。

このパイプラインが行うテストはPythonの構文チェックのみです。生成したプログラムの実行、機能テスト、追加パッケージの自動インストールは行いません。プッシュ先は選択リポジトリのデフォルトブランチで、別ブランチを指定する機能はありません。

## 前提条件

- Python 3.12以上。生成したコードの構文チェックに `python3` コマンドを使うため、サーバーのPATHから実行できるようにしてください。
- Gitコマンド。GitHubリポジトリの取得・コミット・同期に使用します。
- GitHub、Google AI、Copilotの各サービスへHTTPSで接続できるネットワーク。
- 下記のPythonパッケージ。リポジトリに `requirements.txt` は含まれていないため、仮想環境を作成して個別にインストールします。

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   python -m pip install --upgrade pip
   python -m pip install requests cryptography litellm langgraph
   ```

   Windowsでは仮想環境を有効化するコマンドが異なります（PowerShell: `.venv\Scripts\Activate.ps1`）。ただし、生成コードの構文チェックでも `python3` が必要なため、そのコマンドが使える環境で起動してください。

## サーバー環境変数

すべて任意です。未設定時は表の既定値を使用します。

| 変数 | 既定値 | 用途 |
| --- | --- | --- |
| `HOST` | `0.0.0.0` | HTTPサーバーの待ち受けアドレス |
| `PORT` | `8000` | HTTPサーバーの待ち受けポート |
| `GITHUB_OAUTH_CLIENT_ID` | アプリ内蔵のClient ID | GitHub OAuth Appを差し替える場合に指定 |
| `GITHUB_COPILOT_CLIENT_ID` | アプリ内蔵のClient ID | Copilot Device FlowのClient IDを差し替える場合に指定 |
| `APP_CREDENTIALS_FILE` | `~/.config/ai-dev-orchestrator/credentials.enc` | 暗号化した認証情報ファイルの保存先 |
| `APP_CREDENTIALS_KEY` | 未設定（初回起動時に鍵ファイルを自動生成） | Fernet形式の暗号化鍵。指定する場合は鍵を秘密として管理し、認証情報ファイルと一緒に永続化・バックアップ |

`APP_CREDENTIALS_KEY` を指定しない場合、鍵は認証情報ファイルと同じディレクトリの `credentials.key` に保存されます。`APP_CREDENTIALS_FILE` を変更すると鍵のファイル位置もそのディレクトリに追随します。既存の認証情報を保持するには、認証情報ファイルと対応する鍵の両方を維持してください。

## 利用可能なAI環境

AIプロバイダーは **Google Gemini** または **GitHub Copilot** を利用できます。どちらか一方を準備すればAIによる生成を実行できます。両方を設定すると、画面でプラン用・コード生成用のモデルをそれぞれ選択できます。

| プロバイダー | 必要なもの | アプリでの設定 |
| --- | --- | --- |
| Google Gemini | Google AI Studioで作成した有効なGemini APIキーと、利用可能なモデル・クォータ | 認証設定画面のGemini APIキー欄に入力して検証・保存 |
| GitHub Copilot | Copilotを利用できるGitHubアカウント、アカウントのプランで利用可能なモデル・クォータ | 認証設定画面からCopilotにログインし、Device Flowを完了 |

APIキーやAI用トークンをサーバー起動時の環境変数に設定する必要はありません。認証設定画面で登録した認証情報は暗号化して保存されます。Geminiは登録したAPIキーで利用可能なモデルを取得します。Copilotの利用可能モデルはアカウントやプランによって異なるため、画面で選べるモデルを利用してください。利用枠や課金の条件は各プロバイダーのアカウント設定に従います。

GitHub CopilotのログインはAIモデル利用のため、GitHubログインはリポジトリの一覧取得・書き込みのための別認証です。生成結果をGitHubリポジトリへ同期する場合は、GitHubにもログインし、対象リポジトリへの書き込み権限を持つアカウントを使用してください。AIモデルを利用するには、サーバーから各プロバイダーのAPIへHTTPS接続できる必要があります。

## GitHub Codespacesで実行

1. 上記の前提条件を満たし、CodespacesのターミナルでPythonパッケージをインストールします。

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   python -m pip install --upgrade pip
   python -m pip install requests cryptography litellm langgraph
   ```

2. 認証情報の暗号化鍵は初回起動時にサーバー内へ自動生成されます。鍵の保存先は認証情報と同じディレクトリの `credentials.key` です。永続ディスクでこの鍵と `credentials.enc` の両方を保持してください。複数サーバー間で鍵を共有する場合や鍵管理サービスを使う場合は、環境変数 `APP_CREDENTIALS_KEY` で上書きできます。

   ```bash
   python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
   ```

3. GitHubリポジトリ操作用OAuth Appを一度だけ登録します。GitHubの **Settings → Developer settings → OAuth Apps → New OAuth App** で次を入力してください。

   - Application name: `AI Dev Orchestrator`
   - Homepage URL: `https://github.com/ma310ya/ai-dev-orchestrator`
   - Authorization callback URL / Redirect URI: `https://github.com/ma310ya/ai-dev-orchestrator`（Device Flowでは使用しませんが、登録フォームで必須の場合に入力）
   - **Enable Device Flow**: 有効

   登録後に表示されるClient IDが、このアプリ共通の公開識別子です。現在のClient IDはアプリに組み込み済みのため、通常は追加の設定は不要です。別のOAuth Appを使う場合だけ、サーバーの `GITHUB_OAUTH_CLIENT_ID` 環境変数で上書きしてください。Client secretはDevice Flowでは使わず、アプリに設定しないでください。このClient IDを使って各利用者が自分のGitHubアカウントでログインします。GitHub認証は書き込み可能なリポジトリ一覧と同期のため `repo` スコープを要求します。**OAuth Appの設定で「Enable Device Flow」を有効にして保存してください。** ログイン開始でHTTP 400になる場合はこの設定、Client ID、設定の保存を確認してください。CopilotのDevice FlowはLiteLLMのGitHub Copilot連携を利用します。
4. アプリを起動します。

   ```bash
   python app.py
   ```

5. Codespacesの「ポート」タブでポート `8000` を開き、表示された転送URLにブラウザーでアクセスします。一般サーバーではリバースプロキシ等から公開URLへ転送してください。サーバーは `0.0.0.0:8000` で待ち受けます。

## 独立したサーバーで実行

Codespacesは開発・試用用の実行環境であり、必須ではありません。前提条件を満たすLinux VPS、クラウドVM、または社内サーバーにこのリポジトリを配置すれば、Codespacesの起動・ポート転送に依存せず独立して稼働できます。サーバーにPython 3.12以上とGitを用意し、リポジトリのディレクトリで仮想環境と依存パッケージをインストールします。

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install requests cryptography litellm langgraph
python app.py
```

既定では `0.0.0.0:8000` で待ち受けます。サーバーの再起動後も稼働させるには、systemdなどのプロセス管理ツールに登録して自動起動・再起動を設定してください。外部から利用する場合は、HTTPS対応のリバースプロキシを経由させ、必要に応じて `HOST` / `PORT` を設定します。Codespacesと同様、認証済みリバースプロキシやVPNなどでアクセスを制限し、アプリのポートを信頼できないネットワークへ直接公開しないでください。認証情報ファイルと暗号化鍵を永続ストレージに置き、両方をバックアップしてください。

認証設定画面からGitHubとCopilotはブラウザーのDevice Flowでログインし、GeminiはAPIキーを登録します。Gemini APIキーは[Google AI StudioのAPIキー画面](https://aistudio.google.com/app/apikey)で作成し、アプリの認証設定に貼り付けて「Geminiキーを検証して暗号化保存」を押してください。GitHubログイン完了後、書き込み可能なリポジトリ一覧を自動で再取得します。Gemini APIキーを登録すると、利用可能なGeminiモデルがモデル選択肢に動的に追加されます。Copilotで利用できるモデルはアカウントやプランにより異なります。モデル未対応のエラーが出た場合は、別のCopilotモデルまたはGeminiモデルを選んでください。暗号化された認証情報は既定で `~/.config/ai-dev-orchestrator/credentials.enc`、暗号化鍵は同じ場所の `credentials.key` に保存します。永続ディスクを使う一般サーバーでは、必要に応じて `APP_CREDENTIALS_FILE` で保存先を指定してください。GitHub/Copilotの有効期限が切れた場合や認証が拒否された場合は再ログインを促し、Gemini APIキーは失効が検出されるまで保存します。選んだリポジトリのデフォルトブランチへ同期します。`PORT` 環境変数を設定すると待ち受けポートを変更できます。

**運用上の注意:** このアプリ自体には利用者アカウント／セッション認証を実装していません。一般サーバーではHTTPSに加えて、認証済みリバースプロキシ、VPN、または同等のアクセス制限の内側に配置し、信頼できないネットワークへ直接公開しないでください。暗号化鍵と認証情報ファイルの両方を適切にバックアップし、鍵を紛失した場合は保存済み認証情報を復号できません。
