---
title_image_path: mixing_tools.jpg
created_at: 2018-03-05T23:57:23
tags:
  - elixir
  - introduction
  - mix
  - tool
  - dependencies
  - http client
  - json
  - functional programming
---

# Инструментът Mix
Заедно с `elixir`, `elixirc` и `iex` при инсталацията на Elixir получаваме и `mix`. Това е инструмент, който автоматизира и улеснява работата ни за:
- създаване на приложение/библиотека
- компилиране
- тестване
- управление на dependencies
- форматиране на кода
- изпълнение на създадени от нас задачи (tasks)

#### Създаване на проект
Освен в редки случаи, винаги ще създаваме нашите проекти/библиотеки с mix:
```
$ mix new github_client
```
Резултатът от изпълнението на тази команда е:
```
* creating README.md
* creating .formatter.exs
* creating .gitignore
* creating mix.exs
* creating config
* creating config/config.exs
* creating lib
* creating lib/github_client.ex
* creating test
* creating test/test_helper.exs
* creating test/github_client_test.exs

Your Mix project was created successfully.
You can use "mix" to compile it, test it, and more:

    cd github_client
    mix test

Run "mix help" for more commands.
```

`mix` създава основната структура, която изглежда така:
- `.formatter.exs` - използва се от `mix format` за да разбере кои файлове трябва да форматира. Поддържа ограничен набор от възможни настройки, най-съществените от които са дължина на реда и избор на кои функции да не бъдат поставяне скоби около аргументите.
- `mix.exs` съдържа конфигурацията на проекта. В него се определят името, версията зависимосттите на проекта и др.
- `config` папката съдържа конфигурации, използвани от кода на проекта. Пример за такава конфигурация е:
```elixir
config :github_client, GithubClient.Store,
  host: {:system, "INFLUXDB_HOST", "localhost"},
  port: {:system, "INFLUXDB_PORT", 8086},
  pool: [max_overflow: 10, size: 20],
 ```
 - `lib` папката съдържа кода на проекта
 - `test` папката съдържа тестовете на проекта

#### Тестване

При създаването на проекта видяхме, че `mix` ни предлага да влезем в папката и да изпълним тестовете с `mix test`. Нека преди това добавим в `test/github_client_test.exs` още един тест:
```elixir
test "two lists are equal" do
  assert [1,2,3,4,5] == [1,2,3,4,5,6,7]
end
```

И изпълняваме `mix test`. Забелявзваме, че освен кода, генерирал грешката, се показва и разликата между лявата и дясната страна, оцветени в червено или зелено.
![mix test output](https://raw.githubusercontent.com/IvanIvanoff/blog/master/assets/mix_test_output.png)


#### Dependencies
По подразбиране  външните зависимости (dependencies) се инсталират чрез [Hex package manager](https://hex.pm/). Можем вместо това да подадем път към git хранилище  или път към папка на вашия компютър. При първото инсталиране на  пакети, mix автоматично ще инсталира и Hex.

За нашите нужди ще ни трябва библиотека за JSON и библиотека, предоставяща HTTP клиент. Това са [Poison](https://hex.pm/packages/poison) и [HTTPoison](https://hex.pm/packages/httpoison). Всички пакети се намират на [hex.pm](https://hex.pm/). Добавяме ги към `mix.exs` функцията `deps`  и тя вече изглежда така:
```elixir
  defp deps do
    [
      {:poison, "~> 3.1"},
      {:httpoison, "~> 1.0"},
    ]
  end
  ```
Сега трябва само да изпълним `mix deps.get`. 

Tук e моментът да споменем и как mix се справя с различните версии на един пакет. За всеки пакет съществува единствена версия (игнорираме какво се случва при hot code swapping). Това означава, че всички, които зависят от даден пакет, трябва да се съгласят за точно една определена версия. 

Чрез `~>` се задава диапазон от позволени версии на дадения пакет. Tой работи като фиксира всички цифри от версията, без последната, която евентуално може да бъде по-висока. Нека да видим какво се случва като добавим два HTTP клиента, които вътрешно зависят от една библиотека (hackney):
```
{:tesla, "~> 0.10"},
{:httpoison, "~> 1.0"},
```

В `mix.lock` намираме следните два реда:
```
"httpoison": {:hex, :httpoison, "1.0.0", "1f02f827148d945d40b24f0b0a89afe40bfe037171a6cf70f2486976d86921cd", [:mix], [{:hackney, "~> 1.8", [hex: :hackney, repo: "hexpm", optional: false
"tesla": {:hex, :tesla, "0.10.0", "e588c7e7f1c0866c81eeed5c38f02a4a94d6309eede336c1e6ca08b0a95abd3f", [:mix], [{:exjsx, ">= 0.1.0", [hex: :exjsx, repo: "hexpm", optional: true]}, {:fuse
"hackney": {:hex, :hackney, "1.10.1", "c38d0ca52ea80254936a32c45bb7eb414e7a96a521b4ce76d00a69753b157f21", [:rebar3], [{:certifi, "2.0.0", [hex: :certifi, repo: "hexpm", optional: false]
```
Виждаме, че `httpoison` има изискване `:hackney, "~> 1.8"`, а `tesla` има изискване `:hackney, "~> 1.6"`.
Тъй като последната цифра във версията може да бъде по-голяма, то инсталираната версия на `hackney` е `1.10.1` и тя удовлетворява изискванията и на двата пакета.

Веднъж щом се случи това определяне на дадените версии и те бъдат записани в `mix.lock`, то единственият вариант те да бъдат променени е експлицитно да се обновят. Затова е изключително важно да разпространявате `mix.lock`, заедно с останалата част от кода. Той гарантира, че абсолютно същите версии ще бъдат инсталирани всеки път.

### Как да използваме всичко споменато до тук
По-рано създадохме създадоха проекта `github_client` и добавихме две библиотеки към него. Нашият `github_client.ex` изглежда така:
```elixir
defmodule GithubClient do
  @moduledoc """
    Fetch information from the github API
  """
  require Logger

  @github_url "http://api.github.com/"
  @seconds_in_day 60 * 60 * 24

  def issues(org, repo, days_old \\ 30) do
    case HTTPoison.get(issues_url(org, repo), [], follow_redirect: true, max_redirect: 5) do
      {:ok, %HTTPoison.Response{status_code: 200, body: body}} ->
        # |> Enum.map(fn %{"title" => title} -> title end)
        body
        |> Poison.decode!()
        |> Enum.filter(fn %{"created_at" => datetime_iso8601} ->
          {:ok, datetime, _} = DateTime.from_iso8601(datetime_iso8601)
          DateTime.diff(DateTime.utc_now(), datetime) < @seconds_in_day * days_old
        end)
        |> Enum.map(fn %{"title" => title} -> title end)

      {:ok, %HTTPoison.Response{status_code: 404}} ->
        Logger.warn("Github issues for '#{org}/#{repo}' not found")
    end
  end

  defp issues_url(org, repo) do
    @github_url <> "repos/#{org}/#{repo}/issues"
  end
end
```

