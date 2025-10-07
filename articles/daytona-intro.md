---
title: "Daytonaの紹介"
emoji: "☀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Daytona", "Sandbox"]
published: false
---

# はじめに

最近、AIが生成したコードや信頼できないコードを安全に実行するための環境が求められています。
例えば、Code InterpreterやVibe Coding用のWebサービス（Lovable, Replit）などのケースがあります。
今回はそういったサービスを展開する上で重要なレイヤーであるサンドボックス環境を簡単に使うことができるDaytonaというサービスを紹介します。

@[card](https://daytona.run/)

## Daytonaとは？

<!-- SDKもNode.js向けとPython向けがあり、今回はNode.js向けのSDKを使用します。と書く -->

Daytonaは、AIが生成したコードや信頼できないコードを安全に実行するためのサンドボックス環境を提供するサービスです。Daytonaは使いやすさを重視しており、複雑な設定を行うことなく、すぐに利用を開始することができます。
Daytonaの機能は以下の通りです。
- TypeScriptとPythonのサポート
- Dockerイメージのカスタマイズ
- ファイルシステムの操作: 基本的な操作とパーミッションの設定、検索と置換が可能
- ネットワークアクセスの制御: 最大5つのCIDRブロックの許可が可能
- 実行時間とリソースの制限
  - vCPU, メモリ, ストレージ, 実行時間の制限が可能 
  - デフォルトは1vCPU, 1GBメモリ, 3GiBストレージ, 15分
- Git操作
  - Clone, Branch操作, Commit, Push, Pull, Status実行が可能
  - パスワード認証のみに対応なので、GitHubを使う場合はPersonal Access Tokenを利用することになる
- 任意のコマンド、コードの実行: root権限での実行やバックグラウンド実行も可能
- ターミナル操作
- ログの取得: ストリーミングと一括での取得が可能
- SSHアクセス
  - アクセス用のトークンを発行して、任意のSSHクライアントからアクセス可能
  - `ssh <token>@ssh.app.daytona.io` のようにアクセスする
  - トークンの有効期限が設定可能
- プレビューリンクの発行
  - ポートを指定して、外部からアクセス可能なプレビューリンクとトークンを発行できる
  - ブラウザからアクセスする場合は、自動で認証される
  - curlなどでアクセスする場合は、トークンを利用する
- Computer Use
  - GUIアプリケーションを実行するための仮想デスクトップ環境が利用できます
  - マウス、キーボード、スクリーンショットの操作が可能
  - DaytonaのダッシュボードからVNCアクセスも可能

## 試してみる

今回はNode.jsのサンドボックスを作成して、Honoのプロジェクトを作成して、ビルド、起動、プレビューリンクでのアクセスまでを試してみます。

Daytonaはクレジットカードを登録すると、100ドル分のクレジットがもらえます。
まずはアカウントを作成して、クレジットカードを登録しましょう。
![Daytonaのアカウント作成画面](/images/daytona-intro/signup.png)

![Daytonaのダッシュボード](/images/daytona-intro/before-receive-credit.png)

次にAPIキーを発行します。サイドバーからKeysを選択して、Create Keyをクリックします。
![DaytonaのAPIキー発行画面](/images/daytona-intro/api-keys.png)
![DaytonaのAPIキー発行ダイアログ](/images/daytona-intro/api-key-dialog.png)
![DaytonaのAPIキー発行完了](/images/daytona-intro/api-key-created.png)

次に適当なディレクトリを作成して、各種パッケージをインストールします。
package.json
```json
{
  "type": "module",
  "dependencies": {
    "@daytonaio/sdk": "^0.108.0"
  },
  "devDependencies": {
    "@types/node": "^24.7.0",
    "tsx": "^4.20.6"
  }
}
```
コマンド
```bash
npm install
```

次にindex.tsを作成します。以下のコードをコピペしてください。
```typescript
import { Daytona } from '@daytonaio/sdk';

// Daytona クライアントを初期化
const daytona = new Daytona({ apiKey: 'ここに先ほどのAPIキーを入力してください' });

// サンドボックスのインスタンスを作成
const sandbox = await daytona.create({
    language: 'typescript',
});

// コマンドを実行
console.log("npm create を実行中...");
const response = await sandbox.process.executeCommand("npm create hono@latest ./my-app -- --template nodejs --install --pm npm");
console.log(response.result);

console.log("ビルドを実行中...");
const response2 = await sandbox.process.executeCommand("npm run build", "my-app");
console.log(response2.result);

// 対話型セッションを開始
const ptyHandle = await sandbox.process.createPty({
    id: 'interactive-session',
    cols: 300,
    rows: 100,
    cwd: 'my-app',
    onData: (data) => {
        const text = new TextDecoder().decode(data);
        process.stdout.write(text);
    },
});

await ptyHandle.waitForConnection();
await ptyHandle.sendInput('npm run start\n');

// プレビューリンクを発行
const previewInfo = await sandbox.getPreviewLink(3000);
console.log(`プレビューリンク: ${previewInfo.url}`);
console.log(`プレビュートークン: ${previewInfo.token}`);

// ユーザーからの入力を受け付ける
console.log("対話型セッションを開始しました。Ctrl+C exit で終了します。");
process.stdin.setRawMode?.(true);
process.stdin.resume();
process.stdin.on('data', async (data) => {
    await ptyHandle.sendInput(data);
});

// セッションの終了を待機
const result = await ptyHandle.wait();
process.stdin.setRawMode?.(false);
process.stdin.pause();

console.log(`Session completed with exit code: ${result.exitCode}`);

// 後片付け
await sandbox.delete();
```