---
title: "Vercel Sandboxの紹介"
emoji: "🏝️"
type: "tech"
topics: ["Vercel", "Sandbox", "Nodejs", "Python"]
published: false
publication_name: "romanark"
---

# Vercel Sandbox

先日のVercel ShipでVercel Sandboxというものが公開されました。

@[card](https://vercel.com/changelog/run-untrusted-code-with-vercel-sandbox)

## Vercel Sandboxとは？
AIが生成したコードや信頼できないコードを安全に実行するためのサービスです。Fluid Computeを基盤とした、短命なMicro VMで動作し、最大で45分（[デフォルトでは5分まで](https://vercel.com/docs/vercel-sandbox/pricing#:~:text=Sandboxes%20have%20a%20maximum%20runtime%20duration%20of%2045%20minutes%2C%20with%20a%20default%20of%205%20minutes.%20You%20can%20configure%20this%20using%20the%20timeout%20option%20of%20Sandbox.create().)）まで実行できます。サンドボックスごとに最大で8つのvCPU、vCPUごとに2GBのメモリが割り当てられます。
2025-07-30時点ではベータ版のため全てのプランで使えます。

@[card](https://vercel.com/docs/vercel-sandbox/pricing)

### 使用できるランタイム
使用できるランタイムはNode.jsとPythonで、Node.jsのバージョンは22でパッケージマネージャはnpmとpnpm、pythonのバージョンは3.13でパッケージマネージャはpipとuvです。
また、sudoも[いい感じ](https://vercel.com/docs/vercel-sandbox#sudo-config)に使えるそうです。

@[card](https://vercel.com/docs/vercel-sandbox#system-specifications)

## 使ってみる
[ガイド](https://vercel.com/docs/vercel-sandbox#using-vercel-sandbox) が用意されているのでそれに従ってサンドボックスを作成します。

### 1. Vercel CLIをインストール
すでにインストールしている人は不要です。

インストール:
```sh
pnpm i -g vercel
```

### 2. 環境構築
新しいディレクトリ`sandbox-test`を作成して、パッケージをインストールします。

```sh
mkdir sandbox-test && cd sandbox-test
pnpm add @vercel/sandbox
pnpm add -D @types/node
```

### 3. プロジェクトの作成
Vercelのプロジェクトを作成してリンクします
```sh
# プロジェクトをリンク、存在しない場合は新しく作成
vercel link
# 認証トークンを`.env.local`に保存
vercel env pull
```

### 4. コード作成
以下のコードを`sandbox.js`というファイルに保存します。

```ts
import { Sandbox } from '@vercel/sandbox';

async function main() {
  const sandbox = await Sandbox.create();

  await sandbox.writeFiles([
    {
      path: "script.js",
      // Vercel Blogの方ではstreamになっていますが、contentが正しいです。
      content: Buffer.from(`
// パッケージも使えます
const chalk = require("chalk");

console.log(chalk.green("Hello from Vercel Sandbox!"));
console.log("現在の日時:", new Date().toLocaleString());
`)
    }
  ]);

  await sandbox.runCommand({
    cmd: "npm",
    args: ["install", "chalk"],
    stdout: process.stdout,
    stderr: process.stderr
  });

  await sandbox.runCommand({
    cmd: "node",
    args: ["script.js"],
    stdout: process.stdout,
    stderr: process.stderr
  });
}

main().catch(console.error);
```

### 5. 実行
以下のコマンドで実行します。
```sh
pnpm dlx tsx --env-file .env.local main.ts
```

実行すると、以下のような出力が得られます。

```
Hello from Vercel Sandbox!
現在の日時: 7/29/2025, 6:26:27 PM
```

## まとめ
Vercel Sandboxは、信頼できないコードを安全に実行するための強力なツールです。AIが生成したコードや、外部からのコードを実行する際に、セキュリティを確保しつつ、開発者が自由にコードを試すことができます。特に、AIのコード生成が進化する中で、信頼性の低いコードを扱う際のリスクを軽減するための重要な手段となるでしょう。
