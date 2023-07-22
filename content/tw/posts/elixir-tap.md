---
title: "Elixir Kernel.tap"
date: 2023-07-07T22:02:53+08:00
draft: false
---
Elixir 從 1.12.0 開始，新增了一個function 叫 `Kernel.tap/2`，官方的範例是這樣使用

```
iex> tap(1, fn x -> x + 1 end)
1
```

可以看到 `1` pipes 進 tap 這個 function 裡面以後，不管 function 裡面做了什麼事情，他都會回傳 `1`，也就是當初傳入 tap 時的參數

這個有什麼好處呢？我們有時候在 pipes 的過程中，會需要做一些像是`副作用`的效果，但又想要保持傳出去的參數不變，所以有時候會不小心就這樣寫

```elixir
def do_something_b(val) do
  do_other_thing(val)
  # ...
  val
end

def do_thing() do
  val
  |> do_something_a()
  |> do_something_b()
  |> do_something_c()
end
```

但其實我們可以用 `Kernal.tap()` 來做到一樣的事情，就如同文章一開始的範例一樣

```elixir
def do_something_b(val) do
  do_other_thing(val)
end

def do_thing() do
  val
  |> do_something_a()
  |> tap(&do_something_b/1)
  |> do_something_c()
end
```

看一下 Elixir 原始碼是怎麼做到這件事情

```elixir
@doc since: "1.12.0"
defmacro tap(value, fun) do
  quote bind_quoted: [fun: fun, value: value] do
      _ = fun.(value)
      value
  end
end
```

其實也很單純，就是幫你把傳進來的參數丟給你傳進來的 function 後，再丟出去

疑？！等等，這不是跟原本的 code 很像嗎？其實 Elixir 也只是幫你做掉這件事情，就是這麼簡單暴力😂
