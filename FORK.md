# このフォークについて(y-ytakahashi/Prowlarr)

[Prowlarr/Prowlarr](https://github.com/Prowlarr/Prowlarr) のフォーク。[enter-viewer](https://github.com/y-ytakahashi/enter-viewer)
が使う Prowlarr を、上流の安定版に追従しながら、自分のインデクサ定義や機能を足して動かすためのもの。
ライセンスは上流と同じ GPL-3.0(`LICENSE`)。上流の著作権表示・ライセンス表示は変えない。

## ブランチ

| ブランチ | 何か | 誰が書くか |
| --- | --- | --- |
| `custom` | **既定ブランチ。** 上流の安定版タグ(`vX.Y.Z.N`)の上に、自分の変更を積む。enter-viewer のイメージはこの HEAD から作る | 自分(PR か直接 push)と、`ev-sync-upstream` のマージコミット |
| `master` | 上流 `master`(安定版の枝)の写し | `ev-sync-upstream` が fast-forward するだけ。コミットしない |
| `develop` | 上流 `develop`(開発の枝)の写し。先取りしたい変更を cherry-pick する元 | 同上 |

上流は安定版を `master` に出し、タグ `vX.Y.Z.N` を打つ(GitHub Releases の「Latest」。`develop` からの pre-release も
同じ Releases に出るので、「最新の安定版」は Releases の latest = prerelease でも draft でもない最新 で見る)。

## 変更の入れ方

```bash
git clone --branch custom https://github.com/y-ytakahashi/Prowlarr.git && cd Prowlarr
git checkout -b feat/my-indexer
# … 変更(例: src/NzbDrone.Core/Indexers/Definitions/ に C# のインデクサを足す)…
git push -u origin feat/my-indexer   # → custom 向けの PR
```

- 変更は小さく分け、上流の動作を変えない(上流の更新と衝突しにくくするため)。
- YAML(Cardigann)の定義だけなら、フォークを変えずに Prowlarr の `/config/Definitions/Custom/*.yml` に置く道もある。
  C# のインデクサ・アプリ本体の変更がこのフォークの出番。
- 上流に出したい変更は、`develop` から枝を切って上流に PR を出す(上流の PR の作法は `CONTRIBUTING.md`)。

## 上流の取り込み(ev-sync-upstream)

`.github/workflows/ev-sync-upstream.yml` が 6 時間ごと(と手動 `workflow_dispatch`)に:

1. 上流の最新の安定版(Releases の latest)のタグを調べる
2. `master` / `develop` を上流に合わせる(fast-forward だけ。できなければ止めて Issue)
3. そのタグが `custom` に入っていれば終わり
4. 入っていなければ `git merge --no-ff <tag>` で `custom` に取り込み、push する

**rebase ではなく merge** にしてある: rebase は `custom` の force push が要り、自動で誰かの作業を上書きする危険がある。
merge なら履歴は残り、push は fast-forward だけで済む(このワークフローは `--force` を使わない)。
手で rebase したくなったら、自分で `custom` を rebase して `--force-with-lease` で push する(自動では行わない)。

### 衝突したとき

ワークフローは何も push せずに止まり、ラベル `ev-sync` の Issue を立てる(同じタグの Issue が開いていればコメント)。
リポジトリの secret `NTFY_URL`(例: `https://ntfy.sh/<topic>`)を置くと、同じ文を ntfy にも送る。

```bash
git fetch upstream --tags            # upstream = https://github.com/Prowlarr/Prowlarr.git
git checkout custom && git pull --ff-only
git merge --no-ff vX.Y.Z.N           # 衝突を直して commit
git push origin custom               # force は使わない
```

次の回の `ev-sync-upstream` は、そのタグが `custom` に入っていれば何もしない。

### 上流のワークフロー

上流の `.github/workflows/`(issue の自動クローズ、ラベル付け、lock)はこのフォークでは無効にしてある
(Actions の画面で disabled。ファイルは消さない: 消すと上流がそのファイルを変えたときに衝突する)。

## イメージ

enter-viewer の `prowlarr/Dockerfile` と `.github/workflows/build-prowlarr.yml` が、`custom` の HEAD を
チェックアウトして `ghcr.io/y-ytakahashi/enter-viewer-prowlarr` を作る(self-hosted runner。public なこのリポジトリには
self-hosted runner を付けない)。タグは `latest`、`<上流の版>`(例 `2.6.5.5623`)、`<上流の版>-<custom の短い SHA>`。
上流の版は「custom から辿れる `v` タグのうち、版の数字が最大のもの」
(`git tag --merged HEAD --list 'v[0-9]*' | sort -V | tail -1`)で決める。`git describe` は使わない:
上流は develop からの pre-release のタグ(例 `v2.6.5.5620`)を安定版(`v2.6.5.5623`)と同じコミットに打つことがあり、
describe がどちらを返すか決まらない。最大の数字なら、取り込み済みの最新の安定版になる(それより新しい pre-release は
develop にしか無く、custom からは辿れない)。
