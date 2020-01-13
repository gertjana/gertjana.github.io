---
layout: post
title: Practical recursion example in Elixir
comments: true
categories: development
tags:
- ''
- elixir
---

As the new year starts, a lot of people are starting their "new years resolutions", which usually are things like exercise more, eat healthier, which non-quantitive nature are easy to postpone or avoid,  therefor I rather set goals. 

I have an electric bike that keeps track of the total km's cycled, in 2018 I cycled ~5000 km, in 2019 ~3000 km, so a downward trend was showing. (The electric car I got in 2019 might have something to do with that)

One of my (modest) goals for 2020 was to cycle at least 4000km which is around 80 km a week (my daily commute is around 35km), so if I cycle to work at 2 to 3 times a week, I will make it easily.

Now to help keep track, I wrote a small command line tool, that allows my to enter the current kilometers, and predict using linear trend analysis where it would end up at the end of the year.

Linear trend analysis, takes the average change between measurements and then apply that average to a future value. In formula form this is

{% raw %}
$$\frac{\frac{v_{x} \ -v_{n}}{t_{x} \ -t_{n}}}{1} =\frac{\frac{v_{1} -v_{0}}{t_{1} -t_{0}} \ +\ \frac{v_{2} \ -v_{1}}{t_{2} \ -t_{1}} \ ...\frac{v_{n} \ -\ v_{n-1}}{t_{n} -t_{n-1}}}{n}$$
{% endraw %}

or .. the average of all the previous points (0 to n) is equal to the average of the last value (n) and the future value (x) divided by the number of days from the last value to that future date. 

so what we need is a recursive function that does this part:

{% raw %}
$$\frac{v_{1} -v_{0}}{t_{1} -t_{0}} \ +\ \frac{v_{2} \ -v_{1}}{t_{2} \ -t_{1}} \ ...\frac{v_{n} \ -\ v_{n-1}}{t_{n} -t_{n-1}}$$
{% endraw %}

```elixir
  deltas = loop([], nil, measurements)

  defp loop(acc, prev, list) do
    case list do
      [] -> acc
      [head | tail] when prev == nil -> loop(acc, head, tail)
      [head | tail] ->
        delta = (head.value - prev.value) / diff_string_date(prev.date, head.date)
        loop([delta | acc], head, tail)
    end
  end
```

 * We start with an empty accumulator, no previous value and the list of km's cycled
 * when we don't have a previous value, we call ourselves with the head of the list as the previous value and the tail as the list
 * when we have a previous value we calculate the average km/days between the previous and the head of the list, add it to the accumulator and call ourselves again with the head as the previous value and the tail 
 * when we call it with and empty list we return the accumulator (exit condition)

A big thing in recursion is tail recursive or tail-call optimization.

Normally when you call function B from function A the current state of function A will be put on the stack end retreived again when Function B returns, except when the call to function B is the last thing function A does. then it only has to return the result of function B

When calling a recursive function with a large list, this could cause the stack to grow beyond it's memory allowance and crash your application, solving this is calling 'Tail call optimization' and just means we need to make sure the recursive call is the last statement in the function.

We can see from the code that we call ourselves from 2 places, so how can we optimize this loop to be tail call optimized?

As the only thing this function does is pattern match on a list and Elixir also support pattern matching on function arguments we can rewrite this loop as several functions where the arguments vary as they did in the case statement.

```elixir
  deltas = loop([], nil, measurements)

  defp loop(acc, _, []), do: acc
  defp loop(acc, nil, [head | tail]), do: loop(acc, head, tail)
  defp loop(acc, prev, [head | tail]) do
    diff = (head.value - prev.value) / diff_string_date(prev.date, head.date)
    loop([diff | acc], head, tail)
  end
```

You can see we only call ourselves as the last statement, yeah, we're tail call optimized and the compiler will take it from here.

Then all its left is calculating the average of the deltas and multiply it with the number of days from the last measurement to the 31st of December and we are there

```fish
~>./van_moofing trend 2020
With a average of 6.72 km a day and 353 days left in the year,
you will have cycled 2372.0 km in 2020
```

So I didn't cycle that much in the first two weeks, let's blame that on the weather ;)

BTW: I used the excelent [ex_cli](https://hex.pm/packages/ex_cli) library for creating a commandline app, with argument parsing
