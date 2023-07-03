---
title: "參考 Ecto.CheckRepoStatus 實作 check data status"
date: 2023-06-30T21:50:23+08:00
draft: false
---

前陣子開發團隊為了要方便做 data migration，參考了[這篇文章](https://blog.appsignal.com/2020/02/25/migrating-production-data-in-elixir.html)，主要是講如何實作一個 migration 的機制，最終用起來的手感就跟 db migration 一樣方便，但每次合併 data_migration PR 後，都要在 slack 上提醒其他人本機端(local)要記得執行 data_migration，難免覺得稍嫌麻煩，覺得是不是能像是 db_migration 一樣，在執行的時候，在 server 的 log 裡面顯示訊息。

像是這樣:

```
Phoenix.Ecto.PendingMigrationError at GET /
there are pending migrations for repo: Falcon.Repo. Try running `mix ecto.migrate` in the command line to migrate it
```

後來透過 `PendingMigrationError` 找到 Phoenix 是如何實作這個過程，原來是透過 [phoenix_ecto](https://github.com/phoenixframework/phoenix_ecto) 來實現確認功能。


在 Phoenix 專案裡面的 `endpoint.ex` 會看到這個檔案，裡面有一段 code 是這樣

```elixir
  # Code reloading can be explicitly enabled under the
  # :code_reloader configuration of your endpoint.
  if code_reloading? do
    socket "/phoenix/live_reload/socket", Phoenix.LiveReloader.Socket
    plug Phoenix.LiveReloader
    plug Phoenix.CodeReloader
    plug Phoenix.Ecto.CheckRepoStatus, otp_app: :my_app
  end
```

其中 `Phoenix.Ecto.EheckRepoStatus` 就是檢查 db_migrate 的狀態，在這個 plug 你會看的 phoenix 是如何確認 db migration 的狀態

```elixir
  def call(%Conn{} = conn, opts) do
    repos = Application.get_env(opts[:otp_app], :ecto_repos, [])

    for repo <- repos, Process.whereis(repo) do
      # 檢查 pending 的 migration
      unless check_pending_migrations!(repo, opts) do
        check_storage_up!(repo)
      end
    end

    conn
  end
```

知道 Phoenix 怎麼做的以後，我們就可以按照這個邏輯這樣做

```elixir
defmodule MyApp.CheckDataStatus do
  @behaviour Plug

  alias Plug.Conn
  alias MyApp.DataMigrator
  require Logger

  def init(opts) do
    opts
  end

  def call(%Conn{} = conn, _opts) do
    DataMigrator.check_pending_migrations()

    conn
  end
end
```

然後在我們自己實作的 data_migrator 就會長這樣子

```elixir
defmodule MyApp.DataMigrator do
  def check_pending_migrations() do
    list_pending_migrations = get_pending_migrations()

    if Application.get_env(:my_app, :env) == "local" && list_pending_migrations != [] do
      Logger.warning(
        "there are pending data migrations. Please run data migrate by: `mix Myapp.data_migrate`."
      )
    end
  end

  defp get_pending_migrations() do
    already_migrated = get_already_migrated()

    Application.app_dir(:falcon, "priv/data_migrations/*")
    |> Path.wildcard()
    |> Enum.map(&get_migration_info/1)
    |> Enum.reject(fn
      # reject migrations that have already ran
      {version, _} -> Enum.member?(already_migrated, version)
      _ -> true
    end)
  end

  defp get_already_migrated() do
    from(dm in DataMigration, select: dm.version)
    |> Repo.all(data_migration: true)
  end

  defp get_migration_info(file) do
    file
    |> Path.basename()
    |> Path.rootname()
    |> Integer.parse()
    |> case do
      {integer, _} ->
        {integer, file}

      _ ->
        nil
    end
  end
end
```

接下來只要在 local 如果沒有執行到最新的 migration，就會出現提示訊息
