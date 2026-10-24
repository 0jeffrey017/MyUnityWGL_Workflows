# Unity WebGL CI（任意利用）

`.github/workflows/reusable-unity-webgl.yml` は `workflow_call` 専用。自動実行する呼び出し元は追加していない。GDD の正式対象は引き続き Windows PC であり、WebGL の動作保証や MVP の受け入れ条件は変更しない。

公式 Unity CLI を Ubuntu 22.04 にインストールし、`unity build --execute-method ProjectDefense.Editor.CI.WebGLBuild.Build` でビルドする。game-ci コンテナや Unity Hub の旧 CLI は使用しない。

## 呼び出し例

利用する場合だけ、次の内容を別の `.github/workflows/webgl.yml` として追加する。Pages を公開する際は、リポジトリ Settings → Pages → Source を **GitHub Actions** に設定し、`github-pages` environment の公開ブランチ制限も確認する。

```yaml
name: Optional WebGL
on:
  push:
    branches: [main, master]
  pull_request:

permissions: {}

jobs:
  webgl:
    # Fork PRs and Dependabot do not receive the license secret.
    if: github.actor != 'dependabot[bot]' && (github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository)
    permissions:
      contents: read
      pages: write
      id-token: write
    uses: ./.github/workflows/reusable-unity-webgl.yml
    with:
      project_path: .
      unity_version: ''
      cli_version: latest
      use_burst: false
      deploy: true
    secrets:
      UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
```

この例を追加すると、main/master への push で公開される。同一リポジトリの PR はビルドのみ。PR のコードはビルド時に実行されるため、ライセンス secret を渡す呼び出し元には信頼できる変更だけを許可する。公開しない呼び出し元は `deploy: false` とし、`pages: write` と `id-token: write` を省略できる。再利用先は呼び出し元より権限を増やせないため、公開する場合は例の権限が必要。

| Input | 既定値 | 意味 |
|---|---|---|
| `project_path` | `.` | チェックアウト内の Unity プロジェクト。ビルド用 C# もこの配下に必要 |
| `unity_version` | 空文字 | `ProjectSettings/ProjectVersion.txt` の正確な版を使用。上書きも `6000.3.8f1` 等の完全な版のみ |
| `cli_version` | `latest` | 公式インストーラーの最新版。固定時は `1.0.0-beta.14` 等を指定し、`unity self-update --target ... --yes` で切り替える |
| `use_burst` | `false` | false のときだけ `--args '--burst-disable-compilation'` を渡す。true は通常の Burst 設定を尊重する |
| `deploy` | `false` | true かつ main/master への push の場合のみ Pages 公開 |

`use_burst: true` は Burst パッケージの導入や強制有効化を行わない。パッケージがなければ通常どおりビルドする。

## ライセンス

リポジトリまたは組織 secret の `UNITY_LICENSE` に、利用権限のある ULF の**生の内容**を保存する。Base64 やファイル名は渡さない。ワークフローは権限を制限した一時ファイルへ書き、公式 CLI の `unity license activate --file` を実行する。メールアドレスやパスワードによる認証は行わない。

ULF は機器の識別情報に結び付く。手元の PC 用 ULF を GitHub-hosted runner へコピーしても利用できるとは限らない。Unity の手動アクティベーションは、要求を生成したコンピューターで有効化する手順であり、ライセンス種別の制限もある。使い捨て runner でのオフラインアクティベーションと無人ビルドを契約上・技術上許可できることを、ライセンス管理者に確認する必要がある。この条件を満たさない場合、このテンプレートはライセンス認証で停止する。個人の認証情報へ自動的に切り替える処理はない。

アクティベーションを試みた場合は、`always()` の終了処理で `unity license return --yes` を実行する。一時 ULF と非公開のライセンス操作ログは返却失敗時も削除する。返却失敗はジョブ失敗として表示するので、管理者がシート状態を確認する。ライセンス内容・操作ログは成果物へアップロードしない。runner の強制終了や障害では終了処理を保証できないため、シート管理側で残存を確認する。

## ビルドと成果物

- Editor と WebGL モジュールは、解決した完全な版を `unity install <version> --module webgl --child-modules --yes --accept-eula` で導入する。
- 有効な `EditorBuildSettings.scenes` を使用する。空なら明示的に失敗し、存在しないシーンも失敗にする。Inspector への新たなコンポーネント設定は不要。
- C# は `CI_WEBGL_OUTPUT` の出力先を使用する。ビルド中だけ WebGL compression を Disabled、threads を false にし、`finally` で元へ戻す。プロセス強制終了時は `finally` を保証できないため、通常利用はこの使い捨て CI runner を想定する。
- Pages 用の圧縮レスポンスヘッダーやスレッド用の分離ヘッダーに依存しない設定にする。圧縮無効のため転送量は増える。WebGL のブラウザー上の実動作は別途確認する。
- Library cache は OS・プロジェクト・Editor 完全版・WebGL・Burst 指定・Packages/ProjectSettings のハッシュで分離し、Assets のハッシュを完全キーへ含める。互換範囲の cache がなければインポートから実行する。
- checkout は LFS を取得し、`git lfs fsck` を実行する。Git の reset/add、LFS 設定の恒久変更、ディスク消去は行わない。
- `unity-webgl-<run_id>-<run_attempt>` に WebGL 出力、`unity-webgl-diagnostics-<run_id>-<run_attempt>` にインストールログ・CLI 版/help・Editor ビルドログ・provenance を保存する（7 日）。診断情報は Pages 出力先の外に置く。
- ビルドジョブは `contents: read`、公開ジョブだけが `pages: write` / `id-token: write` を持つ。concurrency は公開ジョブだけに設定し、実行中の公開をキャンセルしない。PR の実行は公開処理をキャンセルしない。
- ジョブと各長時間処理に timeout を設定している。キャッシュ保存に失敗した場合は、次の実行で再インポートする。

## 検証記録（2026-10-24）

ローカルの公式 Unity CLI は `1.0.0-beta.14`。`build` / `install` / `license activate` / `license return` / `self-update` の help と公式文書で使用するフラグを確認した。最新版の更新確認は CDN の DNS 解決失敗のため完了していない。CI は実行時に公式インストーラーで最新を取得する。

GitHub Actions 上でのインストール、ライセンス認証、WebGL ビルド、ブラウザー動作、Pages 公開は未実行。リポジトリへ push・commit・公開も行っていない。ローカルの構文検証と Editor コンパイルの結果は `Docs/QAReport.md` の本作業記録を参照する。

## 確認した公式資料

- [Unity CLI のインストールと版固定](https://docs.unity.com/en-us/unity-cli/use-unity-cli)
- [Unity CLI コマンドリファレンス](https://docs.unity.com/en-us/unity-cli/unity-cli-reference)
- [Unity の手動ライセンスアクティベーション](https://docs.unity3d.com/6000.3/Documentation/Manual/ManualActivationGuide.html)
- [WebGL compressionFormat](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/PlayerSettings.WebGL-compressionFormat.html) / [threadsSupport](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/PlayerSettings.WebGL-threadsSupport.html)
- [GitHub Pages のカスタムワークフロー](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages)
