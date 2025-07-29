---
title: "Vercel Sandboxの紹介"
emoji: "🏝️"
type: "tech"
topics: ["Vercel", "Sandbox", "Nodejs", "Python"]
published: false
publication_name: "romanark"
---

# Vercel Sandbox

先日の Vercel Ship で Vercel Sandbox という面白いサービスが公開されました！

@[card](https://vercel.com/changelog/run-untrusted-code-with-vercel-sandbox)

## Vercel Sandbox とは？

AI が生成したコードや信頼できないコードを安全に実行できるサービスです。Fluid Compute を基盤とした短命な Micro VM で動作し、最大で 45 分（[デフォルトでは 5 分まで](<https://vercel.com/docs/vercel-sandbox/pricing#:~:text=Sandboxes%20have%20a%20maximum%20runtime%20duration%20of%2045%20minutes%2C%20with%20a%20default%20of%205%20minutes.%20You%20can%20configure%20this%20using%20the%20timeout%20option%20of%20Sandbox.create().>)）まで実行可能です。サンドボックスごとに最大で 8 つの vCPU、vCPU ごとに 2GB のメモリが割り当てられます。
2025-07-30 時点ではベータ版なので、すべてのプランで無料で使えちゃいます！気前がいいですね。

@[card](https://vercel.com/docs/vercel-sandbox/pricing)

### 使用できるランタイム

使用できるランタイムは Node.js と Python です。Node.js はバージョン 22 で、パッケージマネージャは npm と pnpm に対応しています。python はバージョン 3.13 で、パッケージマネージャは pip と uv が使えます。
また、sudo も[いい感じ](https://vercel.com/docs/vercel-sandbox#sudo-config)に使えるそうです。

@[card](https://vercel.com/docs/vercel-sandbox#system-specifications)

## 実際に使ってみる

ではサンドボックスを作ってみましょう！

### 1. Vercel CLI をインストール

すでにインストールしている人はスキップしてください。

インストール:

```sh
pnpm i -g vercel
```

### 2. 環境構築

新しいディレクトリ`sandbox-test`を作成して、必要なパッケージをインストールします。

```sh
mkdir sandbox-test && cd sandbox-test
pnpm add @vercel/sandbox
pnpm add -D @types/node
```

### 3. プロジェクトの作成

Vercel のプロジェクトを作成してリンクします。

```sh
# プロジェクトをリンク、存在しない場合は新しく作成
vercel link
# 認証トークンを`.env.local`に保存
vercel env pull
```

### 4. コード作成

以下のコードを`sandbox.js`というファイルに保存してください。

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
import chalk from "chalk";

console.log(chalk.green("Hello from Vercel Sandbox!"));
console.log("現在の日時:", new Date().toLocaleString());
`)
    },
    {
      path: "package.json",
      content: Buffer.from(JSON.stringify({
        type: "module",
        dependencies: {
          chalk: "^5.4.1"
        }
      }))
    }
  ]);

  await sandbox.runCommand({
    cmd: "npm",
    args: ["install"],
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

### 5. 実行してみる

以下のコマンドで実行します。TypeScript を使っているので、`tsx`を使っています。

```sh
pnpm dlx tsx --env-file .env.local main.ts
```

実行すると、以下のような出力が得られるはずです！

```
Hello from Vercel Sandbox!
現在の日時: 7/29/2025, 6:26:27 PM
```

## まとめ

Vercel Sandbox は、信頼できないコードを安全に実行するための非常に便利なツールです。
AI が生成したコードや、外部からのコードを実行する際に、セキュリティを確保しながら簡単にコードを実行できるので良さそうですね。

## 参考リンク
- https://vercel.com/docs/vercel-sandbox
- https://vercel.com/docs/vercel-sandbox#using-vercel-sandbox
- https://vercel.com/docs/vercel-sandbox/reference/readme
