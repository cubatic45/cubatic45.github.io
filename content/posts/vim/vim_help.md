+++
title = 'Vim help'
date = 2025-02-10T11:24:02+08:00
draft = false
tags = ['vim']
+++

## Number Sequences

Sometimes we need to generate a sequence of increasing numbers, like:
	•	1, 2, 3, 4, 5, 6, 7, 8, 9, 10
	•	10, 20, 30, 40, 50, 60, 70, 80, 90, 100
	•	Or in a vertical list:

We can use the following Vim commands to generate these lists:

```vim
:put =''.join(range(1, 10), ',')  " Generates 1,2,3,...,9
:put =''.join(range(10, 100, 10), ',')  " Generates 10,20,30,...,90
:put =range(1, 10)  " Generates a vertical list of numbers
```

### Incrementing Numbers
Sometimes we need to change existing numbers, like:

```text
1 -> 2  # <C-a>
2 -> 3  # <C-a>
3 -> 4  # <C-a>
1 -> 2  # g<C-a>
2 -> 4  # g<C-a>
3 -> 6  # g<C-a>
```
We can use C-a (CTRL+A) to increment numbers.

Steps:
1.	Use `<C-v>` to enter Visual Mode and select the number you want to change.
2.	Press `<C-a>` (CTRL+A) to increment the selected number.
3.	Press `<C-x>` (CTRL+X) to decrement the selected number.
4.	Press `g<C-a>` (CTRL+A) to increment all the selected numbers.
5.	Press `g<C-x>` (CTRL+X) to decrement all the selected numbers.

