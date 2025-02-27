# Хвостовая рекурсия

Надо понимать, что рекурсия -- штука не бесплатная. Каждый рекурсивный вызов требует сохранения на стеке предыдущего состояния функции, чтобы вернуться в него после очередного вызова.

И хотя стек в BEAM легковесный и позволяет делать миллионы рекурсивных вызовов, (а не тысячи, как в большинстве мейнстримовых языков), он, все-таки, конечный.

Поэтому в Эликсир, как и во многих других функциональных языках, компилятор делает одну полезную штуку, которая называется **оптимизация хвостовой рекурсии**.

Если рекурсивный вызов является последней строчкой кода в данной функции, и после него больше никаких инструкций нет, то такой вызов называется хвостовым. В этом случае нет необходимости возвращаться по стеку во все вызывающие функции, а можно сразу отдать результат из последнего вызова. И стек не растет, а используется заново каждым новым вызовом, и адрес возврата в нем не меняется.

Это позволяет делать бесконечную рекурсию, которая нужна для бесконечно живущих процессов.  А такие процессы нужны серверам :)

Значит ли это, что всегда нужно всегда использовать хвостовую рекурсию? Нет.

Код без хвостовой рекурсии часто получается короче и проще, а иногда и лучше по производительности. Если нам нужна бесконечная, или очень долгая рекурсия (миллионы итераций), то без хвостовой рекурсии не обойтись. Но если мы уверены, что у нас будет конечное и не очень большое число итераций, то можно выбрать вариант без неё.


## Факториал

В книге Learn Functional Programming with Elixir by Ulisses Almeida описываются две реализации факториала, с хвостовой рекурсией и без нее. И они сравниваются по потреблению памяти, по данным System Monitor для процесса beam.smp.

Обе реализации просты:
```
defmodule Factorial do

  def factorial(0), do: 1
  def factorial(n) when is_integer(n) and n > 0 do
    n * factorial(n - 1)
  end

  def factorial_t(n) do
    factorial_t(n, 1)
  end

  defp factorial_t(0, acc), do: acc
  defp factorial_t(n, acc) do
    factorial_t(n - 1, n * acc)
  end

end
```

У автора получилось, что в реализации без хвостовой рекурсии процесс beam.smp при вычислении факториала 10_000_000 потреблял 756Mb памяти. А с хвостовой рекурсией -- 56Mb. Разница впечатляющая.

Я попробовал воспроизвести эти результаты на версии OTP 23.2 (у автора версия не указана), и у меня получилось так:

Без хвостовой рекурсии:
```
factorial(20_000) - 96Mb
factorial(40_000) - 113Mb
factorial(100_000) - 1.5G,
factorial(200_000) - 2.5G, не вычислил, beam убит, вероятно ООМ киллером. (У меня на ноуте 16Gb памяти, но свободной было 3Gb).
```

С хвостовой рекурсией:
```
factorial(20_000) - 96Mb
factorial(40_000) - 200Mb
factorial(100_000) - 1.3G
```

Что ж, на практике все оказалось не так все просто.

Нужно смотреть, что за память используется. BEAM предлагает для этого
```
:erlang.process_info(self(), :memory)
```

Добавим этот вызов на каждый 1000-й шаг рекурсии:
```
defmodule Lesson_04.Task_04_04_TailRecursion do

  def factorial(0), do: 1
  def factorial(n) when is_integer(n) and n > 0 do
    if rem(n, 1000) == 0, do: report_memory()
    n * factorial(n - 1)
  end

  def factorial_t(n) do
    factorial_t(n, 1)
  end

  defp factorial_t(0, acc), do: acc
  defp factorial_t(n, acc) do
    if rem(n, 1000) == 0, do: report_memory()
    factorial_t(n - 1, n * acc)
  end


  # http://erlang.org/documentation/doc-5.7.4/erts-5.7.4/doc/html/erlang.html#process_info-1
  # memory is measured in bytes, but head and stack are measured in machine words
  # which is 4 bytes for 64 bit arch
  defp report_memory() do
    data = :erlang.process_info(self(), [:memory, :total_heap_size, :stack_size])
    IO.puts("memory #{data[:memory]}, heap #{data[:total_heap_size]}, stack #{data[:stack_size]}")
  end

end
```

И чтобы отсечь внутренее состояние процесса shell, будем запускать вычисление факториала в отдельном процессе:

```
iex(16)> spawn(Factorial, :factorial, [20_000])
#PID<0.136.0>
memory 2688, heap 233, stack 7
memory 55000, heap 6772, stack 6007
memory 142672, heap 17731, stack 12007
memory 230344, heap 28690, stack 18007
memory 230344, heap 28690, stack 24007
memory 372200, heap 46422, stack 30007
memory 372200, heap 46422, stack 36007
memory 372200, heap 46422, stack 42007
memory 601728, heap 75113, stack 48007
memory 601728, heap 75113, stack 54007
memory 601728, heap 75113, stack 60007
memory 601728, heap 75113, stack 66007
memory 601728, heap 75113, stack 72007
memory 973112, heap 121536, stack 78007
memory 973112, heap 121536, stack 84007
memory 973112, heap 121536, stack 90007
memory 973112, heap 121536, stack 96007
memory 973112, heap 121536, stack 102007
memory 973112, heap 121536, stack 108007
memory 973112, heap 121536, stack 114007

iex(17)> spawn(Factorial, :factorial_t, [20_000])
#PID<0.138.0>
memory 2688, heap 233, stack 8
memory 907040, heap 113277, stack 8
memory 2686696, heap 335734, stack 8
memory 12085240, heap 1510552, stack 8
memory 13841864, heap 1730130, stack 8
memory 15587544, heap 1948340, stack 8
memory 17321528, heap 2165088, stack 8
memory 19043112, heap 2380286, stack 8
memory 20751312, heap 2593811, stack 8
memory 22445040, heap 2805527, stack 8
memory 24123064, heap 3015280, stack 8
memory 25783880, heap 3222882, stack 8
memory 27425672, heap 3428106, stack 8
memory 29046264, heap 3630680, stack 8
memory 30642456, heap 3830204, stack 8
memory 32210896, heap 4026259, stack 8
memory 33746272, heap 4218181, stack 8
memory 35241072, heap 4405031, stack 8
memory 36683160, heap 4585292, stack 8
memory 38049928, heap 4756138, stack 8
```

Тут мы видим, что стек ведет себя как должен -- растет в реализации без хвостовой рекурсии, и не растет в хвостовой. При этом быстро растет память в куче. И хвостовая рекурсия потребляет в 40 раз больше памяти.

Если посмотреть на результат вычисления факториала от 20_000, то это будет огромное число на много экранов, которое нужно долго скролить, чтобы увидеть все целиком. Очевидно, что такое число не помещается на стеке, и память для него выделяется в куче. Можно предположить, что хвостовая рекурсия хранит много таких промежуточных результатов, пока их не почистит сборщик мусора, и поэтому потребляет больше памяти.


## Последовательность Фибоначчи

Что ж, факториал -- неудачный пример. Давайте посмотрим что-нибудь другое. Ну возьмем последовательность Фибоначчи, как еще один классический книжный пример рекурсии.

```
  def fibonacci(0), do: 0
  def fibonacci(1), do: 1
  def fibonacci(n) do
    report_memory()
    fibonacci(n - 1) + fibonacci(n - 2)
  end

  def fibonacci_t(0), do: 0
  def fibonacci_t(1), do: 1
  def fibonacci_t(n) do
    report_memory()
    fibonacci_t(n, 2, {0, 1})
  end

  def fibonacci_t(till, step, {fn_2, fn_1}) when till == step do
    report_memory()
    fn_2 + fn_1
  end
  def fibonacci_t(till, step, {fn_2, fn_1}) do
    fibonacci_t(till, step + 1, {fn_1, fn_1 + fn_2})
  end
```

Внезапно нас настигают сложности -- для нехвостовой рекурсии нельзя сказать, какой сейчас шаг. Если протаскивать счетчик шагов через аргументы, то это может исказить результат. В итоге я сделал вызов report_memory на каждом шагу.

Теперь я не могу сделать слишком большое число шагов, потому что это забьет всю консоль, и трудно будет следить за данными. Тем более, что в не хвостовой реализации n много раз прыгает назад. Зато в хвостовой реализации можно взять данные только по первому и последнему шагу:

```
iex(20)> R.fibonacci(6)
memory 34988, heap 4184, stack 58
memory 34988, heap 4184, stack 60
memory 34988, heap 4184, stack 62
memory 34988, heap 4184, stack 64
memory 34988, heap 4184, stack 66
memory 34988, heap 4184, stack 64
memory 34988, heap 4184, stack 62
memory 34988, heap 4184, stack 64
memory 34988, heap 4184, stack 60
memory 34988, heap 4184, stack 62
memory 34988, heap 4184, stack 64
memory 34988, heap 4184, stack 62
8
iex(21)> R.fibonacci_t(20)
memory 34988, heap 4184, stack 58
memory 34988, heap 4184, stack 59
6765
```

Ну ок, стек ведет себя как надо. Куча вообще не растет на такой короткой рекурсии. Пример опять оказался неудачным. Надо попробовать что-то еще.


## Сумма всех элементов списка.

Возьмем еще одну банальную рекурсию, где аккумулятор не будет хранить такие огромные данные, как у факториала, а шаги рекурсии будут простыми.

```
  def sum([]), do: 0
  def sum([head | tail]) do
    report_memory()
    head + sum(tail)
  end

  def sum_t(list) do
    sum_t(list, 0)
  end

  def sum_t([], acc), do: acc
  def sum_t([head | tail], acc) do
    report_memory()
    sum_t(tail, head + acc)
  end
```

Что получится?

```
iex(32)> list = Enum.to_list(0..100)
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21,
 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41,
 42, 43, 44, 45, 46, 47, 48, 49, ...]
iex(5)> R.sum(list)
memory 34460, heap 4184, stack 59
memory 34460, heap 4184, stack 61
memory 34460, heap 4184, stack 63
...
memory 29572, heap 3573, stack 255
memory 29572, heap 3573, stack 257
memory 29572, heap 3573, stack 259
5050
iex(6)> R.sum_t(list)
memory 29572, heap 3573, stack 60
memory 29572, heap 3573, stack 60
memory 29572, heap 3573, stack 60
...
memory 42364, heap 5172, stack 60
memory 42364, heap 5172, stack 60
memory 42364, heap 5172, stack 60
5050
```

Вот теперь наглядно видно, как растет стек. Куча при этом меняется не сильно, и может даже уменьшаться. Очевидно из-за сборки мусора.


## Вывод

В книгах не редко пишут, что хвостовая рекурсия эффективнее. Мы уже убедились, что на практике это не всегда так, и каждый конкретный случай нужно проверять.

В документации по Эрланг есть раздел Efficiency Guide о том, как писать эффективный код. Помимо прочего, там разбирается и хвостовая рекурсия:
[The Seven Myths of Erlang Performance](https://erlang.org/doc/efficiency_guide/myths.html). Разумеется, это справедливо и для Эликсир.

К тому же, у не хвостовой рекурсии код обычно проще и короче, что тоже очень важно.
