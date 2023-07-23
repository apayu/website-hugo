---
title: "How to check the data status during data migration?"
date: 2023-06-30T21:50:23+08:00
draft: false
---

A while ago, the development team was working on facilitation data migration. We have referred to [Migrating Production Data in Elixir](https://blog.appsignal.com/2020/02/25/migrating-production-data-in-elixir.html), which primarily discusses how to implement a data migration mechanism. The final user experience is as convenient as a db migration. After each data_migration PR is merged. A reminder should be posted on Slack to prompt others to run the data_migration locallly. It is somewhat cumbersome, and I wonder if it could be possible display a message in the server log when running, similar to db_migration.

Something like this:

```
Phoenix.Ecto.PendingMigrationError at GET /
there are pending migrations for repo: Falcon.Repo. Try running `mix ecto.migrate` in the command line to migrate it
```

Later, by utilizing `PendingMigrationError`. I discovered how Phoenix implements this process. It turns out that the confirmation functionality is achieved through [phoenix_ecto](https://github.com/phoenixframework/phoenix_ecto).

In the Phoenix project, you will find the `endpoint.ex` file where you will come across a code snippet like this:

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

The `Phoenix.Ecto.CheckRepoStatus` is responsible for checking the status of db_migrate. In this plug, you will see how Phoenix confirms the status of db migration.

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

Now that we know how Phoenix dose it. We can implement it following the same logic.

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

Then, in our self-implemented data_migrator. It will look something like this:


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

Next, if the local environment hasn't executed the latest migration, a prompt message will appear.

Cool!
