# CHANGELOG-BMS

`com.unity.renderstreaming` の BMS 改変記録。upstream の `CHANGELOG.md` とは別に、**BMS が加えた差分のみ**を記録する。

- **upstream 基準バージョン:** `3.1.0-exp.9`（本リポジトリの tag `3.1.0-exp.9` = upstream 無改変）
- **本ブランチの版:** `3.1.0-exp.9-bms.2`（`package.json` の `version`。tag は節目の目印で、参照には使わない）
- **改変ブランチ:** `bms/main`（upstream ミラーの `main` とは分離して管理する）
- **利用側:** aim4unity が UPM git 依存として `bms/main` ブランチを参照する
  （`https://github.com/boston-medical-sciences/UnityRenderStreaming.git?path=/com.unity.renderstreaming#bms/main`）。
  実際に使われるコミットは利用側の `packages-lock.json` の `hash` で固定され、`bms/main` への push だけでは利用側は変化しない。
- **対象 Unity:** 6000.3.x

## 無改変の基準実体

upstream 3.1.0-exp.9 は本リポジトリの tag `3.1.0-exp.9` である。BMS 改変差分は次のコマンドで常に再取得できる。

```bash
git diff 3.1.0-exp.9..bms/main
```

## 改変一覧（3.1.0-exp.9-bms.2）

| ファイル | 目的 | 内容 |
|---|---|---|
| `Runtime/Unity.RenderStreaming.Runtime.asmdef` | Unity 6 対応 | `com.unity.ugui@2.0.0` に対する `URS_USE_TEXTMESHPRO` の versionDefine を追加。Unity 6 では TextMesh Pro が `com.unity.ugui` へ統合され、upstream が参照する `com.unity.textmeshpro` パッケージが解決されないため、同 define が立たずコンパイルが通らない。 |
| `Editor/RenderStreamingWizard.cs` | Unity 6 対応（tarball 揃え） | `RequiredAndroidSdkVersion` を `AndroidApiLevel22` から `AndroidApiLevel25` に変更。Unity 6 では `AndroidApiLevel22` が obsolete エラー（CS0619）になりコンパイルが通らない。レジストリ公開 tarball には同修正が含まれている（tag との差分）。 |
| `package.json` | 版の識別 | `version` を `3.1.0-exp.9` から `3.1.0-exp.9-bms.2` に変更（BMS 改変版であることを判別可能にする）。 |

### 版表記変更による既知の副作用

Render Streaming Wizard の Web アプリ取得（`WebAppDownloader`）は、インストール済みパッケージの `version` をそのまま GitHub リリースの tag に使う
（`https://github.com/Unity-Technologies/UnityRenderStreaming/releases/download/{version}/webserver.exe`）。`-bms.N` 付きの版は upstream に存在しないため 404 になる。

ただしこれは版表記変更だけの問題ではない。**upstream の `3.1.0-exp.9` リリースにはサーバーバイナリが含まれておらず**（成果物は
`com.unity.renderstreaming-3.1.0-exp.9.tgz` のみ）、`WebAppDownloader` の既定値 `3.1.0-exp.8` はタグ自体が存在しない。つまり Wizard からの
取得は本版では元から成立しない。シグナリングサーバーは次のいずれかで用意する。

- バイナリを含む最後のリリース `3.1.0-exp.7` の `webserver.exe` を使う（クライアント側と 1 版ずれる）
- 本リポジトリの `WebApp/` を Node.js で動かす（本版と一致。`npm ci` → `npm run build` → `node ./build/index.js`）

## 改変の反映手順

BMS 改変は `bms/main` に commit して push するだけでよい（tag 更新は不要）。利用側（aim4unity）は
Package Manager で本パッケージを **Update** して `packages-lock.json` の `hash` を更新した時点で新しいコミットを取り込む。

## 再適用手順（upstream 更新時）

1. upstream の新しい tag を fork に取り込む（`git fetch upstream --tags` → tag を push）。
2. `bms/main` に upstream の新 tag をマージし、競合は「改変一覧」の目的に沿って解消する。
3. `package.json` の `version` の `-bms.N` 接尾辞を新しい基準バージョンに合わせて付け直す。
4. 本ファイルの基準バージョン・改変一覧を更新する。
5. （任意）節目の目印として tag `X.Y.Z-bms.N` を作成する。
6. 利用側（aim4unity）で Package Manager の Update により `packages-lock.json` の `hash` を更新し、`Docs/OSS-LICENSES/` の版表記を更新する。
