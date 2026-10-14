# CHANGELOG-BMS

`com.unity.renderstreaming` の BMS 改変記録。upstream の `CHANGELOG.md` とは別に、**BMS が加えた差分のみ**を記録する。

- **upstream 基準バージョン:** `3.1.0-exp.9`（本リポジトリの tag `3.1.0-exp.9` = upstream 無改変）
- **本ブランチの版:** `3.1.0-exp.9-bms.4`（`package.json` の `version`。tag は節目の目印で、参照には使わない）
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

## 改変一覧（3.1.0-exp.9-bms.4）

| ファイル | 目的 | 内容 |
|---|---|---|
| `Runtime/Unity.RenderStreaming.Runtime.asmdef` | Unity 6 対応 | `com.unity.ugui@2.0.0` に対する `URS_USE_TEXTMESHPRO` の versionDefine を追加。Unity 6 では TextMesh Pro が `com.unity.ugui` へ統合され、upstream が参照する `com.unity.textmeshpro` パッケージが解決されないため、同 define が立たずコンパイルが通らない。 |
| `Editor/RenderStreamingWizard.cs` | Unity 6 対応（tarball 揃え） | `RequiredAndroidSdkVersion` を `AndroidApiLevel22` から `AndroidApiLevel25` に変更。Unity 6 では `AndroidApiLevel22` が obsolete エラー（CS0619）になりコンパイルが通らない。レジストリ公開 tarball には同修正が含まれている（tag との差分）。 |
| `Runtime/Scripts/SignalingManager.cs` | API 追加 | シグナリングサーバーへの接続確立時に発火する `onStart` イベントを公開（再接続時も毎回発火）。内部クラス `SignalingManagerInternal` が既に持つ同名イベントを転送するだけの追加。WebSocket が開く前に送った接続要求（`connect` メッセージ）は破棄されるため、利用側が接続確立を契機に接続登録を行うために必要。 |
| `Runtime/Scripts/InputSystem/EmulateInputFieldEvent.cs` | 入力欄操作の修正 | Backspace / Delete の入力欄キーイベントを、文字コード 0x08 / 0x7F の TextEvent から作り、同じキーの StateEvent からは作らない。ブラウザは押し続けている間の自動リピートを含む keydown ごとに TextEvent を送るが、StateEvent はキーの状態が変わったときにしか送らないため、upstream では押し続けても 1 回しか削除されない。 |
| `Runtime/Scripts/InputSystem/EmulateInputFieldEvent.cs`・`Runtime/Scripts/InputReceiver.cs` | API 追加・入力欄操作の修正 | 選択中の入力欄へ、Ctrl を含む C / X / V のキーイベントを（Shift / Alt / Meta との組み合わせも含めて）渡さない。コピー・切り取りは Ctrl のみ（CapsLock は無視）が押されているときに行い、パスワード欄以外で選択中の文字列を新設の `InputReceiver.onInputFieldCopy` イベントで利用側へ渡す。切り取りは続けて選択範囲を削除する。貼り付ける文字列はリモート側が TextEvent で送る。upstream は入力欄の標準処理へ渡すため、ホスト PC のクリップボード（`GUIUtility.systemCopyBuffer`）を読み書きする。 |
| `Runtime/Scripts/InputSystem/Receiver.cs`・`Runtime/Scripts/InputSystem/EmulateInputFieldEvent.cs`・`Runtime/Scripts/InputReceiver.cs` | API 追加 | リモートのキーボード入力を入力欄へ渡す前に判定するフィルタ `InputReceiver.SetInputFieldEventFilter(Func<GameObject, bool>)` を公開。upstream は選択中のオブジェクトが入力欄なら無条件に `ProcessEvent` で文字を渡すため、ホスト PC の前の操作者だけが操作する入力欄（配信映像に映らない UI）が選択されていると、リモートの入力がそこへ入る。フィルタが false を返したオブジェクトには渡さない。未設定（null）なら upstream と同じ挙動。フィルタは `InputReceiver` が保持し、再接続でチャネルが作り直されても引き継ぐ。 |
| `package.json` | 版の識別 | `version` を `3.1.0-exp.9` から `3.1.0-exp.9-bms.4` に変更（BMS 改変版であることを判別可能にする）。 |

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
