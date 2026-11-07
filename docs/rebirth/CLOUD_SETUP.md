# 啟用真正的帳號與雲端存檔

這份遊戲目前預設為訪客模式。登入／註冊介面已接好 Supabase REST Auth，但尚未連接任何專案；沒有假帳號，也不會把密碼存到本機。先用訪客測試遊戲即可。

1. 在你自己的 Supabase 專案啟用 Email / Password 登入，建議保留電子郵件驗證。
2. Authentication → URL Configuration 填入實際 GitHub Pages 遊戲 URL（含 `/rebirth/`）。驗證信完成後回遊戲登入。
3. SQL Editor 執行以下 SQL，建立獨立的新遊戲存檔表。舊遊戲的資料表不需要修改。

```sql
create table if not exists public.duskbound_saves (
  user_id uuid primary key references auth.users(id) on delete cascade,
  payload jsonb not null,
  updated_at timestamptz not null default now()
);
alter table public.duskbound_saves enable row level security;
create policy "Read own save" on public.duskbound_saves
  for select to authenticated using ((select auth.uid()) = user_id);
create policy "Create own save" on public.duskbound_saves
  for insert to authenticated with check ((select auth.uid()) = user_id);
create policy "Update own save" on public.duskbound_saves
  for update to authenticated using ((select auth.uid()) = user_id)
  with check ((select auth.uid()) = user_id);
grant select, insert, update on public.duskbound_saves to authenticated;
create or replace function public.duskbound_touch_save()
returns trigger language plpgsql set search_path = '' as $$
begin new.updated_at = now(); return new; end; $$;
create trigger duskbound_save_modified before update on public.duskbound_saves
for each row execute function public.duskbound_touch_save();
```

4. 在 `index.html` 搜尋 `const CLOUD_CONFIG`，填入該專案 URL 和可公開的 anon / publishable key。**禁止放 service_role 或 secret key**。
5. 把修改後檔案上傳 GitHub。建立兩個測試帳號，確認彼此看不到對方存檔；重新整理、登出、另一裝置登入都要測試。

登入後會要求選擇「載入雲端」或「上傳此裝置」，不會默默替換進度。同步有 updated_at 版本衝突檢查。訪客、本機帳號與舊遊戲使用不同儲存區。

目前沒有忘記密碼、帳號刪除、服務條款、正式防作弊或付款系統；在真實收費／商業發行前需補齊服務端驗證與平台流程。瀏覽器存檔不適合作為付費貨幣的唯一依據。
