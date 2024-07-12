#!elixir

defmodule Format do
  @moduledoc "Convenience functions for formatting printed text"
  alias IO.ANSI

  def color(color, string) do
    "#{ANSI.bright()}#{Function.capture(ANSI, color, 0).()}#{string}#{ANSI.reset()}"
  end

  def bg(color, string) do
    "#{ANSI.bright()}#{Function.capture(ANSI, String.to_atom("#{color}_background"), 0).()}#{string}#{ANSI.reset()}"
  end

  def wordlize(string) do
    string
    |> String.upcase()
    |> String.graphemes()
    |> Enum.map_join(&random_letter_highlighter().(&1))
  end

  def header(text), do: "#{ANSI.underline()}#{color(:red, text)}"
  def cmd(text), do: color(:cyan, text)
  def obj(text), do: "#{ANSI.italic()}#{color(:magenta, text)}"

  def g(text), do: color(:green, text)
  def y(text), do: color(:yellow, text)
  def b(text), do: color(:black, text)
  def w(text), do: color(:white, text)
  def gb(text), do: bg(:green, text)
  def yb(text), do: bg(:yellow, text)
  def bb(text), do: bg(:black, text)

  defp random_letter_highlighter do
    Enum.random([&gb/1, &yb/1, &w(bb(&1))])
  end
end

defmodule GuessAndMarks do
  @moduledoc """
  Represents a player-entered guess word + the game's corresponding marks on that guess.
  """

  @enforce_keys ~w[guess marks]a
  defstruct @enforce_keys

  def parse(string) do
    case form_of(string) do
      :separate -> {:ok, parse_separate(string)}
      :interleaved -> {:ok, parse_interleaved(string)}
      :error -> :error
    end
  end

  def valid?(string), do: form_of(string) != :error

  defp form_of(string) do
    cond do
      Regex.match?(~r/^[a-zñ]{5} [!?x]{5}$/iu, string) -> :separate
      Regex.match?(~r/^([a-zñ][!?x]){5}$/iu, string) -> :interleaved
      true -> :error
    end
  end

  defp parse_separate(guess_and_marks) do
    [guess, marks] = String.split(guess_and_marks)

    %__MODULE__{
      guess: String.downcase(guess),
      marks: marks
    }
  end

  defp parse_interleaved(guess_and_marks) do
    graphemes = String.graphemes(guess_and_marks)

    guess =
      graphemes
      |> Enum.take_every(2)
      |> Enum.join()

    marks =
      graphemes
      |> Enum.drop(1)
      |> Enum.take_every(2)
      |> Enum.join()

    %__MODULE__{
      guess: String.downcase(guess),
      marks: marks
    }
  end

  defimpl String.Chars do
    import Format

    def to_string(%{guess: guess, marks: marks}) do
      Enum.zip(String.graphemes(marks), String.graphemes(guess))
      |> Enum.map(fn {m, l} -> {m, String.upcase(l)} end)
      |> Enum.map(fn
        {"!", l} -> gb(l)
        {"?", l} -> yb(l)
        {"x", l} -> w(bb(l))
      end)
      |> Enum.join()
    end
  end
end

defmodule Marker do
  @doc """
  Given a player-entered guess word and a target "word of the day",
  this returns a string representing the corresponding marks
  that the game would give.
  '!' = letter is in the correct position (green)
  '?' = letter is present in the word but not in the correct position (yellow)
  'x' = letter is not in the word (gray)

      iex> Marker.mark("adieu", "hoard")
      "??xxx"

      iex> Marker.mark("nieto", "otoño")
      "xxx?!"

      iex> Marker.mark("chore", "close")
      "!x!x!"

      iex> Marker.mark("banal", "polar")
      "xxx!?"
  """
  def mark(guess, target) do
    # First pass: mark correct letters
    {guess, target} = mark_correct(guess, target)

    # Second pass: mark present letters
    guess =
      target
      |> String.graphemes()
      |> Enum.reduce(guess, &String.replace(&2, &1, "?", global: false))

    # Final pass: mark all remaining unmarked letters as not present
    String.replace(guess, ~r/[^!?]/u, "x")
  end

  defp mark_correct(guess, target) do
    [String.graphemes(guess), String.graphemes(target)]
    |> Enum.zip()
    |> Enum.reduce({[], []}, fn
      {l, l}, {guess_acc, target_acc} -> {[guess_acc, "!"], target_acc}
      {g, t}, {guess_acc, target_acc} -> {[guess_acc, g], [target_acc, t]}
    end)
    |> then(fn {guess, target} ->
      {to_string(guess), to_string(target)}
    end)
  end
end

defmodule Matcher do
  @doc """
  Given a list of player-entered guesses and their corresponding marks,
  this returns a list of all words in the dictionary that could match.

  Options:
  - `:lang` - `:en` or `:es`, determines which dictionary to use as a source. Default `:es`.
  - `:sort` - if `:vowels`, sorts the results (descending) by the number of unique vowels in the word

      iex> Matcher.get_matches([{"adieu", "??xxx"}])
      [..., "hoard", ...]
  """
  def get_matches(guesses_and_marks, opts \\ []) do
    Path.join([__DIR__, ".wordle_helper", "./dictionary_#{opts[:lang] || :en}.txt"])
    |> File.stream!()
    |> Stream.map(&String.trim/1)
    |> Stream.filter(fn candidate ->
      Enum.all?(guesses_and_marks, fn %{guess: guess, marks: marks} ->
        Marker.mark(guess, candidate) == marks
      end)
    end)
    |> then(fn matches ->
      case opts[:sort] do
        nil -> matches
        :vowels -> Enum.sort_by(matches, &unique_vowel_count/1, :desc)
      end
    end)
    |> Enum.to_list()
  end

  defp unique_vowel_count(s) do
    s
    |> String.graphemes()
    |> Enum.uniq()
    |> Enum.count(fn g -> g in ~w[a e i o u] end)
  end
end

defmodule Runner do
  defstruct guesses_and_marks: [],
            lang: :en

  @help_message (fn ->
                   import Format

                   """
                   #{header("BASIC USAGE")}
                   Record a #{obj("GUESS_AND_MARKS")} for each word you guess.
                   Use #{cmd("suggest")} or #{cmd("suggest vowels")} to see potential matches.

                   #{header("GUESS_AND_MARKS")}
                   A #{obj("GUESS_AND_MARKS")} describes one line from the game board.

                   Example: adieu #{y("?")}#{y("?")}#{g("!")}#{b("x")}#{b("x")}
                   It's composed of your 5-letter guess, followed by 5 punctuation marks
                   representing the marks that the game gave it.

                   #{g("!")} = letter is in the correct position (Green)
                   #{y("?")} = letter is in the word but not in the correct position (Yellow)
                   #{b("x")} = letter is not in the word (Gray)

                   For example, you would enter #{yb("G")}#{gb("A")}#{w(bb("U"))}#{gb("N")}#{yb("T")} as 'gaunt #{y("?")}#{g("!")}#{b("x")}#{g("!")}#{y("?")}'.

                   If you like, you can also "interleave" the letters and marks and
                   the script will still understand it:
                   a#{y("?")}d#{y("?")}i#{g("!")}e#{b("x")}u#{b("x")}

                   #{header("ALL COMMANDS")}
                   #{cmd("help")} | #{cmd("h")}
                     Print this help message.
                   #{cmd("record")} #{obj("GUESS_AND_MARKS")}
                     Record a guess you made and the game's marks on it.
                     Shorthand: Enter #{obj("GUESS_AND_MARKS")} without a command
                   #{cmd("undo")}
                     Forget the last #{obj("GUESS_AND_MARKS")} you recorded.
                   #{cmd("suggest")} | #{cmd("s")}
                     List words from the dictionary that fit the guesses and marks you've entered so far.
                   #{cmd("suggest vowels")} | #{cmd("sv")}
                     Same as #{cmd("suggest")}, but prioritizes words containing a lot of vowels.
                   #{cmd("lang")} #{obj("en")}|#{obj("es")}
                     Set the language of the game you're playing. English and Spanish are supported.
                   #{cmd("show state")} | #{cmd("show")}
                     Print out the current language and the guesses and marks you've entered so far.
                   #{cmd("reset")} | #{cmd("restart")}
                     Clear the guesses and marks you've entered.
                   #{cmd("quit")} | #{cmd("exit")} | #{cmd("q")}
                     Exit the helper.
                   """
                   |> String.trim_trailing()
                 end).()

  @prompt "#{IO.ANSI.underline()}wordle#{IO.ANSI.reset()}> "

  @doc """
  Run the dang darn thing
  """
  def run do
    greet()

    Stream.repeatedly(fn -> IO.gets(@prompt) end)
    |> Stream.map(&String.trim/1)
    |> Stream.map(&parse_command/1)
    |> Stream.each(fn _ -> IO.puts("") end)
    |> Stream.scan(%__MODULE__{}, &process_command/2)
    |> Stream.take_while(&(not is_nil(&1)))
    |> Stream.each(fn _ -> IO.puts("") end)
    |> Stream.run()
  end

  defp greet do
    Enum.random(~w[HELLO HOWDY HOLA! AHOY!])
    |> Format.wordlize()
    |> IO.puts()

    IO.puts("")
    IO.puts("Enter a command (#{Format.cmd("h")} for help)")
  end

  defp parse_command("lang " <> lang) when lang in ~w[en es],
    do: {:set_lang, String.to_atom(lang)}

  defp parse_command("record " <> guess_and_marks), do: {:record, guess_and_marks}

  defp parse_command("suggest"), do: {:suggest, nil}
  defp parse_command("s"), do: {:suggest, nil}
  defp parse_command("suggest vowels"), do: {:suggest, :vowels}
  defp parse_command("sv"), do: {:suggest, :vowels}
  defp parse_command("undo"), do: :remove_last_entry
  defp parse_command("show"), do: :show_state
  defp parse_command("show state"), do: :show_state
  defp parse_command("reset"), do: :reset
  defp parse_command("restart"), do: :reset
  defp parse_command("help"), do: :help
  defp parse_command("h"), do: :help
  defp parse_command("quit"), do: :quit
  defp parse_command("exit"), do: :quit
  defp parse_command("q"), do: :quit

  defp parse_command(s) do
    if GuessAndMarks.valid?(s), do: {:record, s}, else: :error
  end

  defp process_command({:set_lang, new_lang}, state) do
    IO.puts("ok")
    %{state | lang: new_lang}
  end

  defp process_command({:record, guess_and_marks}, state) do
    case GuessAndMarks.parse(guess_and_marks) do
      {:ok, parsed} ->
        IO.puts("Recorded #{parsed}")
        %{state | guesses_and_marks: [parsed | state.guesses_and_marks]}

      :error ->
        IO.puts("Invalid format")
        state
    end
  end

  defp process_command({:suggest, sort_opt}, state) do
    state.guesses_and_marks
    |> Matcher.get_matches(lang: state.lang, sort: sort_opt)
    |> tap(fn matches ->
      IO.inspect(matches)

      count = length(matches)

      results =
        case count do
          0 -> "results :("
          1 -> "result 🎉"
          _ -> "results"
        end

      IO.puts("#{count} #{results}")
    end)

    state
  end

  defp process_command(:remove_last_entry, state) do
    case state.guesses_and_marks do
      [] ->
        IO.puts("Nothing to undo")
        state

      [to_remove | rest] ->
        IO.puts("Removed #{to_remove}")
        %{state | guesses_and_marks: rest}
    end
  end

  defp process_command(:show_state, state) do
    IO.puts(state)

    state
  end

  defp process_command(:reset, state) do
    IO.puts("ok")
    %{state | guesses_and_marks: []}
  end

  defp process_command(:help, state) do
    IO.puts(@help_message)

    state
  end

  defp process_command(:quit, _state) do
    Enum.random(~w[ADIEU ADIOS CIAO! LATER GBYE!])
    |> Format.wordlize()
    |> IO.puts()

    nil
  end

  defp process_command(:error, state) do
    IO.puts("Command not recognized")
    state
  end

  defimpl String.Chars do
    def to_string(state) do
      lang =
        case state.lang do
          :en -> "English"
          :es -> "Spanish"
        end

      guesses_and_marks =
        state.guesses_and_marks
        |> Enum.reverse()
        |> Enum.with_index(1)
        |> Enum.map_join("\n", fn {g_m, i} -> "#{i}) #{g_m}" end)

      """
      Language: #{lang}
      Guesses and marks:
      #{guesses_and_marks}
      """
      |> String.trim_trailing()
    end
  end
end

Runner.run()
