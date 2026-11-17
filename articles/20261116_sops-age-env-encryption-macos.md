---
title: ".envを平文で置かない — SOPS + age で、鍵2つだけでAPIキーをチーム共有する（macOS編）"
emoji: "🔐"
type: tech
topics:
  - SOPS
  - age
  - direnv
  - セキュリティ
  - macOS
published: true
---

:::message
**Key Points**
- チームで共有する秘密は、repoごとに **dev鍵とprod鍵の2つだけ** です。APIキーが何個増えても変わりません
- prod鍵は端末に置かず、GitHub Secretsとクラウド上の暗号化ファイルにだけ置くので、端末が侵害されても本番の値は守られます
- `.env`の便利さ（ディレクトリに入れば使える）はdirenvでそのまま残ります
:::

# はじめに

`.env`は便利ですが、端末に平文で残り、公開repoへの混入事故も後を絶ちません。`.env.example`の更新漏れや、チャットでのキーの受け渡しに困っているチームも多いと思います。

本記事では、**SOPS** と **age** で`.env`を暗号化してgitで共有する方法を、macOSのコマンドで紹介します（Linux編は別記事）。言いたいことは1つで、**チームで共有する秘密鍵はdev鍵とprod鍵の2つだけで済む** ということです。

対象はSSOが存在しないAPIキー・認証情報です。AWSのようにSSO（IAM Identity Center / OIDC）で短期クレデンシャルを使えるものはSOPSに入れません。KMSやVaultを使う大規模な構成も扱いません。想定は3人程度までの小さなチームです。

# 共有する秘密鍵はdev鍵とprod鍵の2つだけ

## dev鍵は端末に、prod鍵はGitHubにだけ置く

![20261116_sops-age-overview](https://pub-61138f8207b54a509ef5e6195db54d4f.r2.dev/images/articles/20261116_sops-age-overview.png)

| 要素 | 置き場所 | 役割 |
| --- | --- | --- |
| dev秘密鍵 | メンバー全員の端末 + クラウド（パスフレーズで暗号化） | `dev.enc.env`の復号 |
| prod秘密鍵 | GitHub prod Environment Secrets + クラウド（パスフレーズで暗号化） | `prod.enc.env`の復号（CDのみ） |
| dev / prod公開鍵 | `.sops.yaml`（repoにコミット） | 暗号化 |
| `secrets/dev.enc.env` / `secrets/prod.enc.env` | repo（gitで追跡） | 暗号化された値 |

APIキーをいくつ追加しても、メンバーに渡すのはdev鍵1つ、GitHubに登録するのはprod鍵1つです。値の漏洩には「暗号化ファイル」と「該当する秘密鍵」の両方が必要なので、repoが漏れても鍵がなければ読めません。

鍵の保管と配布には、クラウド同期フォルダを使います。秘密鍵は`age -p`でパスフレーズ暗号化してから置くので、メンバーが覚えておくのはパスフレーズだけです。鍵はrepoごとに作るので、1つ漏れても影響はそのrepoに限られます。

## メリットとデメリット

![20261116_sops-pros-cons](https://pub-61138f8207b54a509ef5e6195db54d4f.r2.dev/images/articles/20261116_sops-pros-cons.png)

鍵を2つに絞る代わりに、prodの編集とメンバー離脱時の作業に手間がかかります。チームが大きくなり、目安として4人以上になったら、AWS KMSなどへの移行を検討してください。

## 暗号化は公開鍵で誰でもでき、復号は秘密鍵を持つ人だけができる

SOPSは`.env`の **値だけ** をage公開鍵で暗号化します。公開鍵は`.sops.yaml`にあるので誰でも暗号化でき、復号には秘密鍵が必要です。そのため、メンバーの端末はprodの値を書き込めても、読むことはできません。

![20261116_sops-encrypt-decrypt](https://pub-61138f8207b54a509ef5e6195db54d4f.r2.dev/images/articles/20261116_sops-encrypt-decrypt.png)

暗号化後のファイルは、キー名が平文のまま残ります。

```bash
>>> input
cat secrets/dev.enc.env

>>> output
API_KEY=ENC[AES256_GCM,data:...,iv:...,tag:...,type:str]
sops_age__list_0__map_enc=-----BEGIN AGE ENCRYPTED FILE-----\n...
sops_age__list_0__map_recipient=<devPubkey>
sops_lastmodified=2026-11-16T00:00:00Z
sops_mac=ENC[AES256_GCM,data:...,iv:...,tag:...,type:str]
sops_unencrypted_suffix=_unencrypted
sops_version=3.13.3
```

![20261116_sops-age-dev-enc-sec](https://pub-61138f8207b54a509ef5e6195db54d4f.r2.dev/images/articles/20261116_sops-age-dev-enc-sec.png)

# Phase 1：鍵を2つ作り、dev鍵だけをメンバーに配る

## 1-1. ツールを入れて、鍵の置き場所を決める

メンバー全員が同じバージョンを入れます。

```bash
>>> input
brew install sops age direnv gitleaks gh
sops --version --check-for-updates; age --version; direnv version; gitleaks version

>>> output
~~~~ 省略 ~~~~
sops 3.13.3 (latest)
v1.3.2
2.38.2
8.30.1
```

`~/.zshrc`に追記し、シェルを再起動します。`KEYSTORE`は、パスフレーズで暗号化した鍵ファイルを置くクラウド同期フォルダです。iCloud Driveのパスはスペースを含むので、必ず引用符で囲みます。

```bash
eval "$(direnv hook zsh)"
export KEYSTORE="$HOME/Library/Mobile Documents/com~apple~CloudDocs/sops-keys"
```

## 1-2. 平文で置いていた値はすべて再発行する

gitの全履歴を走査し、過去にコミットされた値を探します。見つかった値や、これまで`.env`に平文で置いていた値は、発行元で再発行します（新しい値で動作を確認してから旧値を失効させます）。

```bash
>>> input
gitleaks git --redact -v

>>> output
    ○
    │╲
    │ ○
    ○ ░
    ░    gitleaks

INF <N> commits scanned.
INF scanned ~<size> bytes (<size>) in <time>
INF no leaks found
```

## 1-3. dev鍵を作り、パスフレーズで暗号化してクラウドに置く

鍵はrepoごとに作ります。以降の`<repoName>`は、repoの名前に置き換えてください。`age-keygen`の出力をそのまま`age -p`で暗号化するので、平文の鍵はディスクに残りません。

```bash
>>> input
umask 077; mkdir -p "$KEYSTORE" ~/.config/sops/age
age-keygen | age -p -o "$KEYSTORE/<repoName>.dev.agekey.age"

>>> output
Public key: <devPubkey>
Enter passphrase (leave empty to autogenerate a secure one):
Confirm passphrase:
```

パスフレーズを2回入力します。表示される`Public key`（`age1...`）は、後で`.sops.yaml`に書きます。

:::message alert
暗号化ファイルはクラウドにあるので、手に入れた人はオフラインで何度でもパスフレーズを試せます。パスフレーズは十分に長くしてください。忘れると復旧できません。パスフレーズの運用は別記事で扱います。
:::

## 1-4. prod鍵も同じようにクラウドに置く

prod鍵も同じコマンドで作ります。端末には一度も書き出されません。

```bash
>>> input
age-keygen | age -p -o "$KEYSTORE/<repoName>.prod.agekey.age"

>>> output
Public key: <prodPubkey>
Enter passphrase (leave empty to autogenerate a secure one):
Confirm passphrase:
```

## 1-5. 各メンバーはクラウドから復号してdev鍵を配置する

クラウドの同期が終わっていることを確かめてから、各メンバー（作成した本人も）が自分の端末で実行します。パスフレーズを入力して復号できれば、配置は完了です。

```bash
>>> input
umask 077; mkdir -p ~/.config/sops/age
age -d -o ~/.config/sops/age/<repoName>.agekey "$KEYSTORE/<repoName>.dev.agekey.age"

>>> output
Enter passphrase:
```

# Phase 2：repoに暗号化ファイルを置き、devはdirenv、prodはCDで復号する

## 2-1. `.sops.yaml`でファイルごとに暗号化する鍵を決める

```yaml
creation_rules:
  - path_regex: secrets/dev\.enc\.env$
    age: <devPubkey>
  - path_regex: secrets/prod\.enc\.env$
    age: <prodPubkey>
```

:::message alert
`path_regex`はglobではなく **正規表現** です。
:::

## 2-2. dev / prodの値を`sops edit`で書き込む

`sops edit`は、ファイルがなければ`.sops.yaml`のルールに従って新しく作ります。エディタが開くので`KEY=value`を1行ずつ書いて保存すると、暗号化されて保存されます。prodも公開鍵だけで作れます。

```bash
>>> input
mkdir -p secrets
sops edit secrets/dev.enc.env
sops edit secrets/prod.enc.env

>>> output
（エディタがdev・prodの順に開く。KEY=valueを書いて保存すると出力なし）
```

## 2-3. ディレクトリに入るだけでdevの値を使えるようにする

repoのルートに`.envrc`を作ります。鍵ファイルを`SOPS_AGE_KEY_FILE`で指定し、復号結果をdirenvの`dotenv`でそのまま読み込みます。復号結果はファイルに書かれません。`.envrc`には値を書かないのでコミットして構いません。

```bash
export SOPS_AGE_KEY_FILE="$HOME/.config/sops/age/<repoName>.agekey"
watch_file secrets/dev.enc.env
dotenv <(sops decrypt secrets/dev.enc.env)
```

```bash
>>> input
direnv allow

>>> output
下の画像を参照
```

![20261116_sops-direnv-result](https://pub-61138f8207b54a509ef5e6195db54d4f.r2.dev/images/articles/20261116_sops-direnv-result.png)

## 2-4. 平文のコミットを.gitignoreとgitleaksで止める

```bash
>>> input
printf '.env\n.env.*\n*.agekey\n*.agekey.age\n' >> .gitignore
mkdir -p .githooks
printf '#!/bin/sh\ngitleaks git --pre-commit --staged --redact\n' > .githooks/pre-commit
chmod +x .githooks/pre-commit
git config core.hooksPath .githooks

>>> output
（出力なし）
```

`core.hooksPath`は、clone後に各メンバーが1回設定します。

## 2-5. prod鍵はGitHubのprod環境にだけ登録する

GitHubのSettings → Environmentsで`prod`環境を作り、デプロイできるブランチを`main`に限定します。そのうえで、クラウドのprod鍵を復号し、そのまま`gh`に渡して登録します。平文はディスクにもクリップボードにも残りません。

```bash
>>> input
age -d "$KEYSTORE/<repoName>.prod.agekey.age" | gh secret set SOPS_AGE_KEY --env prod

>>> output
Enter passphrase:
✓ Set Actions secret SOPS_AGE_KEY for prod environment in <owner>/<repoName>
```

![20261116_sops-github-env-secret](https://pub-61138f8207b54a509ef5e6195db54d4f.r2.dev/images/articles/20261116_sops-github-env-secret.png)

:::message
Environmentsはpublic repoか、Pro / Team / Enterpriseのprivate repoでのみ使えます。Freeのprivate repoではrepo Secretsで代用します。その場合はどのワークフローからも参照できるので、prod鍵を使うワークフローはデプロイ用の1つに限ってください。
:::

## 2-6. CDでは値をマスクしてから環境変数に渡す

```yaml
name: deploy
on:
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: prod
    steps:
      - uses: actions/checkout@v7
      - name: Install sops
        run: |
          curl -fsSL -o sops "https://github.com/getsops/sops/releases/download/v<sopsVersion>/sops-v<sopsVersion>.linux.amd64"
          chmod +x sops && sudo mv sops /usr/local/bin/sops
      - name: Decrypt prod secrets
        env:
          SOPS_AGE_KEY: ${{ secrets.SOPS_AGE_KEY }}
        run: |
          sops decrypt secrets/prod.enc.env | while IFS= read -r line; do
            [ -z "$line" ] && continue
            case "$line" in \#*) continue;; esac
            name="${line%%=*}"
            value="${line#*=}"
            [ -n "$value" ] && echo "::add-mask::$value"
            echo "$name=$value" >> "$GITHUB_ENV"
          done
      - name: Deploy
        run: echo "デプロイ処理をここに記述"
```

値は`::add-mask::`でマスクしてから`GITHUB_ENV`に渡します。順序が逆だとログに値が出ることがあります。

## 2-7. 端末でprodが復号できないことを確かめる

端末で`prod.enc.env`を復号してエラーになれば、意図どおりです。

```bash
>>> input
sops decrypt secrets/prod.enc.env

>>> output
Failed to get the data key required to decrypt the SOPS file.

Group 0: FAILED
~~~~ 省略 ~~~~
```

# Phase 3：値の編集も鍵の入れ替えも暗号化したまま行う

## 値は`sops edit`で暗号化したまま編集する

```bash
>>> input
sops edit secrets/dev.enc.env

>>> output
（エディタが開く。保存して閉じると出力なし。変更がなければ次のように表示される）
File has not changed, exiting.
```

prodは、クラウドのprod鍵をパスフレーズで一時的に復号して渡します。ファイルには書きません。

```bash
>>> input
SOPS_AGE_KEY="$(age -d "$KEYSTORE/<repoName>.prod.agekey.age")" sops edit secrets/prod.enc.env

>>> output
Enter passphrase:
（エディタが開く。保存して閉じると出力なし）
```

## 新しいメンバーはdev鍵を受け取ってcloneするだけで始められる

1-1と1-5を行い、repoをcloneして次を実行します。

```bash
>>> input
git config core.hooksPath .githooks
direnv allow

>>> output
direnv: loading ~/path/to/<repoName>/.envrc
direnv: export +<keyName> ... +SOPS_AGE_KEY_FILE
```

## メンバーが抜けたら鍵を入れ替え、値も再発行する

![20261116_sops-key-rotation](https://pub-61138f8207b54a509ef5e6195db54d4f.r2.dev/images/articles/20261116_sops-key-rotation.png)

新しい鍵に差し替え、残るメンバー全員に配り直します。配り終えるまでの間は誰も復号できませんが、数人のチームなら問題になりません。dev鍵の例で、番号は図と対応しています。

**① 新しい鍵を作る**

離脱したメンバーはパスフレーズを知っているので、**新しいパスフレーズ** を使います。旧鍵のファイルは、作業が終わるまで別名で残します。

```bash
>>> input
mv "$KEYSTORE/<repoName>.dev.agekey.age" "$KEYSTORE/<repoName>.dev.agekey.age.old"
age-keygen | age -p -o "$KEYSTORE/<repoName>.dev.agekey.age"

>>> output
Public key: <newPubkey>
Enter passphrase (leave empty to autogenerate a secure one):
Confirm passphrase:
```

**② 鍵を差し替え、データ鍵ごと作り直す**

表示された新しい公開鍵（`<newPubkey>`）で、`.sops.yaml`のdevの公開鍵を書き換えます。書き換える前の公開鍵が`<oldPubkey>`です。そのうえで、旧鍵がまだ手元にある端末で次を実行し、コミットしてpushします。`rotate`は、データ鍵を作り直すと同時に宛先を新しい鍵に差し替えます。

```bash
>>> input
sops rotate -i --add-age <newPubkey> --rm-age <oldPubkey> secrets/dev.enc.env

>>> output
（出力なし）
```

**③ 全員に配り直す**

残るメンバーに新しいパスフレーズを伝え、全員が1-5のコマンドで鍵を配置し直します（旧鍵のファイルは上書きされます）。クラウドフォルダの共有から離脱したメンバーを外し、`.old`を削除します。

**④ 値を再発行する**

APIキーを発行元で再発行し、`sops edit secrets/dev.enc.env`で差し替えてコミットします。

prod鍵の場合は、②で旧prod鍵を渡し、③で配る代わりにGitHub Secretsを更新します。

```bash
>>> input
# ②
SOPS_AGE_KEY="$(age -d "$KEYSTORE/<repoName>.prod.agekey.age.old")" \
  sops rotate -i --add-age <newPubkey> --rm-age <oldPubkey> secrets/prod.enc.env
# ③
age -d "$KEYSTORE/<repoName>.prod.agekey.age" | gh secret set SOPS_AGE_KEY --env prod

>>> output
Enter passphrase:
（②は出力なし）
Enter passphrase:
✓ Set Actions secret SOPS_AGE_KEY for prod environment in <owner>/<repoName>
```

:::message
鍵を入れ替えても、離脱したメンバーが過去に見た値は守れません。APIキーそのものも発行元で再発行してください。
:::

## 侵害されたら、その鍵で読める値をすべて入れ替える

| 事象 | 対応 |
| --- | --- |
| 端末の紛失・侵害 | dev鍵を全repoで入れ替え、devの値を再発行 |
| prod鍵を取り出している最中の侵害 | prod鍵も入れ替え、prodの値を再発行 |
| GitHub側の侵害（疑い） | prod鍵を入れ替え、prodの値を再発行 |
| クラウドと端末の両方の侵害 | パスフレーズを変え、該当repoの鍵をすべて入れ替える |

# Tips：よく使う操作を関数にする

`<repoName>`を毎回入力するのは手間です。`~/.zshrc`に次の関数を追記すると、repoの中で実行するだけでrepo名を自動で補えます。

```bash
_repo() { basename "$(git rev-parse --show-toplevel)"; }

# 1-5 dev鍵を配置する
sops-dev-setup() {
  umask 077; mkdir -p ~/.config/sops/age
  age -d -o ~/.config/sops/age/"$(_repo)".agekey "$KEYSTORE/$(_repo).dev.agekey.age"
}
# Phase 3 prodの値を編集する
sops-prod-edit() {
  SOPS_AGE_KEY="$(age -d "$KEYSTORE/$(_repo).prod.agekey.age")" sops edit secrets/prod.enc.env
}
# 2-5 prod鍵をGitHubに登録する
sops-prod-push() {
  age -d "$KEYSTORE/$(_repo).prod.agekey.age" | gh secret set SOPS_AGE_KEY --env prod
}
```

`.envrc`も次のように書けば、repo名を書き換えずに他のrepoへコピーできます。

```bash
export SOPS_AGE_KEY_FILE="$HOME/.config/sops/age/$(basename "$PWD").agekey"
watch_file secrets/dev.enc.env
dotenv <(sops decrypt secrets/dev.enc.env)
```

:::message alert
どちらも、ディレクトリ名がrepo名と一致していることが前提です。別名でcloneした場合は鍵ファイルが見つかりません。
:::

# まとめ

- チームで共有する秘密鍵は、repoごとに **dev鍵とprod鍵の2つだけ** です。APIキーが増えても配る鍵は増えません。
- 鍵はパスフレーズで暗号化してクラウドに置くので、覚えておくのはパスフレーズだけです。
- prod鍵は端末に置かず、GitHub Secretsとクラウド上の暗号化ファイルにだけ置くので、端末が侵害されても本番の値は守られます。
- direnvで`.env`の便利さはそのまま残り、暗号化ファイルが`.env.example`の役割も兼ねます。
- 代わりに、prodの編集やメンバー離脱時の鍵の入れ替えには手間がかかります。目安として4人以上になったら、AWS KMSなどへの移行を検討してください。

# 参考文献

- [getsops/sops - GitHub](https://github.com/getsops/sops)
- [SOPS Documentation](https://getsops.io/docs/)
- [FiloSottile/age - GitHub](https://github.com/FiloSottile/age)
- [direnv](https://direnv.net/)
- [gitleaks/gitleaks - GitHub](https://github.com/gitleaks/gitleaks)
- [Managing environments for deployment - GitHub Docs](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment)
