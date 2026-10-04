# EmDash 整合筆記

凍結日期： 2026-10-04
來源： https://docs.emdashcms.com/getting-started/ ， https://docs.emdashcms.com/deployment/cloudflare/ ， https://docs.emdashcms.com/existing-project/
本機實測： node v24.21.0， npm 11.19.0， astro v7.3.5， wrangler 4.147.0

## 決策

採用 blog-cloudflare 樣板。理由：已內建 D1 加 R2 綁定，Worker 入口，每分鐘 cron。
不用沙箱外掛時拿掉 sandboxRunner 與 LOADER。

## 端點

- Admin： `/_emdash/admin/`
- Media API： `/_emdash/api/media/file/...`
- 公開頁： `/` 列表，`/posts/[slug]` 內文，`/tags/[tag]` 標籤

## Env 與綁定

wrangler.jsonc 必須保留名稱：

```jsonc
{
  "name": "ai-notes",
  "main": "./src/worker.ts",
  "compatibility_date": "2026-02-24",
  "compatibility_flags": ["nodejs_compat"],
  "d1_databases": [{ "binding": "DB", "database_name": "ai-notes" }],
  "r2_buckets": [{ "binding": "MEDIA", "bucket_name": "ai-notes-media" }],
  "worker_loaders": [{ "binding": "LOADER" }],
  "triggers": { "crons": ["* * * * *"] }
}
```

astro.config.mjs：

```js
import cloudflare from "@astrojs/cloudflare";
import react from "@astrojs/react";
import emdash from "emdash/astro";
import { d1, r2, sandbox } from "@emdash-cms/cloudflare";

export default defineConfig({
  output: "server",
  adapter: cloudflare(),
  integrations: [
    react(),
    emdash({
      database: d1({ binding: "DB" }),
      storage: r2({ binding: "MEDIA" }),
      sandboxRunner: sandbox(),
    }),
  ],
});
```

src/worker.ts：

```ts
import handler, { createScheduledHandler, PluginBridge } from "@emdash-cms/cloudflare/worker";
export { PluginBridge };
export default {
  ...handler,
  scheduled: createScheduledHandler(),
} satisfies ExportedHandler;
```

線上位址： `https://ai-notes.shonen.workers.dev/`。首次部署已自動建立 D1 `ai-notes`、R2 `ai-notes-media`、KV `ai-notes-session`。
線上 setup 尚未完成，首頁顯示空站狀態。必須在正式網域跑一次 `/_emdash/admin` setup，否則 passkey 綁錯網域。
本地驗證文 `hello-ai-notes` 與草稿檢查文只存在本機 D1，未進線上庫。部署前 `dist/` 經 grep 確認無 `DRAFT-NEVER-PUBLISH`。

## Post 形狀

`title, slug, content, tags[], publishedAt, status`

狀態只有 `draft` 與 `published`。列表與標籤頁只讀 published。
