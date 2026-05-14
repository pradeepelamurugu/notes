# Java — CP / LeetCode Data Structures Syntax Guide

> Complete syntax reference for competitive programming. Each section covers: **declaration → insert → access → traverse → remove → useful methods**.

---

## Quick Index

| Data Structure | Class |
|---|---|
| String | `String`, `StringBuilder`, `char[]` |
| Array | `int[]`, `int[][]` |
| ArrayList | `ArrayList<>` |
| LinkedList | `LinkedList<>` |
| HashMap | `HashMap<K,V>` |
| LinkedHashMap | `LinkedHashMap<K,V>` |
| TreeMap | `TreeMap<K,V>` |
| HashSet | `HashSet<>` |
| LinkedHashSet | `LinkedHashSet<>` |
| TreeSet | `TreeSet<>` |
| Stack | `Stack<>` / `Deque` |
| Queue | `Queue<>` / `LinkedList` |
| Deque | `ArrayDeque<>` |
| PriorityQueue | `PriorityQueue<>` |
| Pair / int[] | `int[]` or `Map.Entry` |

---

## 1. String

### Declaration & Conversion

```java
String s = "hello";
String s2 = new String("hello");

// From char array
char[] ch = {'h', 'e', 'l', 'l', 'o'};
String s3 = new String(ch);

// From int / other types
String s4 = String.valueOf(42);          // "42"
String s5 = Integer.toString(42);

// String → int
int n = Integer.parseInt("42");
long l = Long.parseLong("123456789");

// String → char array
char[] arr = s.toCharArray();

// String → char at index
char c = s.charAt(2);                    // 'l'
```

### Traversal

```java
String s = "hello";

// Method 1: charAt index loop
for (int i = 0; i < s.length(); i++) {
    char c = s.charAt(i);
}

// Method 2: toCharArray for-each
for (char c : s.toCharArray()) {
    System.out.println(c);
}

// Method 3: chars() stream
s.chars().forEach(c -> System.out.println((char) c));
```

### Common String Methods

```java
String s = "Hello World";

s.length()                          // 11
s.charAt(0)                         // 'H'
s.indexOf('o')                      // 4
s.lastIndexOf('o')                  // 7
s.substring(6)                      // "World"
s.substring(0, 5)                   // "Hello"
s.toLowerCase()                     // "hello world"
s.toUpperCase()                     // "HELLO WORLD"
s.trim()                            // remove leading/trailing spaces
s.strip()                           // same but unicode-aware
s.replace('l', 'r')                 // "Herro Worrd"
s.replace("World", "Java")          // "Hello Java"
s.contains("World")                 // true
s.startsWith("Hello")               // true
s.endsWith("World")                 // true
s.equals("Hello World")             // true  — use this, NOT ==
s.equalsIgnoreCase("hello world")   // true
s.isEmpty()                         // false
s.isBlank()                         // false
s.split(" ")                        // ["Hello", "World"]
s.split("", 0)                      // split into individual chars
String.join("-", "a", "b", "c")     // "a-b-c"
Collections.frequency(list, 'a')    // count occurrences in list
```

### StringBuilder — use this to build strings in loops

```java
StringBuilder sb = new StringBuilder();

sb.append("hello");
sb.append(' ');
sb.append("world");
sb.insert(5, ",");                   // insert at index
sb.deleteCharAt(0);                  // delete char at index
sb.delete(0, 3);                     // delete range [0, 3)
sb.reverse();
sb.replace(0, 5, "Hi");             // replace range with new string
sb.charAt(0);
sb.setCharAt(0, 'H');
sb.length();
sb.toString();                       // convert back to String

// Build in a loop — O(n), not O(n²) like String concatenation
StringBuilder result = new StringBuilder();
for (char c : arr) {
    result.append(c);
}
String output = result.toString();
```

---

## 2. Arrays

### Declaration & Initialisation

```java
// 1D array
int[] arr = new int[5];              // [0, 0, 0, 0, 0]
int[] arr2 = {1, 2, 3, 4, 5};
int[] arr3 = new int[]{1, 2, 3};

// 2D array
int[][] grid = new int[3][4];        // 3 rows, 4 cols, all 0
int[][] grid2 = {
    {1, 2, 3},
    {4, 5, 6}
};

// Array of strings
String[] words = new String[3];
String[] words2 = {"apple", "banana", "cherry"};
```

### Insert & Access

```java
int[] arr = new int[5];

arr[0] = 10;                         // insert at index
arr[4] = 99;

int val = arr[2];                    // access
int len = arr.length;                // length (no parentheses!)
```

### Traversal

```java
int[] arr = {1, 2, 3, 4, 5};

// Index loop
for (int i = 0; i < arr.length; i++) {
    System.out.println(arr[i]);
}

// For-each
for (int x : arr) {
    System.out.println(x);
}

// 2D array traversal
int[][] grid = {{1,2},{3,4}};
for (int i = 0; i < grid.length; i++) {
    for (int j = 0; j < grid[0].length; j++) {
        System.out.print(grid[i][j] + " ");
    }
}
```

### Sorting & Useful Methods

```java
import java.util.Arrays;

int[] arr = {5, 2, 8, 1, 9};

Arrays.sort(arr);                        // ascending in-place
Arrays.sort(arr, 1, 4);                  // sort subarray [1, 4)
Arrays.fill(arr, 0);                     // fill all with 0
Arrays.fill(arr, 1, 4, -1);             // fill range with -1
Arrays.copyOf(arr, 3)                    // first 3 elements
Arrays.copyOfRange(arr, 1, 4)           // copy [1, 4)
Arrays.equals(arr, arr2)                 // compare two arrays
Arrays.toString(arr)                     // "[1, 2, 5, 8, 9]"
int idx = Arrays.binarySearch(arr, 5);  // only on sorted array

// Sort Integer[] descending (can't do on int[])
Integer[] boxed = {5, 2, 8, 1};
Arrays.sort(boxed, (a, b) -> b - a);    // descending

// Convert int[] to List
List<Integer> list = new ArrayList<>();
for (int x : arr) list.add(x);
// or: Arrays.stream(arr).boxed().collect(Collectors.toList())

// 2D array sort by first column
int[][] intervals = {{3,4},{1,2},{5,6}};
Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
```

---

## 3. ArrayList

### Declaration

```java
import java.util.ArrayList;
import java.util.List;

List<Integer> list = new ArrayList<>();
List<String>  words = new ArrayList<>();

// Initialise with values
List<Integer> list2 = new ArrayList<>(Arrays.asList(1, 2, 3));
List<String>  words2 = new ArrayList<>(List.of("a", "b", "c"));

// Fixed-size list (cannot add/remove)
List<Integer> fixed = Arrays.asList(1, 2, 3);
```

### Insert

```java
List<Integer> list = new ArrayList<>();

list.add(10);                        // append to end
list.add(0, 99);                     // insert at index 0 — shifts others
list.addAll(Arrays.asList(1,2,3));   // append all
list.addAll(0, other);               // insert all at index
list.set(1, 55);                     // replace at index
```

### Access & Remove

```java
int val = list.get(0);               // get by index
list.remove(0);                      // remove by index
list.remove(Integer.valueOf(10));    // remove by value (NOT index)
list.clear();                        // remove all
int size = list.size();
boolean has = list.contains(10);
int idx = list.indexOf(10);
int last = list.lastIndexOf(10);
list.isEmpty();
```

### Traversal

```java
List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3));

// For-each
for (int x : list) System.out.println(x);

// Index loop
for (int i = 0; i < list.size(); i++) System.out.println(list.get(i));

// Iterator
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    int x = it.next();
    if (x == 2) it.remove();        // safe remove during iteration
}

// Stream
list.stream().forEach(System.out::println);
```

### Sorting

```java
import java.util.Collections;

List<Integer> list = new ArrayList<>(Arrays.asList(5, 2, 8, 1));

Collections.sort(list);                        // ascending
Collections.sort(list, (a, b) -> b - a);       // descending
Collections.reverse(list);                     // reverse in-place
Collections.shuffle(list);                     // random shuffle
Collections.min(list);
Collections.max(list);
Collections.frequency(list, 2);               // count occurrences

list.sort((a, b) -> a - b);                   // list's own sort method
```

### 2D — List of Lists

```java
List<List<Integer>> result = new ArrayList<>();

List<Integer> row = new ArrayList<>();
row.add(1); row.add(2);
result.add(row);

// Initialise n rows
List<List<Integer>> graph = new ArrayList<>();
for (int i = 0; i < n; i++) graph.add(new ArrayList<>());

// Access
result.get(0).get(1);                // row 0, col 1
```

---

## 4. LinkedList

```java
import java.util.LinkedList;

LinkedList<Integer> ll = new LinkedList<>();

ll.addFirst(1);                      // add to front
ll.addLast(2);                       // add to back
ll.add(2, 99);                       // insert at index
ll.removeFirst();
ll.removeLast();
ll.getFirst();
ll.getLast();
ll.peek();                           // peekFirst, null if empty
ll.poll();                           // pollFirst, null if empty

// Same traversal as ArrayList
for (int x : ll) System.out.println(x);
```

---

## 5. HashMap

### Declaration

```java
import java.util.HashMap;
import java.util.Map;

Map<String, Integer> map = new HashMap<>();
Map<Character, Integer> freq = new HashMap<>();
Map<Integer, List<Integer>> adjList = new HashMap<>();
```

### Insert & Update

```java
map.put("apple", 1);                          // insert / overwrite
map.putIfAbsent("apple", 5);                  // insert only if key absent

// Frequency counter pattern
map.put(key, map.getOrDefault(key, 0) + 1);

// Java 8 merge — same as getOrDefault + put
map.merge(key, 1, Integer::sum);

// computeIfAbsent — useful for adjacency lists
adjList.computeIfAbsent(node, k -> new ArrayList<>()).add(neighbour);
```

### Access & Remove

```java
map.get("apple")                     // null if missing
map.getOrDefault("apple", 0)         // default if missing
map.containsKey("apple")             // true/false
map.containsValue(1)                 // true/false
map.remove("apple")                  // remove by key
map.remove("apple", 1)               // remove only if key=value
map.size()
map.isEmpty()
map.clear()
```

### Traversal

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1); map.put("b", 2);

// Keys only
for (String key : map.keySet()) System.out.println(key);

// Values only
for (int val : map.values()) System.out.println(val);

// Key-value pairs — most common
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    String key = entry.getKey();
    int val    = entry.getValue();
}

// Java 8 forEach
map.forEach((k, v) -> System.out.println(k + " -> " + v));
```

---

## 6. LinkedHashMap — insertion-order map

```java
import java.util.LinkedHashMap;

Map<String, Integer> map = new LinkedHashMap<>();
// Same API as HashMap
// Iteration order = insertion order

// LRU Cache trick — access-order
Map<Integer, Integer> lru = new LinkedHashMap<>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry<Integer,Integer> e) {
        return size() > capacity;
    }
};
```

---

## 7. TreeMap — sorted by key

```java
import java.util.TreeMap;

TreeMap<Integer, String> tmap = new TreeMap<>();

tmap.put(3, "three");
tmap.put(1, "one");
tmap.put(2, "two");

// Keys always in sorted order
tmap.firstKey()                      // 1
tmap.lastKey()                       // 3
tmap.floorKey(2)                     // 2  (≤ 2)
tmap.ceilingKey(2)                   // 2  (≥ 2)
tmap.lowerKey(2)                     // 1  (< 2)
tmap.higherKey(2)                    // 3  (> 2)
tmap.headMap(3)                      // keys < 3
tmap.tailMap(2)                      // keys ≥ 2
tmap.subMap(1, 3)                    // keys in [1, 3)

// Descending order
TreeMap<Integer,String> desc = new TreeMap<>(Collections.reverseOrder());

// Traverse in sorted key order
for (Map.Entry<Integer, String> e : tmap.entrySet()) {
    System.out.println(e.getKey() + " → " + e.getValue());
}
```

---

## 8. HashSet

### Declaration & Insert

```java
import java.util.HashSet;
import java.util.Set;

Set<Integer> set = new HashSet<>();
Set<String>  words = new HashSet<>();

set.add(1);
set.add(2);
set.add(1);              // duplicate — silently ignored
set.addAll(Arrays.asList(3, 4, 5));
```

### Access & Remove

```java
set.contains(1)          // true
set.remove(1)            // remove by value
set.size()
set.isEmpty()
set.clear()
```

### Traversal

```java
Set<Integer> set = new HashSet<>(Arrays.asList(1, 2, 3));

for (int x : set) System.out.println(x);   // unordered

// Convert to sorted list
List<Integer> sorted = new ArrayList<>(set);
Collections.sort(sorted);
```

### Set operations

```java
Set<Integer> a = new HashSet<>(Arrays.asList(1, 2, 3));
Set<Integer> b = new HashSet<>(Arrays.asList(2, 3, 4));

// Union
Set<Integer> union = new HashSet<>(a);
union.addAll(b);

// Intersection
Set<Integer> inter = new HashSet<>(a);
inter.retainAll(b);

// Difference (a - b)
Set<Integer> diff = new HashSet<>(a);
diff.removeAll(b);
```

---

## 9. LinkedHashSet — insertion-order set

```java
import java.util.LinkedHashSet;

Set<Integer> set = new LinkedHashSet<>();
// Same API as HashSet
// Iteration order = insertion order (useful to deduplicate while preserving order)
```

---

## 10. TreeSet — sorted set

```java
import java.util.TreeSet;

TreeSet<Integer> ts = new TreeSet<>();

ts.add(5); ts.add(1); ts.add(3);

ts.first()                           // 1
ts.last()                            // 5
ts.floor(3)                          // 3  (≤ 3)
ts.ceiling(3)                        // 3  (≥ 3)
ts.lower(3)                          // 1  (< 3)
ts.higher(3)                         // 5  (> 3)
ts.headSet(3)                        // {1} — elements < 3
ts.tailSet(3)                        // {3, 5} — elements ≥ 3
ts.subSet(1, 4)                      // {1, 3} — [1, 4)
ts.pollFirst()                       // remove & return smallest
ts.pollLast()                        // remove & return largest

// Descending order
TreeSet<Integer> desc = new TreeSet<>(Collections.reverseOrder());

// Traversal (always sorted)
for (int x : ts) System.out.println(x);
```

---

## 11. Stack

```java
import java.util.Stack;
import java.util.Deque;
import java.util.ArrayDeque;

// Old way — Stack class (slower, synchronized)
Stack<Integer> stack = new Stack<>();
stack.push(1);
stack.push(2);
stack.peek();            // top element, no remove
stack.pop();             // remove & return top
stack.isEmpty();
stack.size();

// ✅ Preferred: ArrayDeque as Stack (faster)
Deque<Integer> stack2 = new ArrayDeque<>();
stack2.push(1);          // addFirst
stack2.push(2);
stack2.peek();           // peekFirst
stack2.pop();            // removeFirst
stack2.isEmpty();
```

---

## 12. Queue

```java
import java.util.Queue;
import java.util.LinkedList;
import java.util.ArrayDeque;

// LinkedList as Queue
Queue<Integer> q = new LinkedList<>();
q.offer(1);              // enqueue (add to back)
q.offer(2);
q.peek();                // front element, no remove — null if empty
q.poll();                // dequeue (remove from front) — null if empty
q.isEmpty();
q.size();

// ✅ Preferred: ArrayDeque as Queue (faster)
Queue<Integer> q2 = new ArrayDeque<>();
q2.offer(1);
q2.peek();
q2.poll();

// BFS pattern
Queue<Integer> bfs = new ArrayDeque<>();
bfs.offer(start);
while (!bfs.isEmpty()) {
    int node = bfs.poll();
    for (int neighbour : graph.get(node)) {
        if (!visited[neighbour]) {
            visited[neighbour] = true;
            bfs.offer(neighbour);
        }
    }
}
```

---

## 13. Deque (Double-ended queue)

```java
import java.util.Deque;
import java.util.ArrayDeque;

Deque<Integer> dq = new ArrayDeque<>();

// Front operations
dq.addFirst(1);          // offerFirst
dq.peekFirst();
dq.pollFirst();          // removeFirst

// Back operations
dq.addLast(2);           // offerLast
dq.peekLast();
dq.pollLast();           // removeLast

dq.isEmpty();
dq.size();

// Monotonic deque pattern (sliding window maximum)
Deque<Integer> mono = new ArrayDeque<>();
for (int i = 0; i < nums.length; i++) {
    while (!mono.isEmpty() && nums[mono.peekLast()] < nums[i])
        mono.pollLast();
    mono.offerLast(i);
    if (mono.peekFirst() < i - k + 1) mono.pollFirst();
}
```

---

## 14. PriorityQueue (Heap)

```java
import java.util.PriorityQueue;

// Min-heap (default) — smallest element at top
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5);
minHeap.offer(1);
minHeap.offer(3);
minHeap.peek();          // 1  (smallest)
minHeap.poll();          // remove & return 1

// Max-heap — largest element at top
PriorityQueue<Integer> maxHeap = new PriorityQueue<>((a, b) -> b - a);
// or: new PriorityQueue<>(Collections.reverseOrder())

// Custom object heap — sort by second element
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
pq.offer(new int[]{1, 5});
pq.offer(new int[]{2, 3});
pq.poll();               // {2, 3} — smallest second element

// K largest elements pattern
PriorityQueue<Integer> kLargest = new PriorityQueue<>();  // min-heap
for (int num : nums) {
    kLargest.offer(num);
    if (kLargest.size() > k) kLargest.poll();
}
// kLargest now holds k largest elements

minHeap.isEmpty();
minHeap.size();
```

---

## 15. Useful Utility Methods

### Math

```java
Math.max(a, b)
Math.min(a, b)
Math.abs(x)
Math.pow(2, 10)           // 1024.0 (double)
Math.sqrt(16)             // 4.0
Math.floor(3.7)           // 3.0
Math.ceil(3.2)            // 4.0
Math.log(x)               // natural log
Math.log(x) / Math.log(2) // log base 2
Integer.MAX_VALUE         // 2^31 - 1 = 2147483647
Integer.MIN_VALUE         // -2^31
Long.MAX_VALUE
Long.MIN_VALUE
```

### Character

```java
Character.isLetter(c)
Character.isDigit(c)
Character.isLetterOrDigit(c)
Character.isUpperCase(c)
Character.isLowerCase(c)
Character.toUpperCase(c)
Character.toLowerCase(c)
c - '0'                    // char digit → int
c - 'a'                    // char letter → 0-based index
(char)('a' + i)            // int → lowercase letter
```

### Integer / Bit operations

```java
Integer.toBinaryString(n)            // "1101"
Integer.toHexString(n)               // "d"
Integer.bitCount(n)                  // count set bits (1s)
Integer.highestOneBit(n)
Integer.numberOfLeadingZeros(n)
Integer.numberOfTrailingZeros(n)
Integer.reverse(n)                   // reverse bits

// Bit tricks
n & 1                                // check odd/even (1=odd)
n >> 1                               // divide by 2
n << 1                               // multiply by 2
n & (n - 1)                          // clear lowest set bit
n & (-n)                             // isolate lowest set bit
n ^ n                                // 0
a ^ b ^ a                            // b  (XOR trick)
```

---

## 16. Common CP Patterns — Java Syntax

### Frequency Map (character count)

```java
String s = "abracadabra";
Map<Character, Integer> freq = new HashMap<>();
for (char c : s.toCharArray()) {
    freq.merge(c, 1, Integer::sum);
    // same as: freq.put(c, freq.getOrDefault(c, 0) + 1);
}
```

### Two Pointer / Sliding Window

```java
int left = 0, right = 0;
int[] arr = {1, 2, 3, 4, 5};

while (right < arr.length) {
    // expand window
    right++;

    while (/* condition violated */) {
        // shrink window
        left++;
    }
}
```

### Prefix Sum

```java
int[] nums = {1, 2, 3, 4, 5};
int n = nums.length;
int[] prefix = new int[n + 1];
for (int i = 0; i < n; i++) {
    prefix[i + 1] = prefix[i] + nums[i];
}
// sum from index l to r (inclusive)
int rangeSum = prefix[r + 1] - prefix[l];
```

### Binary Search

```java
// Standard binary search on sorted array
int left = 0, right = arr.length - 1;
while (left <= right) {
    int mid = left + (right - left) / 2;   // avoids overflow
    if (arr[mid] == target) return mid;
    else if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
}

// Search on answer space
int lo = 0, hi = 1_000_000;
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (feasible(mid)) hi = mid;
    else lo = mid + 1;
}
```

### DFS — recursive

```java
boolean[] visited = new boolean[n];

void dfs(int node, List<List<Integer>> graph) {
    visited[node] = true;
    for (int neighbour : graph.get(node)) {
        if (!visited[neighbour]) {
            dfs(neighbour, graph);
        }
    }
}
```

### Sorting custom objects

```java
// Sort by length then alphabetically
String[] words = {"banana", "apple", "fig"};
Arrays.sort(words, (a, b) -> {
    if (a.length() != b.length()) return a.length() - b.length();
    return a.compareTo(b);
});

// Sort int[][] by first column
int[][] intervals = {{3,4},{1,2}};
Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
```

### Fast I/O (for competitive programming)

```java
import java.io.*;
import java.util.*;

BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
int n = Integer.parseInt(br.readLine().trim());
StringTokenizer st = new StringTokenizer(br.readLine());
int a = Integer.parseInt(st.nextToken());
int b = Integer.parseInt(st.nextToken());

// Fast output
StringBuilder out = new StringBuilder();
out.append(result).append('\n');
System.out.print(out);
```

---

## Quick Reference — Which DS to Use

| Need | Use |
|---|---|
| Fast random access | `int[]` / `ArrayList` |
| Ordered, unique elements | `TreeSet` |
| Unordered, unique elements | `HashSet` |
| Ordered key-value pairs | `TreeMap` |
| Unordered key-value pairs (fast) | `HashMap` |
| Insertion-order key-value | `LinkedHashMap` |
| LIFO (stack) | `ArrayDeque` as stack |
| FIFO (queue / BFS) | `ArrayDeque` as queue |
| Double-ended / monotonic queue | `ArrayDeque` |
| Min/max in O(log n) | `PriorityQueue` |
| Mutable string building | `StringBuilder` |
| Sorted map with range queries | `TreeMap` |
| Sorted set with range queries | `TreeSet` |

---

## One-Liners to Memorise

- **String compare**: always `.equals()`, never `==`
- **char → int**: `c - '0'` for digits, `c - 'a'` for letters
- **Frequency map**: `map.merge(key, 1, Integer::sum)`
- **Default value**: `map.getOrDefault(key, 0)`
- **Init adjacency list**: `graph.computeIfAbsent(node, k -> new ArrayList<>()).add(nb)`
- **Binary search midpoint**: `mid = left + (right - left) / 2` (no overflow)
- **Min-heap**: `new PriorityQueue<>()` — Max-heap: `new PriorityQueue<>((a,b)->b-a)`
- **Sort descending**: `Arrays.sort(boxed, (a,b)->b-a)` — needs `Integer[]` not `int[]`
- **Stack (fast)**: `Deque<Integer> stack = new ArrayDeque<>()`
- **Queue (fast)**: `Queue<Integer> q = new ArrayDeque<>()`
- **Set ops**: `retainAll` = intersection, `addAll` = union, `removeAll` = difference
- **Bit count**: `Integer.bitCount(n)` — isolate lowest bit: `n & (-n)`
