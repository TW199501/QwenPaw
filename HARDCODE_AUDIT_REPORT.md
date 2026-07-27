# QwenPaw 硬編碼審計報告

日期：2026-07-25　分支：`feature/zh-tw-i18n`　範圍：`src/`（Python 核心）、`console/src/`、`website/src/`

## 方法

按類別分別掃描（排除 `*.test.*` / `*.spec.*` / `node_modules` / `skills`）：

| 類別 | 工具/Pattern |
|---|---|
| Secrets/Token/密碼 | `rg -i -P "(api[_-]?key\|secret\|token\|password)\s*[:=]\s*['\"][A-Za-z0-9_-]{8,}['\"]"` |
| URL/Host/Port | `rg -P "https?://\|localhost:\d+\|127\.0\.0\.1\|0\.0\.0\.0"` |
| 使用者專屬絕對路徑 | `rg -F "/home/" / "/Users/" / "C:\\Users\\"` |
| env/prod 判斷 | Python 腳本比對 `NODE_ENV/ENV === "prod\|dev\|..."` |
| Magic number（timeout/port/limit） | Python 腳本比對 `(port\|timeout\|retries\|limit)\s*[:=]\s*\d+` |
| 前端硬編碼中文文案（未走 i18n） | Python 腳本掃 `.tsx/.ts` 含 CJK 字元的行 |

## 結論摘要

| 類別 | 發現數 | 高風險 | 中風險 | 低風險/合理 |
|---|---|---|---|---|
| Secrets/Token | 少量 | 0 | 0 | 全部 |
| URL/Host/Port | ~30 | 0 | 3 | 其餘 |
| 用戶專屬路徑 | 0 | - | - | - |
| env/prod 判斷 | 0 | - | - | - |
| Magic number (timeout等) | 93 | 0 | 1 | 其餘 |
| 前端硬編碼文案 | ~40 條真陽性 | 0 | 5 | 其餘為已走 i18n 或註解/URL片段 |

**總體結論：本 repo 沒有發現真正的 secrets/憑證外洩、也沒有用戶專屬硬編碼路徑；主要問題集中在（1）console 前端仍有約 40 處中文文案未走 i18next `t()`、以及（2）少數服務綁定位址 `0.0.0.0` 需要留意安全風險。**

---

## 1. Secrets / Token / 密碼（風險：High，若命中）

掃描結果：命中皆為測試檔案中的假值（`tests/unit`、`tests/contract`、`tests/integration` 內的 `test_token`、`test_secret`、`xoxb-test-...` 等），以及：

- `src/qwenpaw/app/routers/tools.py:23` — `PASSWORD = "password"`：這是一個 enum 成員名稱定義（欄位類型標籤），非真實密碼值。**合理，非硬編碼洩漏。**

**結論：未發現硬編碼的真實 API key / secret / password。**

---

## 2. URL / Host / Port（風險：視項目而定）

### 需留意（Medium）

| 位置 | 內容 | 說明 |
|---|---|---|
| `src/qwenpaw/app/channels/onebot/channel.py`（多處） | `bind="0.0.0.0"` | 監聽所有網路介面的預設值。若無法透過設定/環境變數關閉，暴露面較大，建議確認可配置且預設安全（如僅 loopback）。 |
| `src/qwenpaw/app/channels/sip/mini_registrar.py:10`, `sip/__init__.py:157,777,785,815` | `bind="127.0.0.1", port=5060` | SIP 服務固定 port 5060（標準 SIP port，屬合理慣例），bind 是 loopback，風險低，但 port 建議可配置以應對多實例場景。 |
| `src/qwenpaw/tauri/entry.py:271` | `host = "127.0.0.1"` | 待確認是否可被環境變數/設定覆蓋；桌面應用場景通常合理。 |

### 合理（Low / 不需修正）

- `src/qwenpaw/providers/provider_manager.py:1259` — `base_url="http://localhost:1234/v1"`：LM Studio 預設連線位址，屬慣例預設值。
- `src/qwenpaw/providers/provider_manager.py:2393`、`src/qwenpaw/app/routers/local_models.py:312` — `f"http://127.0.0.1:{port}/v1"`：動態帶入 port 變數，非固定硬編碼。
- `src/qwenpaw/providers/ollama_provider.py:37` — 有 `OLLAMA_HOST` 環境變數 fallback 機制，硬編碼僅作預設值。
- `src/qwenpaw/app/routers/fork.py:28` — `_LOCALHOST_ADDRS = {"127.0.0.1", "::1", "localhost"}`：安全用途的白名單常數，合理。
- `src/qwenpaw/cli/chats_cmd.py`, `channels_cmd.py` 等 CLI help text 範例 URL：合理。
- website/console 內文檔範例（`docker run -p 127.0.0.1:8088:8088 ...`）、外部連結（GitHub/Discord/CDN/ModelScope）：合理，皆為展示用途或第三方固定端點。

**建議：** 對 `onebot/channel.py` 的 `0.0.0.0` bind 逐一確認是否有對應設定項可覆寫，若無建議補上（低成本、安全收益高）。

---

## 3. 使用者專屬絕對路徑

掃描 `/home/xxx/`、`/Users/xxx/`、`C:\Users\xxx\` pattern：**無命中**（`src/qwenpaw/security/tool_guard/safety_checks.py:379-381` 僅為註解中的範例路徑說明文字，非真實硬編碼路徑）。

**結論：未發現硬編碼的用戶專屬檔案路徑。**

---

## 4. env/prod 判斷式硬編碼

掃描 `NODE_ENV`/`ENV` 與 `prod`/`production`/`dev`/`development` 的字串比較：**無命中**。前後端均未見寫死的環境分支判斷邏輯。

---

## 5. Magic Number（timeout / port / retries / limit）

93 處命中，型態集中在 `timeout=<秒數>`（httpx/asyncio/subprocess 等呼叫），例如：

```
src/qwenpaw/agents/tools/browser_control.py:3565: timeout=600,  # 10 minutes max
src/qwenpaw/app/channels/qrcode_auth_handler.py:*: httpx.AsyncClient(timeout=10/15)
src/qwenpaw/cli/plugin_commands.py:*: urllib.request.urlopen(req, timeout=120/30)
console/src/api/modules/agents.ts:44: timeout: 10 * 60 * 1000
```

**評估：這類 timeout 數值屬於函式呼叫的局部參數，是常見且合理的寫法，不算「不良硬編碼」**（不像 secrets/host 那樣需要按環境替換）。唯一建議留意的：

- `src/qwenpaw/app/channels/sip/__init__.py:777,785,815` — `port = 5060` 出現 3 次重複字面量，建議抽成模組常數避免未來改動遺漏一處。
- `src/qwenpaw/app/channels/telegram/channel.py:873,1246` — `50 MB` Telegram 檔案大小上限重複出現兩處，建議抽常數集中維護（雖為 Telegram 官方固定值，重複字面量仍有維護風險）。

其餘 timeout 數值分散在各 channel/provider 呼叫點，屬合理的局部配置，暫不建議大動作重構（YAGNI，除非之後有需求要讓 timeout 可配置）。

---

## 6. 前端硬編碼中文文案（未走 i18next）

掃描 `console/src/**/*.tsx,ts` 與 `website/src/**/*.tsx,ts` 中含中文字元、且非透過 `t("key", "...")` 的行，扣除註解、正則表達式、URL fragment、已走 `t()` 的行後，**真陽性約 40 處**，集中在：

### console/ — 需要補 i18n 的檔案

| 檔案 | 說明 |
|---|---|
| `console/src/layouts/constants.ts:82-112` | 「如何更新 QwenPaw」整段中文說明文字直接寫死（有對應英文版走 i18n 邏輯分支，但這段是完整內嵌 markdown，非 t() key） |
| `console/src/pages/Control/CronJobs/components/JobDrawer.tsx:437,637,659,683` | 表單校驗訊息 `"请选择至少一天"`、placeholder `"输入自定义值后按 Enter"` 未走 i18n |
| `console/src/pages/Control/CronJobs/components/columns.tsx:156,161` | `"Cron 表达式："`、`"格式：分钟 小时 日 月 星期"` 寫死中文標籤 |
| `console/src/pages/Control/CronJobs/components/templates.ts:192-315` | Cron 範本的 `textContent`/`agentPrompt` 提示文字大量寫死中文（屬「範本內容」，是否需要 i18n 需與產品確認，若要多語系需重構為按語言切換範本集） |
| `console/src/pages/Inbox/hooks/useInboxData.ts:30-38` | `"Heartbeat 执行成功"` 等狀態文案硬編碼中文，無對應英文分支 |
| `console/src/pages/Inbox/components/PushMessageCard.tsx:50`, `Inbox/utils/traceUtils.ts:206` | 正則裡硬編碼中文關鍵字 `定时任务结果\|心跳结果`（用於解析既有中文輸出，屬另一種耦合風險：若上游文案改動或英文環境會失效） |
| `console/src/pages/Settings/Models/.../RemoteModelManageModal.tsx:252` | `"模型上下文窗口大小，控制上下文压缩阈值（≥1000）"` 未走 t()（同檔其餘字串多數已走 t()，此為漏網之魚） |
| `console/src/pages/Agent/ACP/components/ACPDrawer.tsx:43` | `zh: "如何配置外部-runner"` — 手動 zh/en 對照物件而非 i18next key（架構性選擇，非 bug，但無法擴充到 zh-TW） |
| `console/src/pages/Agent/Config/components/ReactAgentCard.tsx:16` | `{ value: "zh", label: "中文" }` 下拉選單標籤寫死，未隨介面語言切換 |
| `console/src/pages/Settings/PluginManager/components/MarketPluginList.tsx:28-34` | 分類標籤用 `{ zh: "...", en: "..." }` 物件而非 i18next key（架構性，同上） |
| `console/src/pages/Settings/SkillPool/components/ImportBuiltinModal.tsx:163`, `PoolSkillDrawer.tsx:127` | `"中文"` 選項文字寫死 |
| `console/src/layouts/SidebarSettingsPanel.tsx:35-36` | 語言清單 `{ key: "zh", label: "简体中文" }` 等 — **與 `LanguageSwitcher/index.tsx` 重複維護一份語言清單**，且未包含新加入的 `zh-TW`（見下方建議） |
| `console/src/pages/Control/Channels/components/ChannelDrawer.tsx:69-87` | 文檔連結 hash 帶中文錨點（`?lang=zh#钉钉推荐`），是連結而非 UI 文案，風險低但建議確認錨點跟文檔實際 heading 同步 |

### website/ — 屬既定架構模式（非 bug）

`website/src/config.ts`、`Ecosystem.tsx`、`data/testimonials.ts`、`Downloads/index.tsx`、`Nav.tsx` 等大量使用 `xxxZh` / `xxxEn` 成對欄位 + `isZh` 三元判斷，**這是網站既有的雙語內容架構**（非 i18next key 驅動），不是意外的硬編碼疏漏。

⚠️ **與本次新增 zh-TW 相關的風險**：這批 `isZh ? zhText : enText` 寫法是**二元判斷**，zh-TW 語言啟用後會落到 `enText`（因為 `isZh` 通常只判斷 `lang === "zh"`）。之前 `website/src/i18n/SiteLanguageContext.tsx` 的三語循環已支援 zh-TW 的介面框架文案（走 i18next resources），但這些**內容型**（testimonials、生態圖標籤、下載頁 hash 錨點等）欄位是否也要補 `xxxZhTW` 或改用 OpenCC 動態轉換，需要產品決策，不在本次審計自動修正範圍內，僅提出風險提示。

---

## 修正建議優先序

1. **P1（架構一致性）**：`console/src` 中 `LanguageSwitcher/index.tsx` 與 `SidebarSettingsPanel.tsx` 各自維護一份語言清單，`SidebarSettingsPanel.tsx` 目前缺 zh-TW，建議合併成單一 source of truth，避免未來新增語言時漏改。
2. **P1（i18n 補齊）**：`JobDrawer.tsx` 表單校驗訊息、`columns.tsx` cron 標籤、`useInboxData.ts` heartbeat 狀態文案，這幾處是**使用者可見的功能性文案**且明顯遺漏 t()，建議優先補上。
3. **P2**：`RemoteModelManageModal.tsx:252` 補上遺漏的 `t()` 包裹（同檔其他行已是此模式，僅此一行漏改，改動成本低）。
4. **P2**：`sip/__init__.py` 的 `port = 5060` 與 `telegram/channel.py` 的 `50 MB` 重複字面量抽成模組常數。
5. **P3（需產品決策，非本次審計範圍）**：website 的 `xxxZh/xxxEn` 內容型雙語欄位是否要擴充支援 zh-TW；`onebot/channel.py` 的 `0.0.0.0` 預設 bind 是否需要更保守的預設值。
6. **P3**：Cron 範本（`templates.ts`）文案是否要多語系化，需先確認產品需求（是否要支援英文使用者用範本）。

---

## 已知誤報 / 排除項目

- 測試檔案（`*.test.*`、`*.spec.*`、`tests/`）內的假 token/secret/密碼字串。
- 註解、docstring、正則表達式字面量、markdown 文檔中的範例路徑或 URL 片段。
- README/CLI help text 中的展示性 URL、port 範例。
- 已透過 `t("key", "中文預設值")` 走 i18next、僅預設值含中文的正常用法。
- `website/src` 的 `xxxZh/xxxEn` 雙語欄位（既定架構，非疏漏，另見 P3 建議）。
