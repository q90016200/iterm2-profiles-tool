# iterm2-profiles

備份、匯出、匯入 iTerm2 profile 的小工具。純 Python3 標準函式庫實作，不需安裝任何額外套件。

處理的是 **Preferences plist**（`com.googlecode.iterm2` domain 底下的 `New Bookmarks`），透過 `defaults export`/`defaults import` 讀寫，而不是 Dynamic Profiles。每個 profile 若有 `Initial Text` 指向 `~/auto_login_ssh/` 底下的自動登入腳本（例如 `~/auto_login_ssh/gf1prod/xxx.sh`），備份/匯出/匯入時會連腳本一起處理。

> ⚠️ **這些登入腳本裡通常明碼寫著 SSH 密碼**。`backup`/`export` 產生的資料夾請不要上傳雲端、分享或加進版本控制 —— 專案已用 `.gitignore` 排除 `backup/` 與 `iterm2-export-*/`，但你自訂輸出路徑時要自己注意。

## 安裝

```bash
chmod +x iterm2-profiles
```

## 使用方式

無參數直接執行會進入互動選單：

```bash
./iterm2-profiles
```

```
1) 備份
2) 列出 profiles
3) 匯出
4) 匯入
5) 離開
```

選「4) 匯入」時會從 `backup/` 資料夾開始，用數字選單瀏覽子資料夾（輸入編號進入、`..` 回上一層、或直接貼路徑跳過去），找到含 `profiles.json` 的資料夾後選 `0) 使用這個資料夾` 即可，不用手動打完整路徑。

也可以直接下指令：

```bash
./iterm2-profiles backup                        # 備份全部 profile + 對應腳本到 backup/<時間戳記>/
./iterm2-profiles list                           # 列出所有 profile，標出缺腳本／重複名稱
./iterm2-profiles export gf_alfa2 gf_xj_api -o out_dir   # 匯出指定名稱的 profile + 對應腳本
./iterm2-profiles import out_dir                 # 匯入，遇到差異會顯示並詢問是否覆蓋
```

## 換機器 / 還原

`backup` 跟 `export` 產生的是同一種 bundle 格式（`profiles.json` + `scripts/<相對路徑>`）。換一台機器時：

1. 原機器：`./iterm2-profiles backup`
2. 把 `backup/<時間戳>/` 整個資料夾複製過去（USB / AirDrop / 私有 git repo，**不要用一般雲端同步**）
3. 新機器：`./iterm2-profiles import backup/<時間戳>/`

## 腳本不在追蹤的腳本目錄裡也沒關係

`Initial Text` 指到的登入腳本只要**檔案存在，不管放在哪個資料夾**都會被 `backup`/`export` 收進 bundle（放在 `scripts/_other/<原始路徑攤平>`），`import` 時會還原回原本記錄的絕對路徑。`list` 對這種情況會標「ℹ 腳本不在追蹤資料夾內，仍會一併備份」；只有腳本真的找不到時才會標「⚠ 找不到登入腳本」。

`--script-dir` 的路徑改寫規則**只適用於在追蹤資料夾內的腳本**——不在裡面的腳本沒有「換一個新機器要改寫成哪個目錄」的問題，永遠原路徑還原。

## 匯入的規則

- 依 **Name** 比對：找到同名 profile 時，會同時比對 profile 欄位差異與腳本內容差異，兩者只要有一邊不同就印出來、詢問是否覆蓋。
- 覆蓋時**保留舊的 Guid**，避免其他設定（如快捷鍵綁定）因 Guid 改變而失效。
- 找不到同名 profile 時，直接新增（配一個新的大寫格式 Guid）。
- **只會新增/覆蓋，不會刪除**：本機既有、但匯入檔案裡沒有的 profile 完全不受影響。
- 若用 `--script-dir` 指定跟來源不同的腳本目錄，該 profile 的 `Initial Text` 會自動改寫成新路徑；預設情況（都用 `~/auto_login_ssh`）完全不用改寫，因為 `~` 在任何帳號下都會展開成當下使用者的 home 目錄。
- 匯入前若偵測到 iTerm2 還在執行，會提醒先完全結束（Cmd+Q）再繼續，因為 `defaults import` 是整個 domain 覆蓋，避免蓋掉你剛剛在 iTerm2 裡動過的其他設定。

## 自我檢查

```bash
./iterm2-profiles --selftest
```

過程中完全不會呼叫真正的 `defaults` 指令，不會動到你機器上實際的 profile。
