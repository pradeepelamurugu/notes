# Bit Manipulation Techniques

## 1. Swapping Two Numbers (Without a Third Variable)

Traditionally, swapping requires a temp variable. Using XOR (^), you can swap two numbers $A$ and $B$ in three steps:

- **Step 1:** `a = a ^ b`
- **Step 2:** `b = a ^ b` (Now $b$ becomes the original value of $a$)
- **Step 3:** `a = a ^ b` (Now $a$ becomes the original value of $b$)

## 2. Check if the $i$-th Bit is Set

To check if the bit at index $i$ (from right, starting at 0) is 1 or 0:

- **Method 1 (Left Shift):** Check if `(n & (1 << i)) != 0`. If true, the bit is set.
- **Method 2 (Right Shift):** Check if `((n >> i) & 1) == 1`. If true, the bit is set.

## 3. Setting the $i$-th Bit

To force the $i$-th bit to be 1 while keeping others unchanged:

- **Operation:** `n = n | (1 << i)`
- **Logic:** Using the OR (|) operator with a bitmask where only the $i$-th bit is 1 ensures that bit becomes 1 regardless of its previous state.

## 4. Clearing the $i$-th Bit

To force the $i$-th bit to be 0:

- **Operation:** `n = n & ~(1 << i)`
- **Logic:** Create a mask where only the $i$-th bit is 0 (using the NOT operator ~) and perform an AND operation.

## 5. Toggling the $i$-th Bit

To flip the $i$-th bit (0 to 1, or 1 to 0):

- **Operation:** `n = n ^ (1 << i)`
- **Logic:** XORing a bit with 1 always flips it.

## 6. Removing the Rightmost Set Bit

To turn off the last '1' bit from the right:

- **Operation:** `n = n & (n - 1)`
- **Observation:** Subtracting 1 from a number flips the rightmost set bit and all zeros to its right. Performing an AND with the original number clears that specific bit.

## 7. Check if a Number is a Power of Two

A number that is a power of two has exactly one set bit in its binary representation.

- **Check:** `if ((n & (n - 1)) == 0)`
- **Note:** You must also ensure $n > 0$ for this to be valid.

## 8. Counting Set Bits

The following methods count the number of 1s in binary:

- **Iterative Division/Shift:** Loop while $n > 0$, incrementing the count if `n & 1` is 1, then right-shift $n$ by 1.
- **Brian Kernighan's Algorithm:** Use the "Remove last set bit" trick. Loop `n = n & (n - 1)` and increment a counter until $n$ becomes 0.
  - **Complexity:** $O(\text{number of set bits})$, which is faster for sparse numbers.
- **C++ Built-in:** `__builtin_popcount(n)`
