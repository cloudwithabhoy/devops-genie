# Bash Scripting — Interview Questions

---

## 1. Write a shell script to calculate the factorial of a number.

> **Also asked as:** "Factorial script in Bash"

This is a classic logic test to see if you understand loops and variable arithmetic in shell scripting.

**Using a `for` loop:**

```bash
#!/bin/bash

echo "Enter a number:"
read num

factorial=1

for (( i=1; i<=num; i++ ))
do
  factorial=$((factorial * i))
done

echo "The factorial of $num is $factorial"
```

**Using a `while` loop:**

```bash
#!/bin/bash

read -p "Enter a number: " num
temp=$num
fact=1

while [ $temp -gt 1 ]
do
  fact=$((fact * temp))
  temp=$((temp - 1))
done

echo "Factorial of $num is $fact"
```

**Key Points to Mention:**
- **Shebang (`#!/bin/bash`):** Tells the kernel which interpreter to use.
- **Arithmetic Expansion (`$((...))`):** Used for performing integer calculations.
- **Input handling:** Using `read` to get user input.
- **Edge cases:** A good candidate mentions that `0!` is `1` and handling negative inputs.
