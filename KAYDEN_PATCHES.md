# kayden-tools 的修改

這個 repo 是 [punitarani/fli](https://github.com/punitarani/fli) 的鏡像，供 [kayden-tools](https://github.com/KaydenTsai/kayden-tools) 的機票比價功能使用。原作者的程式與授權（`LICENSE.txt`，MIT）維持不變。

| 分支 | 內容 |
|---|---|
| `main` | 與原作者一致，不放任何修改 |
| `kayden-tools` | 原作者某個 commit + 一個修改 commit（本檔說明的內容） |

kayden-tools 的 Dockerfile（`backend/src/Dockerfile` 的 `FLI_SOURCE_URL`）指向 `kayden-tools` 分支上的修改 commit。

## 修改了什麼

讓 `fli dates` 每天除了最低價，也回傳那個班次的總飛行時間與轉機次數，並支援最長飛行時間篩選。只新增欄位與選項，不改既有輸出。

| 檔案 | 函式／位置 | 修改 |
|---|---|---|
| `fli/search/dates.py` | `DatePrice` | 新增選填欄位 `flight_duration_minutes`、`stops` |
| `fli/search/dates.py` | `SearchDates._price_one_date` | 原本只取 `min(prices)`；改成挑出最便宜的班次物件（同價取時長較短者），一併填入時長與轉機次數。來回票查詢不填這兩欄 |
| `fli/cli/utils.py` | `serialize_date_result` | 欄位有值時，JSON 輸出 `flight_duration_minutes`、`stops` |
| `fli/cli/commands/dates.py` | `dates` 指令 | 新增 `--max-duration <分鐘>`，傳進 `DateSearchFilters.max_duration`，並寫進 JSON 的 `query`（成功與錯誤回應都有） |
| `tests/search/test_date_sweep.py` | `TestCheapestItineraryDetails` | 挑選邏輯、同價取短、最長時間篩選、來回票不填 |
| `tests/cli/test_dates.py` | 檔尾兩個測試 | `--max-duration` 傳遞與 JSON 欄位 |

每個修改點都有 `kayden-tools patch` 註解，用 `grep -rn "kayden-tools patch" fli tests` 可以全部找出來。

設計理由：

- 過濾邏輯本來就存在：`fli/search/_tfs.py` 的 `apply_client_side_filters` 已經會用 `filters.max_duration` 排除超時的班次，缺的只是 CLI 選項。
- JSON 欄位刻意不叫 `duration`：`fli dates --duration` 已經代表來回票的行程天數。
- 來回票的一列價格涵蓋去回兩段，用一個時長描述會誤導，所以不填。

## 同步原作者的更新

```bash
git fetch upstream
git switch main && git merge --ff-only upstream/main && git push origin main
git switch kayden-tools && git rebase main
```

rebase 有衝突時，最可能發生在 `_price_one_date` 的結尾（原作者常調整這段的錯誤處理）。依上表重新套用：保留原作者的新邏輯，只把「取最低價」改成「取最便宜的班次物件」。

## 驗證

```bash
uv run --extra dev pytest -q
uv run --extra dev ruff check fli tests
fli dates TPE BKK --from <今天+30> --to <今天+36> --currency TWD --max-duration 240 --format json
```

最後一個指令的每一天都應該有 `flight_duration_minutes`，而且都不超過 240。

## 更新 kayden-tools

1. 把 `kayden-tools` 分支推上 GitHub：`git push --force-with-lease origin kayden-tools`（rebase 後需要 force）。
2. 在 kayden-tools repo 把 `backend/src/Dockerfile` 的 `FLI_SOURCE_URL` 改成新的修改 commit。
3. 在這個 repo 重新匯出相依清單，覆蓋 kayden-tools 的 `backend/src/fli/requirements.txt`（保留檔頭說明）：
   ```bash
   uv export --frozen --no-dev --no-emit-project -o <kayden-tools>/backend/src/fli/requirements.txt
   ```
4. 部署後，打開 kayden-tools 的 `/dev/fli-probe` 做冒煙測試（見 kayden-tools 的 `backend/src/KaydenTools.Api/DevTools/README.md`）。
