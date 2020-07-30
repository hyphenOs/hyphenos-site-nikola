.. title: Thinking in Python - Part 2
.. slug: thinking-in-python-2
.. date: 2019-03-31 11:57:04 UTC+05:30
.. tags: Python
.. link:
.. description:
.. type: text
.. status:
.. author: Abhijit Gadgil
.. summary: This is second post in our 'Thinking in Python' series. This is a blog post about how to run simple random experiments using Python. In this post we will look at how to simulate and verify Birthday Paradox.

# Introduction

Birthday paradox problem discusses that in a group of randomly chosen people, what is the probability that two or more people will share their birthdays? [Wikipedia](https://en.wikipedia.org/wiki/Birthday_problem){:target="\_blank"} has got an extensive mathematical treatment of the problem and it surely by itself is worth a read. But somewhat quite un-intuitive result is if there are about 25 people in a room, the probability that at-least two of them will share their birthday is about half. The result might sound very surprising, but is indeed true.

Let's say we want to verify that this result is indeed true? How can we go about it?

# An Approach

What if we run an experiment where we randomly select a group of N people many times and actually find out how many of those times the above particular assertion holds true. Is that solution going to be close to actual mathematical probability?


## First Steps

How do you find birthdays for individuals? It's actually pretty simple if we ignore the complexity associated with leap years, and choose random numbers between 1 and 365, we can call that number as a birthday of a person. So more concretely, let's find out random birthdays of 10 persons using Python.

```python

import random

birthdays = [random.randint(1,365) for i in range(10)]

print (birthdays)
```

Let's say we want to run this experiment for 1000 times, so essentially 1000 times, we are randomly finding birthdays of 10 persons. This can be done as follows


```python

import random

for i in range(1000):
	birthdays = [random.randint(1,365) for i in range(10)]

```

## Calculating Empirical Probability

We need to now find out, if we run the experiment large number of times how many of those number of times, we actually have 2 or more birthdays that are same. This problem can be stated as follows, if for N people, if number of unique birthdays is N, that means no two people share the birthday. These become results of a single run. To find out empirical probability, we will have a large number of such runs and find out how many of those throw up duplicate birthdays. This can be done in python simply as follows

```python

import random

def are_birthdays_repeated(N):
    birthdays = [random.randint(1, 365) for i in range(10)

    unique_birthdays = set(birthdays)

    if len(unique_birthdays) == len(birthdays):
        return False

    return True
```

Now We can run this experiment for some really large number of runs and calculated unique birthdays in those runs and thus empirical probability.

```python

def calc_empirical_prob(num_of_pepole, runs=1000):

    repeated_birthdays_runs = 0
    for i in range(runs):
        if are_birthdays_repeated(num_of_people):
            repeated_birthdays_runs += 1

    empirical_probability = repeated_birthdays_runs / runs

    return empirical_prbability

```

We can conclude this by finding such probabilities for number of peoples in a room from 2 to 100 and we'll observe the results and see that these probabilities are indeed very close to what are the actual probabilities computed using mathematical formula.

The final code could look like -

```python

import random

def are_birthdays_repeated(N):
    birthdays = [random.randint(1, 365) for i in range(N)]

    unique_birthdays = set(birthdays)

    if len(unique_birthdays) == len(birthdays):
        return False

    return True

def calc_empirical_prob(num_of_people, runs=1000):

    repeated_birthdays_runs = 0
    for i in range(runs):
        if are_birthdays_repeated(num_of_people):
            repeated_birthdays_runs += 1

    empirical_probability = repeated_birthdays_runs / runs

    return empirical_probability

for i in range(2, 100):
    print(i, calc_empirical_prob(i,runs=1000)
```

## Conclusion

We have looked at a very simple problem and calculated empirical probabilities. Shown below are results from one such run.

| Number   | Probability |
|:---------| -----------:|
|2  | 0.003 |
|3  | 0.005 |
|4  | 0.01 |
|5  | 0.026 |
|6  | 0.043 |
|7  | 0.049 |
|8  | 0.066 |
|9  | 0.101 |
|10 | 0.124 |
|11 | 0.152 |
|12 | 0.178 |
|13 | 0.216 |
|14 | 0.204 |
|15 | 0.249 |
|16 | 0.268 |
|17 | 0.331 |
|18 | 0.345 |
|19 | 0.394 |
|20 | 0.405 |
|21 | 0.448 |
|22 | 0.465 |
|23 | 0.502 |
|24 | 0.566 |
|25 | 0.61 |
|26 | 0.63 |
|27 | 0.664 |
|28 | 0.656 |
|29 | 0.671 |
|30 | 0.718 |
|31 | 0.729 |
|32 | 0.746 |
|33 | 0.791 |
|34 | 0.775 |
|35 | 0.833 |
|36 | 0.821 |
|37 | 0.843 |
|38 | 0.862 |
|39 | 0.89 |
|40 | 0.903 |
|41 | 0.906 |
| 42 | 0.911 |
| 43 | 0.918 |
| 44 | 0.918 |
| 45 | 0.952 |
| 46 | 0.95 |
| 47 | 0.943 |
| 48 | 0.961 |
| 49 | 0.968 |
| 50 | 0.972 |
