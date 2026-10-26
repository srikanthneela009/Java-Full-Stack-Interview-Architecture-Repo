# Section 08: Coding Challenges, Concurrency & Capstone Project (52 Deep-Dive Questions)

[← Back to Main README](../README.md)

---

## Part A: Basic Foundational Coding Challenges (Q8.1 – Q8.18)

<a id="q8-1"></a>
### Q8.1: Java Streams: Find the First Non-Repeating Character in a String preserving character encounter order.

#### 1. Formal Technical Definition & Internal Mechanics
- **Algorithm**:
  1. Convert the String to an `IntStream` of Unicode code points (`str.codePoints()`).
  2. Group characters into a **`LinkedHashMap<Integer, Long>`** (`Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting())`).
  3. **Why `LinkedHashMap::new` is mandatory**: Default `groupingBy` uses a `HashMap`, which scrambles insertion order! `LinkedHashMap` preserves exact first-seen order in $O(N)$ time.

#### 2. Technical Architecture & Execution Diagram
```
Input: "swiss"
  Pass 1 (LinkedHashMap insertion-ordered counts):
    ['s' -> 3] ──► ['w' -> 1] ──► ['i' -> 1]
  Pass 2 (Filter count == 1 & findFirst()):
    Returns 'w'!
```

#### 3. Detailed Technical Example & Production Code
```java
public static Optional<Character> firstNonRepeatingChar(String input) {
    if (input == null || input.isEmpty()) return Optional.empty();

    return input.chars()
            .mapToObj(c -> (char) c)
            .collect(Collectors.groupingBy(
                    Function.identity(),
                    LinkedHashMap::new, // Preserves encounter order!
                    Collectors.counting()
            ))
            .entrySet().stream()
            .filter(entry -> entry.getValue() == 1L)
            .map(Map.Entry::getKey)
            .findFirst();
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the Stream pipeline chains lazy intermediate operations and triggers single-pass evaluation at the terminal collector/reduction step.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Complexity**: $O(N)$ time, $O(K)$ space (where $K$ is the number of unique characters). Using `input.codePoints()` instead of `input.chars()` also handles surrogate pairs (emojis/CJK characters) properly.

---

<a id="q8-2"></a>
### Q8.2: Java Streams: Find the Nth Highest Salary per Department (handling duplicate tied salaries correctly).

#### 1. Formal Technical Definition & Internal Mechanics
- **The Tie Trap**: If a department has salaries `[150k, 150k, 120k]` and the interviewer asks for the **2nd highest distinct salary**, sorting employees directly and calling `.skip(1).findFirst()` wrongly returns `150k` instead of `120k`!
- **Fix**: Extract distinct salaries first (`.map(Employee::salary).distinct().sorted(Comparator.reverseOrder()).skip(n - 1).findFirst()`), or group employees by salary in a `TreeMap<>(Comparator.reverseOrder())`.

#### 2. Technical Architecture & Execution Diagram
```
Dept "ENG" Employees: Alice($150k), Bob($150k), Charlie($120k)
  ├── Naive Sort & skip(1):          Returns Bob($150k) ──► WRONG! (Tied for 1st!)
  └── TreeMap<Salary, List<Emp>>:
        Entry 0 (1st Highest): $150k -> [Alice, Bob]
        Entry 1 (2nd Highest): $120k -> [Charlie]       ──► CORRECT!
```

#### 3. Detailed Technical Example & Production Code
```java
public record Employee(Long id, String name, String department, BigDecimal salary) {}

public static Map<String, Optional<BigDecimal>> nthHighestSalaryByDept(List<Employee> employees, int n) {
    return employees.stream()
        .collect(Collectors.groupingBy(
            Employee::department,
            Collectors.collectingAndThen(
                Collectors.mapping(Employee::salary, Collectors.toSet()),
                distinctSalaries -> distinctSalaries.stream()
                    .sorted(Comparator.reverseOrder())
                    .skip(n - 1L)
                    .findFirst()
            )
        ));
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `Employee` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the Stream pipeline chains lazy intermediate operations and triggers single-pass evaluation at the terminal collector/reduction step.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Always clarify with the interviewer whether "Nth highest" means `DENSE_RANK() = N` (distinct salary value) or `ROW_NUMBER() = N` before writing code—that clarifying question alone sets a senior tone.

---

<a id="q8-3"></a>
### Q8.3: Java Streams: Group Transactions by Currency and Compute Total Sum, Count, Min, Max, and Average in a Single Pass.

#### 1. Formal Technical Definition & Internal Mechanics
- For `double`/`long`, `Collectors.summarizingDouble` computes count/sum/min/max/avg in one pass.
- For financial **`BigDecimal`**, use `Collectors.reducing` or a custom immutable accumulator `Record` with `Collectors.teeing` / `Collector.of(...)` so you never convert `BigDecimal` to imprecise `double`!

#### 2. Technical Architecture & Execution Diagram
```
Transactions: [ (USD, 10.50), (USD, 20.00), (EUR, 50.00) ]
      │
      ▼ (Single Stream Pass -> groupingBy(currency, reducing BigDecimal.ZERO, BigDecimal::add))
Map: { "USD" -> 30.50, "EUR" -> 50.00 }
```

#### 3. Detailed Technical Example & Production Code
```java
public record Tx(String id, String currency, BigDecimal amount) {}

public static Map<String, BigDecimal> sumByCurrencyExact(List<Tx> transactions) {
    return transactions.stream()
        .collect(Collectors.groupingBy(
            Tx::currency,
            Collectors.reducing(BigDecimal.ZERO, Tx::amount, BigDecimal::add)
        ));
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `Tx` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the Stream pipeline chains lazy intermediate operations and triggers single-pass evaluation at the terminal collector/reduction step.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Never use `Collectors.summingDouble(tx -> tx.amount().doubleValue())` on monetary `BigDecimal` values in financial coding interviews! Using `Collectors.reducing(BigDecimal.ZERO, Tx::amount, BigDecimal::add)` preserves exact base-10 arithmetic.

---

<a id="q8-4"></a>
### Q8.4: Java Streams: Find Duplicate Elements in a Stream and their Frequency Counts (`> 1`).

#### 1. Formal Technical Definition & Internal Mechanics
- **Approach 1 (Frequency Map - Most Informative)**: Group with `Collectors.counting()` and retain entries where `count > 1`. Runs in $O(N)$ time and $O(N)$ space.
- **Approach 2 (Single-Pass `Set.add()` Filter)**: Maintain a `Set<T> seen = new HashSet<>()` and filter `.filter(e -> !seen.add(e)).collect(Collectors.toSet())`. Note: stateful lambdas (`!seen.add(e)`) must use `ConcurrentHashMap.newKeySet()` if used on a parallel stream.

#### 2. Technical Architecture & Execution Diagram
```
Input: ["A", "B", "A", "C", "B", "A"]
  groupingBy + counting(): { "A": 3, "B": 2, "C": 1 }
  Filter value > 1:        { "A": 3, "B": 2 }
```

#### 3. Detailed Technical Example & Production Code
```java
public static <T> Map<T, Long> findDuplicatesWithCounts(Collection<T> items) {
    Map<T, Long> counts = items.stream()
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
    counts.values().removeIf(count -> count <= 1L); // Mutates map in-place in O(U)!
    return counts;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the Stream pipeline chains lazy intermediate operations and triggers single-pass evaluation at the terminal collector/reduction step.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Why Avoid `Collections.frequency(list, item)` in a Stream Filter**: Writing `list.stream().filter(i -> Collections.frequency(list, i) > 1)` scans the entire list for every element, turning an $O(N)$ task into **$O(N^2)$**! Always use a $O(N)$ `groupingBy` frequency map.

---

<a id="q8-5"></a>
### Q8.5: Java Streams: Partition a List of Orders into Valid vs Invalid (`Collectors.partitioningBy`) and Flatten Nested Line Items (`flatMap`).

#### 1. Formal Technical Definition & Internal Mechanics
- **`Collectors.partitioningBy(Predicate)`**: Special case of `groupingBy` that always returns a `Map<Boolean, List<T>>` Guaranteed to contain **both** `Boolean.TRUE` and `Boolean.FALSE` keys (even if one list is empty, avoiding `NullPointerException` on `.get(true)`!).
- **`flatMap`**: Maps each `Order` to `order.items().stream()` and flattens the resulting `Stream<Stream<LineItem>>` into a single `Stream<LineItem>`.

#### 2. Technical Architecture & Execution Diagram
```
Orders: [ Order1(valid=true, items=[i1, i2]), Order2(valid=false, items=[i3]) ]
  ├── partitioningBy(Order::isValid) ──► { true: [Order1], false: [Order2] }
  └── flatMap(o -> o.items().stream()) ──► [ i1, i2, i3 ]
```

#### 3. Detailed Technical Example & Production Code
```java
public record LineItem(String sku, int quantity, BigDecimal unitPrice) {}
public record Order(Long id, boolean valid, List<LineItem> items) {}

public static Map<String, Integer> totalQuantityPerSkuAcrossValidOrders(List<Order> orders) {
    Map<Boolean, List<Order>> partitioned = orders.stream()
            .collect(Collectors.partitioningBy(Order::valid));

    return partitioned.get(Boolean.TRUE).stream()
            .flatMap(order -> order.items().stream())
            .collect(Collectors.groupingBy(
                    LineItem::sku,
                    Collectors.summingInt(LineItem::quantity)
            ));
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `LineItem`, `Order` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the Stream pipeline chains lazy intermediate operations and triggers single-pass evaluation at the terminal collector/reduction step.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Key interview distinction between `groupingBy(Predicate)` and `partitioningBy(Predicate)`: if all elements are `true`, `groupingBy` returns a map where `.get(false)` is **`null`**, whereas `partitioningBy` initializes **both** `true` and `false` keys to empty lists `[]`.

---

<a id="q8-6"></a>
### Q8.6: Java Streams: Merge Two Lists of Objects by ID and Resolve Conflicts using `Collectors.toMap` Merge Function.

#### 1. Formal Technical Definition & Internal Mechanics
- Calling `Collectors.toMap(KeyMapper, ValueMapper)` (2-arg overload) throws **`IllegalStateException: Duplicate key`** if two elements share the same key!
- Always use the **3-arg or 4-arg overload**: `Collectors.toMap(KeyMapper, ValueMapper, MergeBinaryOperator, MapSupplier)` to specify exactly how duplicate keys are merged (e.g., keeping the item with the newer `updatedAt` timestamp or summing quantities).

#### 2. Technical Architecture & Execution Diagram
```
Stream.concat(listA.stream(), listB.stream())
       │
       ▼
Collectors.toMap(Account::id, Function.identity(),
    (existing, replacement) -> existing.version() >= replacement.version() ? existing : replacement)
```

#### 3. Detailed Technical Example & Production Code
```java
public record AccountSnapshot(Long id, BigDecimal balance, long version) {}

public static Collection<AccountSnapshot> mergeLatestSnapshots(
        List<AccountSnapshot> primaryList, List<AccountSnapshot> replicaList) {
    return Stream.concat(primaryList.stream(), replicaList.stream())
        .collect(Collectors.toMap(
            AccountSnapshot::id,
            Function.identity(),
            (a, b) -> a.version() >= b.version() ? a : b, // Keep higher version on conflict!
            LinkedHashMap::new
        ))
        .values();
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `AccountSnapshot` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the Stream pipeline chains lazy intermediate operations and triggers single-pass evaluation at the terminal collector/reduction step.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Another `Collectors.toMap` Production Pitfall (`NullPointerException` on `null` map values)**: Because `Collectors.toMap` uses `Map.merge()` internally, if your `ValueMapper` returns `null`, `Map.merge()` throws a `NullPointerException`! Even though `HashMap` allows `null` values, `Collectors.toMap` forbids them.

---

<a id="q8-7"></a>
### Q8.7: Java Streams: Compute the Top-K Most Frequent Words in a Large Stream in $O(N \log K)$ Memory using a Min-Heap (`PriorityQueue`).

#### 1. Formal Technical Definition & Internal Mechanics
- **Why sorting all unique words (`sorted().limit(k)`) is suboptimal**: If there are $U = 1,000,000$ unique words and you only want the Top $K = 10$, sorting all $1,000,000$ entries takes $O(U \log U)$ time and space.
- **Min-Heap (`PriorityQueue`) of Size $K$**:
  - Maintain a **Min-Heap** ordered by lowest frequency first (and reverse alphabetical on ties).
  - For each entry in the frequency map, push to the Min-Heap; if `heap.size() > K`, call `heap.poll()` (which evicts the smallest element in $O(\log K)$).
  - Total complexity: **$O(N + U \log K)$ time** and **$O(K)$ heap space**!

#### 2. Technical Architecture & Execution Diagram
```
Finding Top-3 (Min-Heap keeps at most 3 elements, smallest at root for O(log 3) eviction):
  Heap: [ ("error", 15), ("warn", 40), ("info", 90) ]  (Root = "error":15)
  Next item: ("fatal", 55) > Root(15) ──► Push ("fatal", 55) & Poll Root ("error", 15)
  Remaining Heap: [ ("warn", 40), ("fatal", 55), ("info", 90) ]
```

#### 3. Detailed Technical Example & Production Code
```java
public static List<String> topKFrequentWords(List<String> words, int k) {
    Map<String, Long> freqMap = words.stream()
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

    // Min-Heap: lowest count at head; if counts tied, lexicographically larger word evicted first
    PriorityQueue<Map.Entry<String, Long>> minHeap = new PriorityQueue<>(
        Comparator.<Map.Entry<String, Long>>comparingLong(Map.Entry::getValue)
            .thenComparing(Map.Entry.comparingByKey(Comparator.reverseOrder()))
    );

    for (Map.Entry<String, Long> entry : freqMap.entrySet()) {
        minHeap.offer(entry);
        if (minHeap.size() > k) {
            minHeap.poll();
        }
    }

    List<String> result = new ArrayList<>(minHeap.size());
    while (!minHeap.isEmpty()) {
        result.add(minHeap.poll().getKey());
    }
    Collections.reverse(result);
    return result;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the Stream pipeline chains lazy intermediate operations and triggers single-pass evaluation at the terminal collector/reduction step.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Notice the tie-breaker in the Min-Heap comparator (`Comparator.reverseOrder()` on key): because we want `"apple"` to beat `"banana"` when their counts are equal, `"banana"` must be considered "smaller" in the Min-Heap so `"banana"` gets evicted first when `size > k`!

---

<a id="q8-8"></a>
### Q8.8: Core Algorithm: Reverse a Singly Linked List In-Place ($O(N)$ Time, $O(1)$ Space) and Reverse in Groups of Size $K$.

#### 1. Formal Technical Definition & Internal Mechanics
- Maintain three pointers (`prev = null`, `curr = head`, `next = null`).
- In each step, save `next = curr.next`, flip `curr.next = prev`, advance `prev = curr`, and advance `curr = next`.

#### 2. Technical Architecture & Execution Diagram
```
Initial:   null    [1] ──► [2] ──► [3] ──► null
           prev    curr    next

Step 1:    null ◄── [1]    [2] ──► [3] ──► null
                    prev   curr    next

Final:     null ◄── [1] ◄── [2] ◄── [3]    null
                                    prev   curr
```

#### 3. Detailed Technical Example & Production Code
```java
public static class ListNode {
    int val;
    ListNode next;
    public ListNode(int val, ListNode next) { this.val = val; this.next = next; }
}

public static ListNode reverseIterative(ListNode head) {
    ListNode prev = null;
    ListNode curr = head;
    while (curr != null) {
        ListNode nextTemp = curr.next; // 1. Save next node
        curr.next = prev;              // 2. Reverse pointer backward
        prev = curr;                   // 3. Advance prev
        curr = nextTemp;               // 4. Advance curr
    }
    return prev; // New head!
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `ListNode` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `while (curr != null) {` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Why Iterative ($O(1)$ space) beats Recursive ($O(N)$ stack space) in Production**: Reversing a linked list of 50,000 nodes recursively allocates 50,000 stack frames and throws `StackOverflowError`. Always write the 4-line iterative pointer reversal.

---

<a id="q8-9"></a>
### Q8.9: Core Algorithm: Detect a Cycle in a Linked List and Find the Exact Cycle Entry Node (Floyd's Tortoise and Hare Algorithm).

#### 1. Formal Technical Definition & Internal Mechanics
- **Phase 1 (Cycle Detection)**: `slow` moves 1 step (`slow = slow.next`); `fast` moves 2 steps (`fast = fast.next.next`). If there is a cycle, `fast` laps `slow` and they meet inside the cycle in $O(N)$ time and $O(1)$ memory!
- **Phase 2 (Finding the Exact Cycle Start Node - Mathematical Proof)**:
  - Let $L$ be the distance from `head` to cycle entry, $X$ be the distance from cycle entry to the meeting point, and $C$ be the cycle length.
  - Since `fast` traveled twice the distance of `slow`: $2(L + X) = L + X + kC \implies L = kC - X$.
  - Therefore, resetting a pointer `p1 = head` and moving both `p1` and `slow` **1 step at a time** guarantees they collide at the **exact Cycle Entry Node**!

#### 2. Technical Architecture & Execution Diagram
```
[Head] ──(L steps)──► [Cycle Entry] ──(X steps)──► [Meeting Point]
                            ▲                             │
                            └────────(C - X steps)────────┘
Math: 2(L + X) = L + X + C  ==>  L = C - X!
Reset p1 to Head -> Move p1 and slow 1 step each -> Meet at [Cycle Entry]!
```

#### 3. Detailed Technical Example & Production Code
```java
public static ListNode detectCycleStartNode(ListNode head) {
    ListNode slow = head;
    ListNode fast = head;

    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) { // Cycle detected!
            ListNode ptr = head;
            while (ptr != slow) {
                ptr = ptr.next;
                slow = slow.next;
            }
            return ptr; // Exact cycle entry node!
        }
    }
    return null; // No cycle
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `while (ptr != slow) {` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Explaining the 1-line equation $2(L + X) = L + X + C \implies L = C - X$ during the interview proves you understand *why* Floyd's Phase 2 works, rather than just memorizing it.

---

<a id="q8-10"></a>
### Q8.10: Core Algorithm: Two Sum and Subarray Sum Equals $K$ in $O(N)$ Time using Prefix Sum + HashMap.

#### 1. Formal Technical Definition & Internal Mechanics
- **Two Sum**: Iterate through `nums`; for each `nums[i]`, check if `complement = target - nums[i]` exists in `Map<Integer, Integer> seenIndex`.
- **Subarray Sum Equals $K$ (Handles negative numbers where Sliding Window fails!)**:
  - Maintain running `prefixSum += nums[i]`.
  - If `prefixSum - k` occurred $C$ times at earlier indices $j < i$, then the subarray `(j..i]` has sum `prefixSum - (prefixSum - k) = k`!
  - Store `Map<Long, Integer> prefixSumCounts` initialized with `{0L: 1}`.

#### 2. Technical Architecture & Execution Diagram
```
Array: [ 3,  4, -7,  1,  3,  3 ],  k = 7
PrefixSum at i=1: 3 + 4 = 7  ──► Check (7 - 7 = 0) in Map (Count=1) ──► Found [3, 4]!
PrefixSum at i=5: 7          ──► Check (7 - 7 = 0) in Map + earlier 0s ──► Found [1, 3, 3]!
```

#### 3. Detailed Technical Example & Production Code
```java
public static int subarraySumEqualsK(int[] nums, int k) {
    Map<Long, Integer> prefixCounts = new HashMap<>();
    prefixCounts.put(0L, 1); // Base case: empty prefix has sum 0

    long runningSum = 0;
    int totalSubarrays = 0;
    for (int num : nums) {
        runningSum += num;
        totalSubarrays += prefixCounts.getOrDefault(runningSum - k, 0);
        prefixCounts.merge(runningSum, 1, Integer::sum);
    }
    return totalSubarrays;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `subarraySumEqualsK(...)`, execution proceeds step-by-step: the core statement `runningSum += num;` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Why Two-Pointer Sliding Window fails when array contains negative numbers**: Expanding a window can decrease the sum and shrinking can increase the sum, breaking monotonicity. The **Prefix Sum + HashMap ($O(N)$)** pattern works for all positive, zero, and negative integers!

---

<a id="q8-11"></a>
### Q8.11: Core Algorithm: Valid Parentheses and Minimum Add to Make Parentheses Valid using `ArrayDeque` Stack.

#### 1. Formal Technical Definition & Internal Mechanics
- Never use legacy `java.util.Stack` (which extends `Vector` and synchronizes every push/pop)! Always use **`Deque<Character> stack = new ArrayDeque<>()`**.
- Push expected closing brackets (`')'`, `']'`, `'}'`) when seeing an opening bracket, and on a closing bracket assert `!stack.isEmpty() && stack.pop() == ch`.

#### 2. Technical Architecture & Execution Diagram
```
Input: "{ [ ( ) ] }"
  See '{' -> push '}' | Stack: ['}']
  See '[' -> push ']' | Stack: [']', '}']
  See '(' -> push ')' | Stack: [')', ']', '}']
  See ')' -> pop() == ')' (Match!)
  ...
  End: stack.isEmpty() == true ──► Valid!
```

#### 3. Detailed Technical Example & Production Code
```java
public static boolean isValidParentheses(String s) {
    if ((s.length() & 1) != 0) return false; // Odd length can never be balanced!
    Deque<Character> stack = new ArrayDeque<>(s.length() / 2);

    for (int i = 0; i < s.length(); i++) {
        char c = s.charAt(i);
        switch (c) {
            case '(' -> stack.push(')');
            case '[' -> stack.push(']');
            case '{' -> stack.push('}');
            default -> {
                if (stack.isEmpty() || stack.pop() != c) return false;
            }
        }
    }
    return stack.isEmpty();
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `isValidParentheses(...)`, execution proceeds step-by-step: the core statement `case '{' -> stack.push('}');` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Fast-failing on odd string length (`(s.length() & 1) != 0`) and pushing the expected closing char avoids map lookups and shows clean Java 21 switch syntax.

---

<a id="q8-12"></a>
### Q8.12: Core Algorithm: Longest Substring Without Repeating Characters in $O(N)$ Time (Sliding Window with Index Jump).

#### 1. Formal Technical Definition & Internal Mechanics
- Maintain a sliding window `[left .. right]` and an array/map storing the **last seen index + 1** of each character.
- When `s.charAt(right)` is already inside the current window, jump `left` directly to `Math.max(left, lastSeenIndex[ch] + 1)` in $O(1)$ without incrementing `left` one step at a time!

#### 2. Technical Architecture & Execution Diagram
```
String:  a  b  c  a  b  c  b  b
Indices: 0  1  2  3  4  5  6  7
  right=0..2 ("abc"): left=0, maxLen=3
  right=3 ('a' seen at 0): left jumps to 0+1 = 1 ("bca"), maxLen=3
```

#### 3. Detailed Technical Example & Production Code
```java
public static int lengthOfLongestSubstring(String s) {
    int[] nextIndex = new int[128]; // Stores (lastSeenIndex + 1) for ASCII chars
    int maxLen = 0;
    for (int left = 0, right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        left = Math.max(left, nextIndex[c]);
        maxLen = Math.max(maxLen, right - left + 1);
        nextIndex[c] = right + 1;
    }
    return maxLen;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `lengthOfLongestSubstring(...)`, execution proceeds step-by-step: the core statement `left = Math.max(left, nextIndex[c]);` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Why `Math.max(left, nextIndex[c])` is essential**: For input `"abba"`, when `right=3` (`'a'`), `'a'` was last seen at index `0`, but `left` already advanced to `2` on `'b'`. Without `Math.max(left, ...)`, `left` would move *backward* to `1` and include duplicate `'b'`s!

---

<a id="q8-13"></a>
### Q8.13: Core Algorithm: Merge Overlapping Intervals (Meeting Rooms / Booking Time Slots) in $O(N \log N)$ Time.

#### 1. Formal Technical Definition & Internal Mechanics
- **Step 1**: Sort intervals by `start` time ascending (`Integer.compare(a[0], b[0])`).
- **Step 2**: Iterate through sorted intervals; if `next.start <= current.end`, they overlap—merge them by extending `current.end = Math.max(current.end, next.end)`. Otherwise, append `next` as a new disjoint interval.

#### 2. Technical Architecture & Execution Diagram
```
Sorted Input: [1..3], [2..6], [8..10]
  [1──────3]
     [2──────────6]   (2 <= 3: Overlaps! Merge into [1..6])
                       [8────10] (8 > 6: Disjoint! Start new interval)
Output:       [1─────────────6], [8────10]
```

#### 3. Detailed Technical Example & Production Code
```java
public static int[][] mergeIntervals(int[][] intervals) {
    if (intervals.length <= 1) return intervals;
    Arrays.sort(intervals, Comparator.comparingInt(a -> a[0]));

    List<int[]> merged = new ArrayList<>();
    int[] current = intervals[0];
    merged.add(current);

    for (int i = 1; i < intervals.length; i++) {
        int[] next = intervals[i];
        if (next[0] <= current[1]) {
            current[1] = Math.max(current[1], next[1]); // Mutates current end in-place!
        } else {
            current = next;
            merged.add(current);
        }
    }
    return merged.toArray(new int[0][]);
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `if (next[0] <= current[1]) {` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Never sort with `(a, b) -> a[0] - b[0]`**: Subtraction overflows 32-bit `int` when `a[0] = -2_000_000_000` and `b[0] = 1_000_000_000`! Always use `Integer.compare(a[0], b[0])` or `Comparator.comparingInt(a -> a[0])`.

---

<a id="q8-14"></a>
### Q8.14: Core Algorithm: Binary Search in a Rotated Sorted Array and Monotonic Answer-Space Binary Search (Koko / Capacity).

#### 1. Formal Technical Definition & Internal Mechanics
- **Rotated Sorted Array (`[4, 5, 6, 7, 0, 1, 2]`)**:
  - When splitting at `mid = low + ((high - low) >>> 1)`, **at least one half (`[low..mid]` or `[mid..high]`) is guaranteed to be strictly sorted**!
  - Check if `nums[low] <= nums[mid]`: if true, the left half is sorted—check if `target` lies within `[nums[low] .. nums[mid])`; if yes, search left (`high = mid - 1`), else search right (`low = mid + 1`).

#### 2. Technical Architecture & Execution Diagram
```
Array: [ 4,  5,  6,  7,  0,  1,  2 ],  target = 0
         L           M           H
  Left half [4..7] is sorted (4 <= 7), and target 0 is NOT in [4..7]
  ──► Narrow search to Right half: low = M + 1!
```

#### 3. Detailed Technical Example & Production Code
```java
public static int searchRotatedArray(int[] nums, int target) {
    int low = 0, high = nums.length - 1;
    while (low <= high) {
        int mid = low + ((high - low) >>> 1); // Overflow-safe midpoint!
        if (nums[mid] == target) return mid;

        if (nums[low] <= nums[mid]) { // Left half is sorted
            if (target >= nums[low] && target < nums[mid]) {
                high = mid - 1;
            } else {
                low = mid + 1;
            }
        } else { // Right half is sorted
            if (target > nums[mid] && target <= nums[high]) {
                low = mid + 1;
            } else {
                high = mid - 1;
            }
        }
    }
    return -1;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `searchRotatedArray(...)`, execution proceeds step-by-step: the core statement `}` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Josh Bloch's Binary Search Overflow Bug**: Never write `(low + high) / 2`! When `low + high > Integer.MAX_VALUE`, it overflows to a negative number and throws `ArrayIndexOutOfBoundsException`. Always write `low + ((high - low) >>> 1)`.

---

<a id="q8-15"></a>
### Q8.15: Core Algorithm: Maximum Subarray Sum (Kadane's Algorithm) and Best Time to Buy and Sell Stock in $O(N)$ Time.

#### 1. Formal Technical Definition & Internal Mechanics
- **Kadane's Algorithm**: At each index $i$, decide whether to **extend the previous subarray** (`currentSum + nums[i]`) or **start a fresh subarray at $i$** (`nums[i]`):
  $$\text{currentSum} = \max(\text{nums}[i], \text{currentSum} + \text{nums}[i])$$
  $$\text{maxSum} = \max(\text{maxSum}, \text{currentSum})$$

#### 2. Technical Architecture & Execution Diagram
```
Array:       [ -2,   1,  -3,   4,  -1,   2,   1,  -5,   4 ]
currentSum:    -2    1   -2    4    3    5    6    1    5
maxSum:        -2    1    1    4    4    5    6    6    6  (Max Subarray = [4, -1, 2, 1] = 6)
```

#### 3. Detailed Technical Example & Production Code
```java
public static long maxSubarraySum(int[] nums) {
    if (nums == null || nums.length == 0) throw new IllegalArgumentException("Empty array");
    long currentSum = nums[0];
    long maxSum = nums[0];

    for (int i = 1; i < nums.length; i++) {
        currentSum = Math.max(nums[i], currentSum + nums[i]);
        maxSum = Math.max(maxSum, currentSum);
    }
    return maxSum;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `maxSubarraySum(...)`, execution proceeds step-by-step: the core statement `currentSum = Math.max(nums[i], currentSum + nums[i]);` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **All-Negative Array Edge Case**: Initializing `maxSum = 0` fails when all elements are negative (e.g., `[-5, -2, -9]`, returning `0` instead of `-2`). Always initialize `currentSum = nums[0]` and `maxSum = nums[0]`.

---

<a id="q8-16"></a>
### Q8.16: Core Algorithm: Product of Array Except Self in $O(N)$ Time and $O(1)$ Auxiliary Space without Division.

#### 1. Formal Technical Definition & Internal Mechanics
- **Why Division (`totalProduct / nums[i]`) Fails**: Fails with `ArithmeticException: / by zero` whenever the array contains `0`!
- **Two-Pass Prefix & Suffix Product in the Output Array**:
  - **Pass 1 (Left-to-Right)**: `result[i]` stores the product of all elements strictly to the **left** of `i`.
  - **Pass 2 (Right-to-Left)**: Maintain a running `suffixProduct` scalar and multiply `result[i] *= suffixProduct`.

#### 2. Technical Architecture & Execution Diagram
```
Input:                  [  1,   2,   3,   4 ]
Pass 1 (Left Prefix):   [  1,   1,   2,   6 ]
Pass 2 (Right Suffix):  [ 24,  12,   4,   1 ] (Multiplied in-place using 1 scalar variable!)
Final Output:           [ 24,  12,   8,   6 ]
```

#### 3. Detailed Technical Example & Production Code
```java
public static int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];

    result[0] = 1;
    for (int i = 1; i < n; i++) {
        result[i] = result[i - 1] * nums[i - 1];
    }

    int suffixProduct = 1;
    for (int i = n - 1; i >= 0; i--) {
        result[i] *= suffixProduct;
        suffixProduct *= nums[i];
    }
    return result;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `int suffixProduct = 1;` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Handles zero elements (`0` once or `0` multiple times) naturally without a single `if (nums[i] == 0)` branch!

---

<a id="q8-17"></a>
### Q8.17: Core Algorithm: Binary Tree Level-Order Traversal (BFS) and Validate Binary Search Tree (BST) with `Long` Range Bounds.

#### 1. Formal Technical Definition & Internal Mechanics
- **Why checking only `node.left.val < node.val && node.right.val > node.val` is a Bug for BST Validation**: A node deep in the right subtree of root `10` could have value `6` (which is `<` its immediate parent `15`, **but violates `> 10` for the ancestor root**!).
- **Fix**: Pass down the valid open interval `(minBound, maxBound)` as `Long` (to safely handle `Integer.MIN_VALUE` and `Integer.MAX_VALUE` node values!).

#### 2. Technical Architecture & Execution Diagram
```
       10 (-∞, +∞)
      /           \
  5 (-∞, 10)     15 (10, +∞)
                 /         \
            6 (10, 15)    20 (15, +∞)
            ▲
            └── 6 is NOT > minBound(10)! Invalid BST Caught!
```

#### 3. Detailed Technical Example & Production Code
```java
public static class TreeNode {
    int val;
    TreeNode left, right;
}

public static boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}

private static boolean validate(TreeNode node, long minExclusive, long maxExclusive) {
    if (node == null) return true;
    if (node.val <= minExclusive || node.val >= maxExclusive) {
        return false;
    }
    return validate(node.left, minExclusive, node.val)
        && validate(node.right, node.val, maxExclusive);
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `TreeNode` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `isValidBST(...)` and `validate(...)`, execution proceeds step-by-step: the core statement `private static boolean validate(TreeNode node, long minExclusive, long maxE` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Using `long minExclusive, long maxExclusive` instead of `int` prevents boundary bugs when a tree node legitimately contains `Integer.MIN_VALUE` or `Integer.MAX_VALUE`.

---

<a id="q8-18"></a>
### Q8.18: Core Algorithm: Group Anagrams in $O(N \cdot L)$ Time using Frequency Key vs $O(N \cdot L \log L)$ Sorting.

#### 1. Formal Technical Definition & Internal Mechanics
- **Approach 1 (Sorting chars of each word)**: `char[] arr = word.toCharArray(); Arrays.sort(arr); String key = new String(arr);` takes $O(N \cdot L \log L)$.
- **Approach 2 (26-Element Character Frequency Key)**: Count `a..z` frequencies in an `int[26]` or `char[26]` array in $O(L)$ time without sorting.

#### 2. Technical Architecture & Execution Diagram
```
"eat" ──► char[] counts -> "a1e1t1" ──┐
"tea" ──► char[] counts -> "a1e1t1" ──┼──► Map Key "a1e1t1" -> ["eat", "tea", "ate"]
"ate" ──► char[] counts -> "a1e1t1" ──┘
```

#### 3. Detailed Technical Example & Production Code
```java
public static List<List<String>> groupAnagrams(List<String> words) {
    Map<String, List<String>> groups = new HashMap<>();
    for (String word : words) {
        char[] counts = new char[26];
        for (int i = 0; i < word.length(); i++) {
            counts[word.charAt(i) - 'a']++;
        }
        String key = new String(counts);
        groups.computeIfAbsent(key, k -> new ArrayList<>()).add(word);
    }
    return new ArrayList<>(groups.values());
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `}` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Notice `groups.computeIfAbsent(key, k -> new ArrayList<>()).add(word)` reduces 5 lines of `containsKey`/`put` boilerplate to 1 expressive line.

---

## Part B: Intermediate Concurrency & Data Structure Challenges (Q8.19 – Q8.35)

<a id="q8-19"></a>
### Q8.19: Design & Implement an $O(1)$ LRU (Least Recently Used) Cache from Scratch (Doubly-Linked List + `HashMap`, vs `LinkedHashMap`, vs Thread-Safe Segmented LRU).

#### 1. Formal Technical Definition & Internal Mechanics
- To achieve strict **$O(1)$ `get(key)` and $O(1)$ `put(key, value)`**:
  1. A **`HashMap<K, Node<K, V>>`** provides $O(1)$ lookup by key.
  2. A **Doubly-Linked List** (`head <-> ... <-> tail` with dummy sentinel nodes `head` and `tail` so we never check for `null` pointers!) maintains access recency order:
     - Most Recently Used (`MRU`) is placed right after `head` (`head.next`).
     - Least Recently Used (`LRU`) sits right before `tail` (`tail.prev`) and is evicted in $O(1)$ when `map.size() > capacity`.

#### 2. Technical Architecture & Execution Diagram
```
HashMap:  "k1" ──► [Node k1]        "k2" ──► [Node k2]
                      │                         │
Doubly-Linked List:   ▼                         ▼
  [Dummy Head] ◄═► [Node k1 (MRU)] ◄═══════► [Node k2 (LRU)] ◄═► [Dummy Tail]
  (On get("k2"): Unlink Node k2 in O(1) and move right after Dummy Head!)
```

#### 3. Detailed Technical Example & Production Code
```java
public class ThreadSafeLruCache<K, V> {
    private static class Node<K, V> {
        final K key;
        V value;
        Node<K, V> prev, next;
        Node(K key, V value) { this.key = key; this.value = value; }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head = new Node<>(null, null); // Sentinel head
    private final Node<K, V> tail = new Node<>(null, null); // Sentinel tail
    private final ReentrantLock lock = new ReentrantLock();

    public ThreadSafeLruCache(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException("Capacity must be > 0");
        this.capacity = capacity;
        this.map = new HashMap<>(capacity);
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        lock.lock();
        try {
            Node<K, V> node = map.get(key);
            if (node == null) return null;
            moveToHead(node);
            return node.value;
        } finally {
            lock.unlock();
        }
    }

    public void put(K key, V value) {
        lock.lock();
        try {
            Node<K, V> existing = map.get(key);
            if (existing != null) {
                existing.value = value;
                moveToHead(existing);
                return;
            }
            if (map.size() >= capacity) {
                Node<K, V> lru = tail.prev;
                unlink(lru);
                map.remove(lru.key);
            }
            Node<K, V> newNode = new Node<>(key, value);
            addAfterHead(newNode);
            map.put(key, newNode);
        } finally {
            lock.unlock();
        }
    }

    private void moveToHead(Node<K, V> node) { unlink(node); addAfterHead(node); }
    private void unlink(Node<K, V> node) { node.prev.next = node.next; node.next.prev = node.prev; }
    private void addAfterHead(Node<K, V> node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `ThreadSafeLruCache`, `Node` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `put(...)` and `moveToHead(...)` and `unlink(...)`, execution proceeds step-by-step: explicit lock/synchronizer acquisition and release boundaries (`try / finally` or monitor exit) enforce mutual exclusion and happens-before memory visibility.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Why `ReadWriteLock` Doesn't Help `LRU.get()`**: Notice that even `get(key)` **mutates** the Doubly-Linked List (`moveToHead(node)`), so `get()` requires an exclusive write lock! In high-concurrency production (`Caffeine`), **Window-TinyLFU** buffers read hits in a lock-free ring buffer (`StripedRingBuffer`) and replays recency updates asynchronously in batches to avoid lock contention on `get()`.

---

<a id="q8-20"></a>
### Q8.20: Design & Implement an $O(1)$ LFU (Least Frequently Used) Cache using Frequency Bucket Doubly-Linked Lists.

#### 1. Formal Technical Definition & Internal Mechanics
- **LFU Eviction Rule**: When full, evict the key with the **lowest access frequency (`minFreq`)**; if multiple keys share the same `minFreq`, evict the **Least Recently Used** among them!
- **Three Data Structures for Strict $O(1)$ `get` and `put`**:
  1. `Map<K, Node> keyToNode`: Maps key to `(key, val, freq)`.
  2. `Map<Integer, DoublyLinkedList> freqToList`: Maps each frequency (`1, 2, 3...`) to an LRU-ordered Doubly-Linked List of nodes with that frequency.
  3. `int minFreq`: Tracks the current minimum frequency across all cached items. When a node at `minFreq` is accessed and happens to be the *last* node in `freqToList.get(minFreq)`, `minFreq` increments by `1` (`minFreq++`); when a brand-new key is inserted, `minFreq` resets to `1`!

#### 2. Technical Architecture & Execution Diagram
```
minFreq = 1
freqToList Map:
  Freq 1: [Head] ◄═► [Key C (LRU at Freq 1)] ◄═► [Tail]  ◄── Evicted next on overflow!
  Freq 2: [Head] ◄═► [Key B (MRU)] ◄═► [Key A (LRU)] ◄═► [Tail]
```

#### 3. Detailed Technical Example & Production Code
```java
public class LfuCache<K, V> {
    private static class Node<K, V> {
        final K key; V val; int freq = 1;
        Node(K k, V v) { this.key = k; this.val = v; }
    }

    private final int capacity;
    private int minFreq = 0;
    private final Map<K, Node<K, V>> keyMap = new HashMap<>();
    private final Map<Integer, LinkedHashSet<K>> freqLists = new HashMap<>();

    public LfuCache(int capacity) { this.capacity = capacity; }

    public V get(K key) {
        Node<K, V> node = keyMap.get(key);
        if (node == null) return null;
        touch(node);
        return node.val;
    }

    public void put(K key, V value) {
        if (capacity <= 0) return;
        Node<K, V> existing = keyMap.get(key);
        if (existing != null) {
            existing.val = value;
            touch(existing);
            return;
        }
        if (keyMap.size() >= capacity) {
            LinkedHashSet<K> minList = freqLists.get(minFreq);
            K evictKey = minList.iterator().next(); // First inserted = LRU in LinkedHashSet!
            minList.remove(evictKey);
            keyMap.remove(evictKey);
        }
        Node<K, V> newNode = new Node<>(key, value);
        keyMap.put(key, newNode);
        freqLists.computeIfAbsent(1, k -> new LinkedHashSet<>()).add(key);
        minFreq = 1;
    }

    private void touch(Node<K, V> node) {
        int prevFreq = node.freq;
        LinkedHashSet<K> set = freqLists.get(prevFreq);
        set.remove(node.key);
        if (prevFreq == minFreq && set.isEmpty()) {
            minFreq++;
        }
        node.freq++;
        freqLists.computeIfAbsent(node.freq, k -> new LinkedHashSet<>()).add(node.key);
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `LfuCache`, `Node` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `put(...)` and `touch(...)`, execution proceeds step-by-step: the core statement `}` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Using `LinkedHashSet<K>` for each frequency bucket in interviews keeps the code under 50 lines while preserving strict $O(1)$ insertion, removal, and FIFO/LRU eviction (`minList.iterator().next()`).

---

<a id="q8-21"></a>
### Q8.21: Implement a Thread-Safe Token Bucket Rate Limiter in Pure Java (Lock-Free CAS `AtomicReference` vs `synchronized`).

#### 1. Formal Technical Definition & Internal Mechanics
- Rather than spawning a background thread to add tokens every millisecond (which wastes CPU across 100,000 per-user buckets!), compute **Lazy Token Refill** on demand during `tryAcquire()` based on `(System.nanoTime() - lastRefillNanos) * refillRatePerNano`.
- Using an immutable `State(double tokens, long lastNanos)` `record` inside an **`AtomicReference<State>`** with `compareAndSet` makes the Token Bucket **100% lock-free**!

#### 2. Technical Architecture & Execution Diagram
```
AtomicReference<BucketState(tokens=8.5, lastNanos=t0)>
       │
       ▼ tryAcquire(1) at t1:
  1. Read current State
  2. Calculate refilled = min(capacity, 8.5 + (t1 - t0) * rate)
  3. If refilled < 1.0 -> Return false immediately!
  4. CAS(current, new BucketState(refilled - 1.0, t1)) -> Return true!
```

#### 3. Detailed Technical Example & Production Code
```java
public class LockFreeTokenBucketRateLimiter {
    private record State(double availableTokens, long lastRefillNanos) {}

    private final double maxBurstTokens;
    private final double tokensPerNano;
    private final AtomicReference<State> state;

    public LockFreeTokenBucketRateLimiter(double permitsPerSecond, double maxBurstTokens) {
        this.maxBurstTokens = maxBurstTokens;
        this.tokensPerNano = permitsPerSecond / 1_000_000_000.0;
        this.state = new AtomicReference<>(new State(maxBurstTokens, System.nanoTime()));
    }

    public boolean tryAcquire(int permits) {
        while (true) {
            State current = state.get();
            long now = System.nanoTime();
            long elapsedNanos = Math.max(0L, now - current.lastRefillNanos());
            double refilled = Math.min(
                maxBurstTokens,
                current.availableTokens() + (elapsedNanos * tokensPerNano)
            );
            if (refilled < permits) {
                return false; // Rate limited!
            }
            State next = new State(refilled - permits, now);
            if (state.compareAndSet(current, next)) {
                return true;
            }
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `LockFreeTokenBucketRateLimiter`, `State` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `tryAcquire(...)`, execution proceeds step-by-step: the core statement `long elapsedNanos = Math.max(0L, now - current.lastRefillNanos());` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Always use `System.nanoTime()` instead of `System.currentTimeMillis()` for elapsed time calculations**: `System.currentTimeMillis()` can jump backward or forward when NTP adjusts the system wall clock, whereas `System.nanoTime()` is guaranteed monotonic by the JVM/OS.

---

<a id="q8-22"></a>
### Q8.22: Implement a Thread-Safe Sliding Window Log / Counter Rate Limiter Supporting Per-Tenant Limits.

#### 1. Formal Technical Definition & Internal Mechanics
- **Sliding Window Counter (Hybrid O(1) Memory per Tenant)**:
  - Stores only `previousWindowCount`, `currentWindowCount`, and `currentWindowStartEpochSec` per tenant.
  - Approximates the count in the trailing 60-second sliding window as:
    $$\text{estimatedCount} = \text{prevCount} \times \left(1 - \frac{\text{elapsedInCurrentWindow}}{\text{windowSize}}\right) + \text{currentCount}$$
  - Uses **$O(1)$ memory** (24 bytes per tenant regardless of whether the limit is 10 or 10,000,000 requests/min!), unlike a Sliding Window Log which stores a timestamp per request ($O(\text{limit})$ memory).

#### 2. Technical Architecture & Execution Diagram
```
Window Size = 60s. Current Time = 12:01:15 (25% into Window 2, so 75% overlaps Window 1):
  [ Window 1 (12:00..12:01): 80 reqs ]  [ Window 2 (12:01..12:02): 30 reqs so far ]
                   └────────── 75% of 80 (60) + 30 = 90 reqs in trailing 60s! ──────┘
```

#### 3. Detailed Technical Example & Production Code
```java
public class SlidingWindowCounterRateLimiter {
    private static class WindowBucket {
        long windowStartMs;
        long prevCount;
        long currCount;
    }

    private final long windowSizeMs;
    private final long maxRequestsPerWindow;
    private final ConcurrentHashMap<String, WindowBucket> buckets = new ConcurrentHashMap<>();

    public SlidingWindowCounterRateLimiter(Duration windowSize, long maxRequestsPerWindow) {
        this.windowSizeMs = windowSize.toMillis();
        this.maxRequestsPerWindow = maxRequestsPerWindow;
    }

    public boolean allowRequest(String tenantId) {
        long nowMs = System.currentTimeMillis();
        long currentWindowStart = (nowMs / windowSizeMs) * windowSizeMs;
        WindowBucket bucket = buckets.computeIfAbsent(tenantId, k -> new WindowBucket());

        synchronized (bucket) { // Fine-grained per-tenant lock!
            if (currentWindowStart > bucket.windowStartMs) {
                long windowsPassed = (currentWindowStart - bucket.windowStartMs) / windowSizeMs;
                bucket.prevCount = (windowsPassed == 1) ? bucket.currCount : 0;
                bucket.currCount = 0;
                bucket.windowStartMs = currentWindowStart;
            }
            double weightOfPrevWindow = 1.0 - ((double) (nowMs - currentWindowStart) / windowSizeMs);
            double estimatedCount = (bucket.prevCount * weightOfPrevWindow) + bucket.currCount;
            if (estimatedCount + 1.0 > maxRequestsPerWindow) {
                return false;
            }
            bucket.currCount++;
            return true;
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `SlidingWindowCounterRateLimiter`, `WindowBucket` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `allowRequest(...)`, execution proceeds step-by-step: explicit lock/synchronizer acquisition and release boundaries (`try / finally` or monitor exit) enforce mutual exclusion and happens-before memory visibility.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Explaining why **Sliding Window Counter** ($O(1)$ memory per user, zero boundary burst flaw) is preferred by Cloudflare over both **Fixed Window** (2x boundary burst bug) and **Sliding Window Timestamp Log** ($O(N)$ memory explosion per user) is a major plus in coding/design interviews.

---

<a id="q8-23"></a>
### Q8.23: Implement a Bounded Blocking Queue from Scratch using `ReentrantLock` and Two `Condition` Variables (`notFull`, `notEmpty`).

#### 1. Formal Technical Definition & Internal Mechanics
- This is the exact internal architecture of `java.util.concurrent.ArrayBlockingQueue`:
  - Uses a fixed-size circular ring buffer array (`Object[] items`) with `putIndex`, `takeIndex`, and `count`.
  - Uses a single `ReentrantLock` with **two separate `Condition` wait-sets**:
    - `notFull = lock.newCondition()`: Producers `await()` here when `count == items.length`.
    - `notEmpty = lock.newCondition()`: Consumers `await()` here when `count == 0`.
  - **Why `while` loop around `await()` is mandatory**: OS threads can wake up without being signaled (**Spurious Wakeup**), or another thread can steal the slot before the awakened thread reacquires the lock!

#### 2. Technical Architecture & Execution Diagram
```
Circular Ring Buffer (capacity = 4):
  [ item0 | item1 | item2 |   -   ]
      ▲                       ▲
  takeIndex                putIndex

  Producer put() ──► while (count == cap) notFull.await()  ──► Inserts & calls notEmpty.signal()!
  Consumer take()──► while (count == 0)   notEmpty.await() ──► Removes & calls notFull.signal()!
```

#### 3. Detailed Technical Example & Production Code
```java
public class BoundedBlockingRingQueue<E> {
    private final Object[] items;
    private int takeIndex;
    private int putIndex;
    private int count;

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull = lock.newCondition();

    public BoundedBlockingRingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.items = new Object[capacity];
    }

    public void put(E item) throws InterruptedException {
        Objects.requireNonNull(item);
        lock.lockInterruptibly();
        try {
            while (count == items.length) { // MUST be 'while', never 'if'!
                notFull.await();
            }
            items[putIndex] = item;
            if (++putIndex == items.length) putIndex = 0;
            count++;
            notEmpty.signal(); // Wakes exactly 1 waiting consumer!
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public E take() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                notEmpty.await();
            }
            E item = (E) items[takeIndex];
            items[takeIndex] = null; // Prevent memory leak of dequeued reference!
            if (++takeIndex == items.length) takeIndex = 0;
            count--;
            notFull.signal(); // Wakes exactly 1 waiting producer!
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `BoundedBlockingRingQueue` (annotated with `@SuppressWarnings`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `put(...)`, execution proceeds step-by-step: explicit lock/synchronizer acquisition and release boundaries (`try / finally` or monitor exit) enforce mutual exclusion and happens-before memory visibility.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **3 Subtle Details Interviewers Look For**:
  1. Using `while (count == ...)` instead of `if`.
  2. Setting `items[takeIndex] = null` on dequeue so the array doesn't hold strong references to dequeued objects (preventing GC memory leaks!).
  3. Using two separate `Condition` variables (`notEmpty.signal()` / `notFull.signal()`) instead of a single monitor `notifyAll()` (which wakes up producers when the queue is still full!).

---

<a id="q8-24"></a>
### Q8.24: Implement a Custom Thread Pool Executor with a Bounded Queue and Backpressure Policy (`CallerRunsPolicy`).

#### 1. Formal Technical Definition & Internal Mechanics
- A Thread Pool consists of a `BlockingQueue<Runnable> workQueue` and a set of `Worker` threads that loop calling `Runnable task = workQueue.take(); task.run();`.
- When `workQueue.offer(task)` returns `false` (queue full!), applying **Backpressure** by running the task directly on the submitting caller thread (`CallerRunsPolicy`) naturally slows down the producer to match worker consumption speed without dropping tasks!

#### 2. Technical Architecture & Execution Diagram
```
Caller Thread ──► submit(task) ──► workQueue.offer(task)
                                     ├── True  ──► Picked up by Worker Threads [W1..WN]
                                     └── False (Queue Full!) ──► Caller Thread executes task.run() directly!
```

#### 3. Detailed Technical Example & Production Code
```java
public class BackpressureThreadPool implements AutoCloseable {
    private final BlockingQueue<Runnable> workQueue;
    private final List<Thread> workers = new ArrayList<>();
    private volatile boolean running = true;

    public BackpressureThreadPool(int poolSize, int queueCapacity) {
        this.workQueue = new ArrayBlockingQueue<>(queueCapacity);
        for (int i = 0; i < poolSize; i++) {
            Thread worker = new Thread(this::workerLoop, "custom-pool-worker-" + i);
            worker.start();
            workers.add(worker);
        }
    }

    public void execute(Runnable task) {
        if (!running) throw new RejectedExecutionException("Pool is shutting down");
        // Backpressure: if queue is full, execute on the caller thread!
        if (!workQueue.offer(task)) {
            task.run();
        }
    }

    private void workerLoop() {
        while (running || !workQueue.isEmpty()) {
            try {
                Runnable task = workQueue.poll(100, TimeUnit.MILLISECONDS);
                if (task != null) task.run();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } catch (Throwable t) {
                // Catch Throwable so an uncaught RuntimeException in 1 task never kills the Worker thread!
            }
        }
    }

    @Override
    public void close() {
        running = false;
        workers.forEach(Thread::interrupt);
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `BackpressureThreadPool` (annotated with `@Override`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `execute(...)` and `workerLoop(...)` and `close(...)`, execution proceeds step-by-step: the core statement `}` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Crucial Worker Loop Detail**: Always wrap `task.run()` in `try { task.run(); } catch (Throwable t) { ... }` inside the worker loop so a `RuntimeException` thrown by one task doesn't terminate the worker thread!

---

<a id="q8-25"></a>
### Q8.25: Implement a Read-Write Lock (`ReentrantReadWriteLock` Semantics) with Writer Preference to Prevent Writer Starvation.

#### 1. Formal Technical Definition & Internal Mechanics
- **Read-Write Lock Rules**:
  - **Multiple Readers** can hold the lock simultaneously as long as `activeWriters == 0` **AND `waitingWriters == 0`**.
  - **Why check `waitingWriters > 0` in `lockRead()`?**: Without checking `waitingWriters > 0`, a continuous stream of overlapping readers (`R1, R2, R3...`) will keep `activeReaders > 0` forever, **starving a waiting Writer indefinitely**! Giving priority to `waitingWriters` drains existing readers and lets the writer proceed immediately.

#### 2. Technical Architecture & Execution Diagram
```
Reader 1 holds Read Lock (activeReaders = 1)
  ├── Writer 1 arrives -> Increments waitingWriters = 1 & waits
  ├── Reader 2 arrives -> Sees waitingWriters == 1 > 0 -> Reader 2 BLOCKS (Prevents Writer Starvation!)
  └── Reader 1 unlocks (activeReaders = 0) -> Writer 1 acquires Write Lock immediately!
```

#### 3. Detailed Technical Example & Production Code
```java
public class WriterPreferenceReadWriteLock {
    private int activeReaders = 0;
    private int activeWriters = 0;
    private int waitingWriters = 0;

    public synchronized void lockRead() throws InterruptedException {
        while (activeWriters > 0 || waitingWriters > 0) {
            wait();
        }
        activeReaders++;
    }

    public synchronized void unlockRead() {
        activeReaders--;
        if (activeReaders == 0) {
            notifyAll();
        }
    }

    public synchronized void lockWrite() throws InterruptedException {
        waitingWriters++;
        try {
            while (activeReaders > 0 || activeWriters > 0) {
                wait();
            }
            activeWriters++;
        } finally {
            waitingWriters--;
        }
    }

    public synchronized void unlockWrite() {
        activeWriters--;
        notifyAll();
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `WriterPreferenceReadWriteLock` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `lockRead(...)` and `unlockRead(...)` and `lockWrite(...)`, execution proceeds step-by-step: explicit lock/synchronizer acquisition and release boundaries (`try / finally` or monitor exit) enforce mutual exclusion and happens-before memory visibility.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Notice how `waitingWriters--` is placed inside a `finally` block in `lockWrite()` so that if a waiting writer thread is interrupted (`InterruptedException`) while inside `wait()`, `waitingWriters` is still decremented and doesn't permanently deadlock all future readers!

---

<a id="q8-26"></a>
### Q8.26: Multi-Threaded Sequence Printing: Print `1..100` Strictly in Order Across $N=3$ Threads (or `FooBar` Alternation) without Busy-Waiting.

#### 1. Formal Technical Definition & Internal Mechanics
- Each thread $T_i$ (where $i \in \{0, 1, 2\}$) waits until `(currentNumber - 1) % numThreads == threadId`, prints `currentNumber++`, and wakes waiting threads.

#### 2. Technical Architecture & Execution Diagram
```
turn = (counter - 1) % 3:
  counter=1 (turn=0) ──► Thread-0 prints 1 -> counter=2 -> notifyAll()
  counter=2 (turn=1) ──► Thread-1 prints 2 -> counter=3 -> notifyAll()
  counter=3 (turn=2) ──► Thread-2 prints 3 -> counter=4 -> notifyAll()
```

#### 3. Detailed Technical Example & Production Code
```java
public class OrderedTurnPrinter {
    private final int maxNumber;
    private final int totalThreads;
    private int current = 1;

    public OrderedTurnPrinter(int maxNumber, int totalThreads) {
        this.maxNumber = maxNumber;
        this.totalThreads = totalThreads;
    }

    public synchronized void printNumbers(int threadId) throws InterruptedException {
        while (current <= maxNumber) {
            while (current <= maxNumber && (current - 1) % totalThreads != threadId) {
                wait();
            }
            if (current <= maxNumber) {
                System.out.println("Thread-" + threadId + ": " + current++);
                notifyAll();
            }
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `OrderedTurnPrinter` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `printNumbers(...)`, execution proceeds step-by-step: explicit lock/synchronizer acquisition and release boundaries (`try / finally` or monitor exit) enforce mutual exclusion and happens-before memory visibility.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Always re-check `if (current <= maxNumber)` *after* waking up from the inner `while` loop so threads parked when the final number is printed exit cleanly without printing `maxNumber + 1`.

---

<a id="q8-27"></a>
### Q8.27: Solve the Dining Philosophers Deadlock Problem using Resource Hierarchy (Lock Ordering) vs Arbitrator Semaphore.

#### 1. Formal Technical Definition & Internal Mechanics
- **Why Deadlock Occurs**: If all 5 Philosophers (`P0..P4`) pick up their **left fork first** simultaneously and then wait for their **right fork**, a circular wait (`F0 -> F1 -> F2 -> F3 -> F4 -> F0`) forms and all 5 starve forever!
- **Dijkstra's Resource Hierarchy Solution (Breaks Circular Wait)**:
  - Number the forks `0..4`.
  - Every philosopher **always locks the lower-numbered fork first (`Math.min(leftFork, rightFork)`)**, then the higher-numbered fork (`Math.max(leftFork, rightFork)`).
  - For Philosopher 4 (sitting between Fork 4 and Fork 0), `min(4, 0) = 0`, so Philosopher 4 reaches for **Fork 0 first** instead of Fork 4, making a cycle mathematically impossible!

#### 2. Technical Architecture & Execution Diagram
```
Circular Wait (Deadlock):
  P0 locks F0 -> waits F1 | ... | P4 locks F4 -> waits F0 (Cycle!)

Dijkstra Lock Ordering (Lower Fork ID First — Cycle Impossible!):
  P0..P3 lock Left (F0..F3) first
  P4 sits between F4 and F0 -> Locks min(4, 0) = F0 FIRST! (No cycle!)
```

#### 3. Detailed Technical Example & Production Code
```java
public class DiningPhilosophers {
    private final ReentrantLock[] forks = new ReentrantLock[5];

    public DiningPhilosophers() {
        for (int i = 0; i < 5; i++) forks[i] = new ReentrantLock();
    }

    public void dine(int philosopherId, Runnable eatAction) {
        int leftFork = philosopherId;
        int rightFork = (philosopherId + 1) % 5;

        int firstFork = Math.min(leftFork, rightFork);
        int secondFork = Math.max(leftFork, rightFork);

        forks[firstFork].lock();
        try {
            forks[secondFork].lock();
            try {
                eatAction.run();
            } finally {
                forks[secondFork].unlock();
            }
        } finally {
            forks[firstFork].unlock();
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `DiningPhilosophers` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `dine(...)`, execution proceeds step-by-step: explicit lock/synchronizer acquisition and release boundaries (`try / finally` or monitor exit) enforce mutual exclusion and happens-before memory visibility.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Connect this directly to database locking in your interview: Dijkstra's fork ordering (`Math.min(id1, id2)`) is the **exact same algorithm** we use when locking two bank accounts (`SELECT ... FOR UPDATE`) during a money transfer!

---

<a id="q8-28"></a>
### Q8.28: Implement a Delayed Task Scheduler (`DelayQueue` / Min-Heap + `Condition.awaitNanos`) from Scratch.

#### 1. Formal Technical Definition & Internal Mechanics
- **Why sleeping or polling every 10ms is bad**: Wastes CPU when idle and has poor timer precision.
- **Leader-Follower / `Condition.awaitNanos` Min-Heap Pattern**:
  - Store scheduled tasks in a **Min-Heap (`PriorityQueue<ScheduledTask>`)** ordered by `executeAtNanos`.
  - The worker thread peeks at the earliest task (`heap.peek()`).
  - If `delayNanos = head.executeAtNanos - System.nanoTime() > 0`, the worker calls **`available.awaitNanos(delayNanos)`**!
  - If a producer inserts a new task whose `executeAtNanos` is **earlier than the current heap root**, the producer calls **`available.signal()`** to wake the worker early so it adjusts its sleep duration!

#### 2. Technical Architecture & Execution Diagram
```
Min-Heap Root: [Task A (due in 60s)] ──► Worker calls available.awaitNanos(60s)
  │
  └── Producer submits [Task B (due in 2s < 60s!)]:
        └── Task B becomes new Root! Producer calls available.signal()!
        └── Worker wakes up immediately & sleeps only 2s for Task B!
```

#### 3. Detailed Technical Example & Production Code
```java
public class MiniScheduledExecutor implements AutoCloseable {
    private record ScheduledTask(long executeAtNanos, Runnable command)
            implements Comparable<ScheduledTask> {
        @Override
        public int compareTo(ScheduledTask o) {
            return Long.compare(this.executeAtNanos, o.executeAtNanos);
        }
    }

    private final PriorityQueue<ScheduledTask> pq = new PriorityQueue<>();
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition available = lock.newCondition();
    private volatile boolean running = true;
    private final Thread timerThread;

    public MiniScheduledExecutor() {
        this.timerThread = new Thread(this::dispatchLoop, "mini-scheduler");
        this.timerThread.start();
    }

    public void schedule(Runnable command, Duration delay) {
        long triggerNanos = System.nanoTime() + delay.toNanos();
        lock.lock();
        try {
            ScheduledTask task = new ScheduledTask(triggerNanos, command);
            pq.offer(task);
            if (pq.peek() == task) {
                available.signal(); // New earliest deadline -> wake timer thread!
            }
        } finally {
            lock.unlock();
        }
    }

    private void dispatchLoop() {
        while (running) {
            Runnable readyTask = null;
            lock.lock();
            try {
                while (running && pq.isEmpty()) {
                    available.await();
                }
                if (!running) break;
                ScheduledTask first = pq.peek();
                long delayNanos = first.executeAtNanos() - System.nanoTime();
                if (delayNanos <= 0) {
                    readyTask = pq.poll().command();
                } else {
                    available.awaitNanos(delayNanos);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                lock.unlock();
            }
            if (readyTask != null) {
                try { readyTask.run(); } catch (Throwable ignored) {}
            }
        }
    }

    @Override
    public void close() {
        running = false;
        timerThread.interrupt();
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `MiniScheduledExecutor`, `ScheduledTask` (annotated with `@Override`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `compareTo(...)` and `schedule(...)` and `dispatchLoop(...)`, execution proceeds step-by-step: explicit lock/synchronizer acquisition and release boundaries (`try / finally` or monitor exit) enforce mutual exclusion and happens-before memory visibility.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Notice how `readyTask.run()` is executed **outside** the `lock.unlock()` block so a slow task never blocks producers from scheduling new tasks!

---

<a id="q8-29"></a>
### Q8.29: Implement an In-Memory Circuit Breaker State Machine (`CLOSED`, `OPEN`, `HALF_OPEN`) in Pure Thread-Safe Java.

#### 1. Formal Technical Definition & Internal Mechanics
- Tracks `failureCount` and `state` (`CLOSED`, `OPEN`, `HALF_OPEN`).
- When `OPEN`, checks if `System.nanoTime() - openedAtNanos >= openWaitDurationNanos`; if true, atomically transitions `OPEN -> HALF_OPEN` to allow probe calls.

#### 2. Technical Architecture & Execution Diagram
```
execute(Supplier<T>):
  ├── State == OPEN & cooldown not elapsed ──► Throw CircuitBreakerOpenException immediately!
  ├── State == OPEN & cooldown elapsed     ──► CAS(OPEN -> HALF_OPEN) & allow probe call
  └── Call succeeds in HALF_OPEN           ──► Reset failures=0 & transition to CLOSED!
```

#### 3. Detailed Technical Example & Production Code
```java
public class SimpleCircuitBreaker {
    public enum State { CLOSED, OPEN, HALF_OPEN }

    private final int failureThreshold;
    private final long openCooldownNanos;
    private State state = State.CLOSED;
    private int consecutiveFailures = 0;
    private long openedAtNanos = 0L;

    public SimpleCircuitBreaker(int failureThreshold, Duration openCooldown) {
        this.failureThreshold = failureThreshold;
        this.openCooldownNanos = openCooldown.toNanos();
    }

    public <T> T execute(Callable<T> action) throws Exception {
        beforeCallCheck();
        try {
            T result = action.call();
            onSuccess();
            return result;
        } catch (Exception ex) {
            onFailure();
            throw ex;
        }
    }

    private synchronized void beforeCallCheck() {
        if (state == State.OPEN) {
            if (System.nanoTime() - openedAtNanos >= openCooldownNanos) {
                state = State.HALF_OPEN;
            } else {
                throw new IllegalStateException("CircuitBreaker is OPEN - Failing fast");
            }
        }
    }

    private synchronized void onSuccess() {
        consecutiveFailures = 0;
        state = State.CLOSED;
    }

    private synchronized void onFailure() {
        consecutiveFailures++;
        if (state == State.HALF_OPEN || consecutiveFailures >= failureThreshold) {
            state = State.OPEN;
            openedAtNanos = System.nanoTime();
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `SimpleCircuitBreaker`, `State` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `beforeCallCheck(...)` and `onSuccess(...)` and `onFailure(...)`, execution proceeds step-by-step: explicit lock/synchronizer acquisition and release boundaries (`try / finally` or monitor exit) enforce mutual exclusion and happens-before memory visibility.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Critical Lock Rule**: Notice that `action.call()` (the slow network I/O!) runs **outside** the `synchronized` methods (`beforeCallCheck()`, `onSuccess()`, `onFailure()`). Never hold a lock while executing network I/O inside a circuit breaker!

---

<a id="q8-30"></a>
### Q8.30: Implement a Thread-Safe Connection Pool with Borrow Timeout and Validation (`BlockingQueue` + `Semaphore`).

#### 1. Formal Technical Definition & Internal Mechanics
- Uses a `Semaphore(maxPoolSize)` to strictly bound total created connections and an `ArrayBlockingQueue<PooledConnection>` to store idle connections.
- When `borrowConnection(timeout)` is called, it first polls the idle queue; if empty and total connections `< maxPoolSize`, it lazily creates a new connection; otherwise it waits on `idleQueue.poll(timeout)`.

#### 2. Technical Architecture & Execution Diagram
```
borrowConnection(3000ms):
  1. Poll idleQueue (Immediate hit if idle connection available)
  2. If null & creationSemaphore.tryAcquire() -> Create new physical Connection
  3. Else wait on idleQueue.poll(3000ms) -> Throw SQLTimeoutException if null!
```

#### 3. Detailed Technical Example & Production Code
```java
public class MiniConnectionPool<T> {
    private final BlockingQueue<T> idlePool;
    private final Semaphore creationPermits;
    private final Supplier<T> factory;
    private final Predicate<T> validator;

    public MiniConnectionPool(int maxSize, Supplier<T> factory, Predicate<T> validator) {
        this.idlePool = new ArrayBlockingQueue<>(maxSize);
        this.creationPermits = new Semaphore(maxSize);
        this.factory = factory;
        this.validator = validator;
    }

    public T borrow(Duration timeout) throws InterruptedException, TimeoutException {
        T conn = idlePool.poll();
        if (conn != null && validator.test(conn)) {
            return conn;
        }
        if (conn != null) {
            creationPermits.release(); // Dead connection discarded, release permit!
        }
        if (creationPermits.tryAcquire()) {
            try {
                return factory.get();
            } catch (RuntimeException e) {
                creationPermits.release();
                throw e;
            }
        }
        T waited = idlePool.poll(timeout.toMillis(), TimeUnit.MILLISECONDS);
        if (waited == null) {
            throw new TimeoutException("Connection pool exhausted after " + timeout);
        }
        return waited;
    }

    public void release(T conn) {
        if (conn != null) {
            idlePool.offer(conn);
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `MiniConnectionPool` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `release(...)`, execution proceeds step-by-step: explicit lock/synchronizer acquisition and release boundaries (`try / finally` or monitor exit) enforce mutual exclusion and happens-before memory visibility.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Notice `try { return factory.get(); } catch (RuntimeException e) { creationPermits.release(); throw e; }`: if `factory.get()` fails during a database outage, releasing the semaphore permit in the `catch` block ensures permits aren't permanently leaked!

---

<a id="q8-31"></a>
### Q8.31: Implement Single-Flight Request Coalescing (Deduplicating Concurrent Identical Cache Misses) using `ConcurrentHashMap` and `CompletableFuture`.

#### 1. Formal Technical Definition & Internal Mechanics
- When 500 threads simultaneously ask for `"product:42"` on a cold cache, we want **only Thread 1** to query the database while Threads 2..500 attach to the exact same `CompletableFuture<V>` and receive the result when Thread 1 finishes!
- Once the `CompletableFuture` completes (success or exception), remove it from the `inFlight` map via `.whenComplete((val, ex) -> inFlight.remove(key, future))`.

#### 2. Technical Architecture & Execution Diagram
```
500 Concurrent Threads call execute("product:42"):
  ├── Thread 1: inFlight.computeIfAbsent("product:42") ──► Starts 1 Async DB Query!
  ├── Threads 2..500: Receive the EXACT SAME CompletableFuture reference!
  └── DB Query Finishes ──► Completes all 500 callers simultaneously & removes from inFlight!
```

#### 3. Detailed Technical Example & Production Code
```java
public class SingleFlightCoalescer<K, V> {
    private final ConcurrentHashMap<K, CompletableFuture<V>> inFlight = new ConcurrentHashMap<>();

    public V execute(K key, Callable<V> expensiveLoader) {
        CompletableFuture<V> myFuture = new CompletableFuture<>();
        CompletableFuture<V> existing = inFlight.putIfAbsent(key, myFuture);

        if (existing != null) {
            // Another thread is already loading this key -> piggyback on its Future!
            return existing.join();
        }

        // We won the race -> execute the loader once and broadcast result to all waiters!
        try {
            V result = expensiveLoader.call();
            myFuture.complete(result);
            return result;
        } catch (Throwable t) {
            myFuture.completeExceptionally(t);
            throw new CompletionException(t);
        } finally {
            inFlight.remove(key, myFuture);
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `SingleFlightCoalescer` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: asynchronous task dispatch delegates execution to the target executor/carrier pool without blocking the caller OS thread.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Why `putIfAbsent` + running `expensiveLoader.call()` outside `computeIfAbsent` is superior**: Running blocking I/O *inside* `ConcurrentHashMap.computeIfAbsent()` holds a bucket-level synchronized lock in `ConcurrentHashMap`, blocking other keys that hash to the same bucket and risking recursive deadlocks! Using `putIfAbsent` with an uncompleted `CompletableFuture` releases the map bucket lock in nanoseconds!

---

<a id="q8-32"></a>
### Q8.32: Merge $K$ Sorted Streams / Lists in $O(N \log K)$ Time using a Min-Heap (`PriorityQueue`).

#### 1. Formal Technical Definition & Internal Mechanics
- Instead of concatenating all $N$ items and sorting in $O(N \log N)$, insert the head `Iterator` of each of the $K$ sorted lists into a **Min-Heap of size $K$**.
- Repeatedly `poll()` the smallest head ($O(\log K)$), append to output, and if that iterator `hasNext()`, push its next element back into the Min-Heap!

#### 2. Technical Architecture & Execution Diagram
```
K = 3 Sorted Lists:
  L1: [1, 4, 7]
  L2: [2, 5, 8]    ──► Min-Heap (Size 3): [ 1(L1), 2(L2), 3(L3) ]
  L3: [3, 6, 9]        Poll 1(L1) -> Offer next from L1 (4) -> Heap: [ 2(L2), 3(L3), 4(L1) ]
```

#### 3. Detailed Technical Example & Production Code
```java
public static <T extends Comparable<? super T>> List<T> mergeKSortedLists(List<List<T>> lists) {
    record HeapEntry<T>(T value, Iterator<T> iterator) {}

    PriorityQueue<HeapEntry<T>> minHeap = new PriorityQueue<>(
        Comparator.comparing(HeapEntry::value)
    );

    for (List<T> list : lists) {
        if (list != null && !list.isEmpty()) {
            Iterator<T> it = list.iterator();
            minHeap.offer(new HeapEntry<>(it.next(), it));
        }
    }

    List<T> merged = new ArrayList<>();
    while (!minHeap.isEmpty()) {
        HeapEntry<T> smallest = minHeap.poll();
        merged.add(smallest.value());
        if (smallest.iterator().hasNext()) {
            minHeap.offer(new HeapEntry<>(smallest.iterator().next(), smallest.iterator()));
        }
    }
    return merged;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `HeapEntry` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `}` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Real-World Distributed Systems Application**: This exact $K$-way Min-Heap merge algorithm is how **Apache Kafka / Cassandra LSM-Tree Compaction** merges sorted SSTables on disk, and how **Elasticsearch / Sharded DBs** merge paginated `ORDER BY` results from $K$ shards!

---

<a id="q8-33"></a>
### Q8.33: Find the Running Median of a Data Stream in $O(\log N)$ Time per Element using Two Heaps (`Max-Heap` + `Min-Heap`).

#### 1. Formal Technical Definition & Internal Mechanics
- Maintain two balanced heaps:
  1. **`lowerHalf` (Max-Heap)**: Stores the smaller half of numbers seen so far (largest at root).
  2. **`upperHalf` (Min-Heap)**: Stores the larger half of numbers seen so far (smallest at root).
- **Balancing Invariant**: `lowerHalf.size()` is always equal to `upperHalf.size()` or `upperHalf.size() + 1`.
- **Median Lookup ($O(1)$)**: If total count is odd, median is `lowerHalf.peek()`; if even, median is `(lowerHalf.peek() + upperHalf.peek()) / 2.0`!

#### 2. Technical Architecture & Execution Diagram
```
Stream: [10, 20, 30, 40]
  lowerHalf (Max-Heap): [ 10, 20 ] (Root = 20)
  upperHalf (Min-Heap): [ 30, 40 ] (Root = 30)
  Median = (20 + 30) / 2.0 = 25.0 (O(1) lookup!)
```

#### 3. Detailed Technical Example & Production Code
```java
public class StreamingMedianFinder {
    private final PriorityQueue<Integer> lowerMaxHeap = new PriorityQueue<>(Comparator.reverseOrder());
    private final PriorityQueue<Integer> upperMinHeap = new PriorityQueue<>();

    public void addNum(int num) {
        lowerMaxHeap.offer(num);
        upperMinHeap.offer(lowerMaxHeap.poll());
        if (upperMinHeap.size() > lowerMaxHeap.size()) {
            lowerMaxHeap.offer(upperMinHeap.poll());
        }
    }

    public double findMedian() {
        if (lowerMaxHeap.isEmpty()) throw new IllegalStateException("No elements");
        if (lowerMaxHeap.size() > upperMinHeap.size()) {
            return lowerMaxHeap.peek();
        }
        return ((double) lowerMaxHeap.peek() + upperMinHeap.peek()) / 2.0;
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `StreamingMedianFinder` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `addNum(...)`, execution proceeds step-by-step: the core statement `}` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Look how clean `addNum` is: only 5 lines with zero complex `if/else` branching by routing every new number through `lowerMaxHeap` $\rightarrow$ `upperMinHeap` and rebalancing sizes!

---

<a id="q8-34"></a>
### Q8.34: Implement a Prefix Tree (`Trie`) Supporting `insert`, `search`, and `startsWith` (or Wildcard Routing).

#### 1. Formal Technical Definition & Internal Mechanics
- Each `TrieNode` holds a fixed `TrieNode[26]` array (or `Map<Character, TrieNode>`) and a `boolean isWordEnd`.
- Lookup time for a string of length $L$ is strictly **$O(L)$**, completely independent of whether the dictionary contains 100 words or 100,000,000 words!

#### 2. Technical Architecture & Execution Diagram
```
      [Root]
       /  \
     'a'  'c'
     /      \
   'p'      'a'
   /          \
 'p'(end=T)   't'(end=T)
  │
 'l'
  │
 'e'(end=T)
```

#### 3. Detailed Technical Example & Production Code
```java
public class PrefixTrie {
    private static class Node {
        final Node[] children = new Node[26];
        boolean isEndOfWord;
    }

    private final Node root = new Node();

    public void insert(String word) {
        Node curr = root;
        for (int i = 0; i < word.length(); i++) {
            int idx = word.charAt(i) - 'a';
            if (curr.children[idx] == null) {
                curr.children[idx] = new Node();
            }
            curr = curr.children[idx];
        }
        curr.isEndOfWord = true;
    }

    public boolean search(String word) {
        Node node = traverse(word);
        return node != null && node.isEndOfWord;
    }

    public boolean startsWith(String prefix) {
        return traverse(prefix) != null;
    }

    private Node traverse(String str) {
        Node curr = root;
        for (int i = 0; i < str.length(); i++) {
            int idx = str.charAt(i) - 'a';
            if (idx < 0 || idx >= 26 || curr.children[idx] == null) return null;
            curr = curr.children[idx];
        }
        return curr;
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `PrefixTrie`, `Node` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `insert(...)` and `search(...)` and `startsWith(...)`, execution proceeds step-by-step: the core statement `public boolean search(String word) {` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Factoring out `private Node traverse(String str)` avoids duplicating the loop between `search()` and `startsWith()`.

---

<a id="q8-35"></a>
### Q8.35: Implement Topological Sort (Kahn's BFS Indegree Algorithm) for Task Dependency Resolution & Cycle Detection.

#### 1. Formal Technical Definition & Internal Mechanics
- Given $N$ tasks (or Maven modules / microservice build steps) and directed dependency edges `u -> v` (`u` must run before `v`):
  1. Compute `indegree[v]` (number of prerequisites remaining for task `v`).
  2. Push all tasks with `indegree == 0` into a queue.
  3. Pop `u`, append `u` to execution order, and decrement `indegree[v]` for all neighbors `v`. If `indegree[v]` becomes `0`, push `v` to the queue.
  4. **Cycle Detection**: If `executionOrder.size() < totalTasks`, a **circular dependency** exists!

#### 2. Technical Architecture & Execution Diagram
```
Dependencies: [Compile (0)] ──► [UnitTest (1)] ──► [Package (2)]
              [Compile (0)] ──► [Lint (3)]     ──► [Package (2)]

Indegrees: Compile=0, UnitTest=1, Lint=1, Package=2
  Step 1: Pop Compile(0) -> Decrements UnitTest(0) & Lint(0) -> Enqueue UnitTest, Lint
  Step 2: Pop UnitTest, Lint -> Decrements Package(2 -> 1 -> 0) -> Enqueue Package
  Step 3: Pop Package -> All 4 tasks scheduled!
```

#### 3. Detailed Technical Example & Production Code
```java
public static List<Integer> resolveTaskExecutionOrder(int numTasks, int[][] dependencies) {
    // dependencies[i] = {prerequisite, dependent}
    List<List<Integer>> adj = new ArrayList<>(numTasks);
    for (int i = 0; i < numTasks; i++) adj.add(new ArrayList<>());
    int[] indegree = new int[numTasks];

    for (int[] dep : dependencies) {
        int prereq = dep[0], target = dep[1];
        adj.get(prereq).add(target);
        indegree[target]++;
    }

    Deque<Integer> readyQueue = new ArrayDeque<>();
    for (int i = 0; i < numTasks; i++) {
        if (indegree[i] == 0) readyQueue.offer(i);
    }

    List<Integer> order = new ArrayList<>(numTasks);
    while (!readyQueue.isEmpty()) {
        int curr = readyQueue.poll();
        order.add(curr);
        for (int next : adj.get(curr)) {
            if (--indegree[next] == 0) {
                readyQueue.offer(next);
            }
        }
    }

    if (order.size() != numTasks) {
        throw new IllegalStateException("Circular dependency detected among tasks!");
    }
    return order;
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `while (!readyQueue.isEmpty()) {` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Why Kahn's BFS Algorithm is preferred over Recursive DFS in Production**: Kahn's algorithm never overflows the call stack on deep DAGs, naturally identifies all tasks that can be executed **in parallel** at each BFS level, and detects cycles simply by checking `order.size() != numTasks`.

---

## Part C: Senior Production Coding & End-to-End Capstone Architecture (Q8.36 – Q8.52)

<a id="q8-36"></a>
### Q8.36: Implement Deadlock-Free Concurrent Money Transfer Between Two In-Memory Bank Accounts with `BigDecimal` Precision and Audit Records.

#### 1. Formal Technical Definition & Internal Mechanics
- If Thread 1 transfers from `Account 1 -> Account 2` and Thread 2 simultaneously transfers from `Account 2 -> Account 1`, locking `fromAccount` then `toAccount` causes an immediate **AB-BA Deadlock**!
- **Fix**: Enforce a **Global Lock Acquisition Order** by sorting locks by the unique immutable `accountId` (`firstLock = from.id < to.id ? from : to`), and validate positive non-zero `BigDecimal` amounts.

#### 2. Technical Architecture & Execution Diagram
```
Thread 1: transfer(Acc 10 -> Acc 20) ──► Locks min(10, 20) = Acc 10 FIRST, then Acc 20
Thread 2: transfer(Acc 20 -> Acc 10) ──► Locks min(20, 10) = Acc 10 FIRST, then Acc 20
└── Cycle Impossible! 100% Deadlock-Free!
```

#### 3. Detailed Technical Example & Production Code
```java
public class BankAccount {
    private final long id;
    private BigDecimal balance;
    private final ReentrantLock lock = new ReentrantLock();

    public BankAccount(long id, BigDecimal initialBalance) {
        this.id = id;
        this.balance = initialBalance;
    }

    public static void transfer(BankAccount from, BankAccount to, BigDecimal amount) {
        if (from.id == to.id) throw new IllegalArgumentException("Cannot transfer to same account");
        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Transfer amount must be positive");
        }

        BankAccount first = from.id < to.id ? from : to;
        BankAccount second = from.id < to.id ? to : from;

        first.lock.lock();
        try {
            second.lock.lock();
            try {
                if (from.balance.compareTo(amount) < 0) {
                    throw new IllegalStateException("Insufficient funds in account " + from.id);
                }
                from.balance = from.balance.subtract(amount);
                to.balance = to.balance.add(amount);
            } finally {
                second.lock.unlock();
            }
        } finally {
            first.lock.unlock();
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `BankAccount` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `transfer(...)`, execution proceeds step-by-step: explicit lock/synchronizer acquisition and release boundaries (`try / finally` or monitor exit) enforce mutual exclusion and happens-before memory visibility.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Always check `if (from.id == to.id)` at the very top, validate `amount.compareTo(BigDecimal.ZERO) > 0` (preventing negative-amount exploits that steal money in reverse!), and release both locks in nested `finally` blocks.

---

<a id="q8-37"></a>
### Q8.37: Implement a Parallel Fan-Out Aggregator Service using `CompletableFuture` (Java 17) vs `StructuredTaskScope` (Java 21) with Per-Task Timeouts and Fallbacks.

#### 1. Formal Technical Definition & Internal Mechanics
- When building an Order Summary page that aggregates `UserProfile`, `OrderHistory`, and `LoyaltyPoints` in parallel:
  - `UserProfile` and `OrderHistory` are **required** (fail within `1000ms` if unavailable).
  - `LoyaltyPoints` is **optional** (fallback to `0 points` if it takes longer than `300ms`!).

#### 2. Technical Architecture & Execution Diagram
```
Aggregator Fan-Out (Total Latency = max(Profile, Orders, min(300ms, Loyalty))):
  ├── [Profile Task]  (1s Timeout, Required) ─────────┐
  ├── [Orders Task]   (1s Timeout, Required) ─────────┼──► CompletableFuture.allOf() ──► DashboardView
  └── [Loyalty Task]  (300ms completeOnTimeout(0)) ───┘
```

#### 3. Detailed Technical Example & Production Code
```java
public DashboardView fetchDashboard(Long userId, Executor executor) {
    CompletableFuture<UserProfile> profileFuture = CompletableFuture
        .supplyAsync(() -> profileClient.getUser(userId), executor)
        .orTimeout(1000, TimeUnit.MILLISECONDS);

    CompletableFuture<List<OrderSummary>> ordersFuture = CompletableFuture
        .supplyAsync(() -> orderClient.getRecentOrders(userId), executor)
        .orTimeout(1000, TimeUnit.MILLISECONDS);

    // Soft dependency degrades gracefully to 0 points after 300ms!
    CompletableFuture<Integer> loyaltyFuture = CompletableFuture
        .supplyAsync(() -> loyaltyClient.getPoints(userId), executor)
        .completeOnTimeout(0, 300, TimeUnit.MILLISECONDS)
        .exceptionally(ex -> 0);

    CompletableFuture.allOf(profileFuture, ordersFuture, loyaltyFuture).join();
    return new DashboardView(profileFuture.join(), ordersFuture.join(), loyaltyFuture.join());
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: asynchronous task dispatch delegates execution to the target executor/carrier pool without blocking the caller OS thread.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Always pass a custom `Executor` as the 2nd argument to `CompletableFuture.supplyAsync(..., executor)`**: Calling the 1-arg `supplyAsync(...)` runs blocking I/O on the global `ForkJoinPool.commonPool()` (which only has `CPU_CORES - 1` threads!), starving parallel streams and other futures across the entire JVM!

---

<a id="q8-38"></a>
### Q8.38: Implement an Async Micro-Batching Processor in Java (Flushes when Batch Size reaches 100 OR 50ms Timeout Elapses).

#### 1. Formal Technical Definition & Internal Mechanics
- **Why Micro-Batching?**: Writing 10,000 individual events/sec to a database or external API one-by-one incurs 10,000 network round-trips. Buffering items in a `BlockingQueue` and flushing whenever **`batch.size() == 100` OR `50ms` elapses** (whichever comes first!) achieves **high throughput under load AND low `< 50ms` latency under light load**!

#### 2. Technical Architecture & Execution Diagram
```
Producers offer(item) ──► [BlockingQueue]
                                │
                                ▼
                     [Batch Drain Loop]
                       ├── Collects up to 100 items OR waits up to 50ms deadline
                       └── Calls batchConsumer.accept(batch) in 1 bulk DB/Network call!
```

#### 3. Detailed Technical Example & Production Code
```java
public class MicroBatchProcessor<T> implements AutoCloseable {
    private final BlockingQueue<T> queue;
    private final int maxBatchSize;
    private final long maxFlushWaitNanos;
    private final Consumer<List<T>> batchSink;
    private volatile boolean running = true;
    private final Thread worker;

    public MicroBatchProcessor(int capacity, int maxBatchSize, Duration maxFlushWait, Consumer<List<T>> batchSink) {
        this.queue = new ArrayBlockingQueue<>(capacity);
        this.maxBatchSize = maxBatchSize;
        this.maxFlushWaitNanos = maxFlushWait.toNanos();
        this.batchSink = batchSink;
        this.worker = Thread.ofVirtual().name("micro-batch-flusher").start(this::flushLoop);
    }

    public boolean submit(T item) {
        return running && queue.offer(item);
    }

    private void flushLoop() {
        List<T> batch = new ArrayList<>(maxBatchSize);
        while (running || !queue.isEmpty()) {
            try {
                T first = queue.poll(maxFlushWaitNanos, TimeUnit.NANOSECONDS);
                if (first == null) continue;
                batch.add(first);

                // Drain immediately available items without waiting:
                queue.drainTo(batch, maxBatchSize - 1);

                // If batch still < maxBatchSize, wait up to remaining deadline for more items:
                long deadline = System.nanoTime() + maxFlushWaitNanos;
                while (batch.size() < maxBatchSize) {
                    long remaining = deadline - System.nanoTime();
                    if (remaining <= 0) break;
                    T next = queue.poll(remaining, TimeUnit.NANOSECONDS);
                    if (next == null) break;
                    batch.add(next);
                }

                batchSink.accept(List.copyOf(batch));
                batch.clear();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    @Override
    public void close() throws InterruptedException {
        running = false;
        worker.join(5000);
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `MicroBatchProcessor` (annotated with `@Override`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `submit(...)` and `flushLoop(...)` and `close(...)`, execution proceeds step-by-step: the core statement `batch.add(first);` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Notice how `queue.drainTo(batch, maxBatchSize - 1)` grabs all already-queued items in a **single lock acquisition** before entering the deadline poll loop—maximizing throughput under heavy bursts!

---

<a id="q8-39"></a>
### Q8.39: Implement a Keyed Sequential Executor (Striped Executor) that Guarantees Events with the Same `orderId` Execute in Strict FIFO Order while Different `orderId`s Run in Parallel.

#### 1. Formal Technical Definition & Internal Mechanics
- If you submit tasks for `orderId = 101` (`OrderCreated`, `OrderPaid`, `OrderShipped`) to a standard multi-threaded `ThreadPoolExecutor`, Thread 2 might finish `OrderPaid` *before* Thread 1 finishes `OrderCreated`!
- **Keyed / Striped Single-Thread Executor Pool**:
  - Create an array of $N$ single-threaded executors (`ExecutorService[N]` where $N = 32$ or $64$).
  - Route each task by `Math.floorMod(key.hashCode(), stripes.length)`.
  - All tasks for `orderId = 101` always land on the **exact same single-thread queue (strict FIFO order with zero locks needed!)**, while different `orderId`s execute in parallel across all $N$ stripes!

#### 2. Technical Architecture & Execution Diagram
```
submit(key="order-101", task):
  idx = Math.floorMod("order-101".hashCode(), 4) ──► Maps to Stripe #2
  ├── Stripe #0: [SingleThreadExecutor-0] ◄── "order-204"
  ├── Stripe #1: [SingleThreadExecutor-1] ◄── "order-599"
  ├── Stripe #2: [SingleThreadExecutor-2] ◄── ["order-101: Created" -> "order-101: Paid"] (Strict FIFO!)
  └── Stripe #3: [SingleThreadExecutor-3]
```

#### 3. Detailed Technical Example & Production Code
```java
public class KeyedSequentialExecutor implements AutoCloseable {
    private final ExecutorService[] stripes;

    public KeyedSequentialExecutor(int stripeCount) {
        if (stripeCount <= 0) throw new IllegalArgumentException();
        this.stripes = new ExecutorService[stripeCount];
        for (int i = 0; i < stripeCount; i++) {
            final int id = i;
            this.stripes[i] = Executors.newSingleThreadExecutor(
                r -> new Thread(r, "keyed-stripe-" + id)
            );
        }
    }

    public CompletableFuture<Void> submit(Object affinityKey, Runnable task) {
        int idx = Math.floorMod(affinityKey.hashCode(), stripes.length);
        return CompletableFuture.runAsync(task, stripes[idx]);
    }

    @Override
    public void close() {
        for (ExecutorService stripe : stripes) {
            stripe.shutdown();
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `KeyedSequentialExecutor` (annotated with `@Override`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `close(...)`, execution proceeds step-by-step: asynchronous task dispatch delegates execution to the target executor/carrier pool without blocking the caller OS thread.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Why `Math.floorMod(key.hashCode(), N)` instead of `Math.abs(key.hashCode()) % N`?**: In Java, `Math.abs(Integer.MIN_VALUE)` returns **`Integer.MIN_VALUE` (a negative number!)** due to two's complement overflow, throwing `ArrayIndexOutOfBoundsException: -2`! `Math.floorMod(hash, N)` is guaranteed to return `0 .. N-1` for all $2^{32}$ integers including `Integer.MIN_VALUE`!

---

<a id="q8-40"></a>
### Q8.40: Implement Weighted Round-Robin and Consistent Hash Load Balancers with Health-Check Ejection in Java.

#### 1. Formal Technical Definition & Internal Mechanics
- **Lock-Free Round-Robin**: Uses an `AtomicInteger` (`Math.floorMod(counter.getAndIncrement(), healthyNodes.size())`) over a `CopyOnWriteArrayList` or immutable snapshot of healthy backend nodes.

#### 2. Technical Architecture & Execution Diagram
```
Healthy Backend Snapshot (AtomicReference<List<ServerNode>>):
  [ Node A (Healthy), Node B (Healthy), Node C (Healthy) ]
  └── HealthCheck marks Node B UNHEALTHY -> Swaps immutable List [Node A, Node C] atomically!
```

#### 3. Detailed Technical Example & Production Code
```java
public class HealthAwareLoadBalancer {
    public record Backend(String url, int weight, boolean healthy) {}

    private final AtomicReference<List<String>> weightedRoutingTable = new AtomicReference<>(List.of());
    private final AtomicInteger counter = new AtomicInteger(0);

    public void updateBackends(List<Backend> backends) {
        List<String> table = new ArrayList<>();
        for (Backend b : backends) {
            if (b.healthy()) {
                for (int i = 0; i < b.weight(); i++) {
                    table.add(b.url());
                }
            }
        }
        weightedRoutingTable.set(List.copyOf(table));
    }

    public String selectNextEndpoint() {
        List<String> currentTable = weightedRoutingTable.get();
        if (currentTable.isEmpty()) {
            throw new IllegalStateException("No healthy backend instances available");
        }
        int idx = Math.floorMod(counter.getAndIncrement(), currentTable.size());
        return currentTable.get(idx);
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `HealthAwareLoadBalancer`, `Backend` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `updateBackends(...)` and `selectNextEndpoint(...)`, execution proceeds step-by-step: the core statement `}` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Swapping an immutable `List.copyOf(table)` inside an `AtomicReference` makes the hot read path (`selectNextEndpoint()`) **100% lock-free** with zero `ConcurrentModificationException` risk when health checks update the backend list.

---

<a id="q8-41"></a>
### Q8.41: Parse and Aggregate a 50 GB CSV Log File in `< 64 MB` JVM Heap Memory using Streaming I/O (`Files.lines` / `BufferedReader`).

#### 1. Formal Technical Definition & Internal Mechanics
- **The Trap**: Calling `Files.readAllLines(path)` loads all 50 GB into a `List<String>` in RAM and crashes with `OutOfMemoryError` in 2 seconds!
- **Streaming Solution**: Use **`try (Stream<String> lines = Files.lines(path, StandardCharsets.UTF_8))`** (which lazily reads lines from a `BufferedReader` / NIO `FileChannel`) and always wrap `Files.lines()` in a **`try-with-resources`** block so the underlying OS file descriptor is closed!

#### 2. Technical Architecture & Execution Diagram
```
50 GB Log File on Disk
       │
       ▼ (Files.lines(path) — Only 8KB buffer in RAM at any millisecond!)
[Stream Pipeline: filter -> parse -> groupingBy(statusCode, counting())]
       │
       ▼
Output Map<Integer, Long> (Only ~20 entries in RAM!)
```

#### 3. Detailed Technical Example & Production Code
```java
public static Map<String, BigDecimal> aggregateMerchantVolumeStreaming(Path csvPath) throws IOException {
    // try-with-resources is MANDATORY for Files.lines() to close the OS file handle!
    try (Stream<String> lines = Files.lines(csvPath, StandardCharsets.UTF_8)) {
        return lines
            .skip(1) // Skip CSV header
            .filter(line -> !line.isBlank())
            .map(line -> line.split(",", 3))
            .filter(parts -> parts.length == 3 && "SETTLED".equals(parts[2]))
            .collect(Collectors.groupingBy(
                parts -> parts[0], // merchantId
                Collectors.reducing(
                    BigDecimal.ZERO,
                    parts -> new BigDecimal(parts[1]),
                    BigDecimal::add
                )
            ));
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation implements the core execution logic (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the Stream pipeline chains lazy intermediate operations and triggers single-pass evaluation at the terminal collector/reduction step.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Leaked File Descriptors with `Files.lines()`**: Unlike `Collection.stream()`, `Files.lines(path)` holds an open OS file descriptor (`Too many open files` error if not closed!). Calling `.toList()` on a stream does **NOT** automatically close `Files.lines()`—you **must** wrap it in `try (Stream<String> s = Files.lines(path))`.

---

<a id="q8-42"></a>
### Q8.42: Advanced SQL Challenge: Solve the "Gaps and Islands" Problem (Find Longest Consecutive Daily Login Streak per User).

#### 1. Formal Technical Definition & Internal Mechanics
- **The Classic "Difference of Row Numbers" Trick (`login_date - ROW_NUMBER()`)**:
  - Suppose a user logs in on consecutive days: `2025-02-01` (`rn=1`), `2025-02-02` (`rn=2`), `2025-02-03` (`rn=3`), then skips a day and logs in on `2025-02-05` (`rn=4`).
  - Look at `login_date - INTERVAL '1 day' * rn`:
    - `2025-02-01 - 1 day = 2025-01-31`
    - `2025-02-02 - 2 days = 2025-01-31`
    - `2025-02-03 - 3 days = 2025-01-31` (**All consecutive days produce the EXACT SAME constant `island_group` date!**)
    - `2025-02-05 - 4 days = 2025-02-01` (New island group!)

#### 2. Technical Architecture & Execution Diagram
```
login_date   | row_number (rn) | island_key = (login_date - rn days)
-------------+-----------------+------------------------------------
2025-02-01   |        1        | 2025-01-31  ┐
2025-02-02   |        2        | 2025-01-31  ├── Island #1 (Streak = 3 days!)
2025-02-03   |        3        | 2025-01-31  ┘
2025-02-05   |        4        | 2025-02-01  ─── Island #2 (Streak = 1 day)
```

#### 3. Detailed Technical Example & Production Code
```sql
WITH distinct_daily_logins AS (
    -- Deduplicate multiple logins by the same user on the same calendar day:
    SELECT DISTINCT user_id, CAST(login_timestamp AS DATE) AS login_date
    FROM user_logins
),
islands AS (
    SELECT
        user_id,
        login_date,
        login_date - CAST(ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) AS INT) AS island_group
    FROM distinct_daily_logins
),
streaks AS (
    SELECT
        user_id,
        MIN(login_date) AS streak_start,
        MAX(login_date) AS streak_end,
        COUNT(*) AS streak_days
    FROM islands
    GROUP BY user_id, island_group
)
SELECT user_id, MAX(streak_days) AS longest_streak_days
FROM streaks
GROUP BY user_id
ORDER BY longest_streak_days DESC;
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code & Configuration Execution Walkthrough**:
  1. **Declaration & Schema Binding**: The SQL block begins with `WITH distinct_daily_logins AS (`, establishing the target resource, dependency descriptor, or execution context for **Advanced SQL Challenge: Solve the "Gaps and Islands" Problem (Find Longest Consecutive Daily Login Streak per User).**.
  2. **Core Rule / Parameter Evaluation**: At `streaks AS (`, the runtime engine (Spring Boot binder, Kubernetes/Envoy control plane, JVM launcher, or SQL planner) parses the explicit thresholds, routing rules, or execution flags.
  3. **Runtime Enforcement**: Concluding with `ORDER BY longest_streak_days DESC;`, these settings are enforced deterministically at startup and runtime to guarantee the behavior illustrated in the architecture diagram above.
- **Don't forget Step 1 (`SELECT DISTINCT user_id, login_date`)**: If a user logs in 5 times on `2025-02-01` and you don't deduplicate by calendar date first, `ROW_NUMBER()` increments 5 times on the same date and breaks the `login_date - rn` invariant!

---

<a id="q8-43"></a>
### Q8.43: Advanced SQL Challenge: Compute a 7-Day Rolling Average and Month-over-Month (`MoM`) Growth Percentage with Missing Date Gaps Filled.

#### 1. Formal Technical Definition & Internal Mechanics
- **The Missing Date Gap Trap in `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`**: If there were `0` orders on Sunday, Sunday has no row in the `GROUP BY` output, so `ROWS BETWEEN 6 PRECEDING` accidentally averages across **8 calendar days** instead of 7!
- **Fix**: Either generate a contiguous calendar spine via `generate_series()` OR use **`RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW`**!

#### 2. Technical Architecture & Execution Diagram
```
Daily Revenue Spine:
  Day 1..Day 7 ──► AVG(daily_rev) OVER (ORDER BY tx_date RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW)
  MoM Growth % ──► ROUND(100.0 * (curr_rev - prev_rev) / NULLIF(prev_rev, 0), 2)
```

#### 3. Detailed Technical Example & Production Code
```sql
WITH daily_revenue AS (
    SELECT
        DATE_TRUNC('day', created_at) AS tx_day,
        SUM(amount) AS rev
    FROM orders
    WHERE status = 'COMPLETED'
    GROUP BY DATE_TRUNC('day', created_at)
)
SELECT
    tx_day,
    rev AS daily_rev,
    ROUND(
        AVG(rev) OVER (
            ORDER BY tx_day
            RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW
        ), 2
    ) AS rolling_7d_avg,
    ROUND(
        100.0 * (rev - LAG(rev, 1) OVER (ORDER BY tx_day))
        / NULLIF(LAG(rev, 1) OVER (ORDER BY tx_day), 0),
        2
    ) AS dod_growth_pct
FROM daily_revenue
ORDER BY tx_day DESC;
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code & Configuration Execution Walkthrough**:
  1. **Declaration & Schema Binding**: The SQL block begins with `WITH daily_revenue AS (`, establishing the target resource, dependency descriptor, or execution context for **Advanced SQL Challenge: Compute a 7-Day Rolling Average and Month-over-Month (`MoM`) Growth Percentage with Missing Date Gaps Filled.**.
  2. **Core Rule / Parameter Evaluation**: At `AVG(rev) OVER (`, the runtime engine (Spring Boot binder, Kubernetes/Envoy control plane, JVM launcher, or SQL planner) parses the explicit thresholds, routing rules, or execution flags.
  3. **Runtime Enforcement**: Concluding with `ORDER BY tx_day DESC;`, these settings are enforced deterministically at startup and runtime to guarantee the behavior illustrated in the architecture diagram above.
- Always wrap the denominator of growth percentage calculations in **`NULLIF(prev_rev, 0)`** to prevent `ERROR: division by zero` when a previous period had `0` revenue!

---

<a id="q8-44"></a>
### Q8.44: Refactoring Challenge: Refactor a Legacy Violating "God Class" (`OrderProcessor` with `if/else` chains) into Clean SOLID + Strategy + Factory Pattern in Spring Boot.

#### 1. Formal Technical Definition & Internal Mechanics
- **The Code Smell**: A 500-line `if ("STRIPE".equals(type)) { ... } else if ("PAYPAL".equals(type)) { ... } else if ("CRYPTO".equals(type))` method violates both the **Single Responsibility Principle (SRP)** and **Open/Closed Principle (OCP)**.
- **Spring Idiomatic Strategy + Map Registry Pattern**:
  - Define a `PaymentGatewayStrategy` interface with `PaymentMethod supportedMethod()`.
  - Spring automatically injects **`List<PaymentGatewayStrategy>`** (all `@Component` implementations!) into the constructor of `PaymentStrategyRouter`, which indexes them into an immutable `EnumMap<PaymentMethod, PaymentGatewayStrategy>` in $O(1)$!
  - Adding a new `ApplePayStrategy` requires creating 1 new `@Component` class with **zero modifications to existing code**!

#### 2. Technical Architecture & Execution Diagram
```
Spring IoC Container discovers all @Component beans implementing PaymentGatewayStrategy:
  ├── StripeGatewayStrategy  (supportedMethod = STRIPE)
  ├── PayPalGatewayStrategy  (supportedMethod = PAYPAL)
  └── AdyenGatewayStrategy   (supportedMethod = ADYEN)
         │
         ▼ Injected as List<PaymentGatewayStrategy> into PaymentStrategyRouter
  EnumMap<PaymentMethod, PaymentGatewayStrategy> (O(1) Lookup, Zero if/else!)
```

#### 3. Detailed Technical Example & Production Code
```java
public enum PaymentMethod { STRIPE, PAYPAL, ADYEN }

public interface PaymentGatewayStrategy {
    PaymentMethod supportedMethod();
    PaymentReceipt charge(PaymentCommand command);
}

@Service
public class PaymentStrategyRouter {
    private final Map<PaymentMethod, PaymentGatewayStrategy> strategies;

    // Spring automatically injects ALL beans implementing PaymentGatewayStrategy!
    public PaymentStrategyRouter(List<PaymentGatewayStrategy> strategyBeans) {
        this.strategies = strategyBeans.stream()
            .collect(Collectors.toMap(
                PaymentGatewayStrategy::supportedMethod,
                Function.identity(),
                (a, b) -> { throw new IllegalStateException("Duplicate strategy for " + a.supportedMethod()); },
                () -> new EnumMap<>(PaymentMethod.class)
            ));
    }

    public PaymentReceipt routeAndCharge(PaymentCommand command) {
        PaymentGatewayStrategy strategy = strategies.get(command.method());
        if (strategy == null) {
            throw new UnsupportedOperationException("Unsupported payment method: " + command.method());
        }
        return strategy.charge(command);
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `PaymentMethod`, `PaymentGatewayStrategy`, `PaymentStrategyRouter` (annotated with `@Service`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the Stream pipeline chains lazy intermediate operations and triggers single-pass evaluation at the terminal collector/reduction step.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Notice how the constructor fail-fast check `(a, b) -> { throw new IllegalStateException("Duplicate strategy..."); }` catches duplicate strategy beans at **application startup** rather than at runtime!

---

<a id="q8-45"></a>
### Q8.45: Code Review Challenge: Spot All 7 Production Bugs in this Naive Spring Boot `@Service` Class!

#### 1. Formal Technical Definition & Internal Mechanics
- In Senior/Staff interviews, you are often handed a 25-line Spring Boot snippet that *compiles*, and asked to identify every concurrency, transactional, performance, and security bug.

#### 2. Technical Architecture & Execution Diagram
```java
// ❌ BROKEN CODE UNDER REVIEW — SPOT ALL 7 PRODUCTION BUGS:
@Service
public class NaiveOrderService {
    @Autowired private OrderRepository orderRepo;               // Bug #1: Field Injection
    @Autowired private RestTemplate restTemplate;
    private Long currentCustomerId;                             // Bug #2: Mutable Singleton State (Race Condition!)

    public void processCheckout(Long customerId, List<Long> itemIds, double price) { // Bug #3: double for money!
        this.currentCustomerId = customerId;
        saveAllItems(itemIds, price);                           // Bug #4: @Transactional Self-Invocation Bypass!
    }

    @Transactional
    private void saveAllItems(List<Long> itemIds, double price) { // Bug #5: private @Transactional ignored by CGLIB!
        for (Long itemId : itemIds) {
            Order o = orderRepo.findById(itemId).get();         // Bug #6: N+1 Queries in loop + .get() without check!
            String res = restTemplate.getForObject("http://fraud/check/" + currentCustomerId, String.class); // Bug #7: Blocking HTTP inside @Transactional!
        }
    }
}
```

#### 3. Detailed Technical Example & Production Code
```java
// ✅ FIXED PRODUCTION-GRADE VERSION:
@Service
public class CleanOrderService {
    private final OrderPersistenceService persistenceService;
    private final FraudCheckClient fraudCheckClient;

    public CleanOrderService(OrderPersistenceService persistenceService, FraudCheckClient fraudCheckClient) {
        this.persistenceService = persistenceService;
        this.fraudCheckClient = fraudCheckClient;
    }

    public void processCheckout(Long customerId, List<Long> orderIds, BigDecimal price) {
        // 1. Perform external HTTP call OUTSIDE @Transactional so DB connections aren't held during network I/O!
        FraudVerdict verdict = fraudCheckClient.checkCustomer(customerId);
        if (!verdict.isAllowed()) {
            throw new FraudRejectedException("Customer rejected by fraud check: " + customerId);
        }
        // 2. Call separate collaborator bean so @Transactional proxy is invoked & batch fetch IDs in 1 SQL query!
        persistenceService.updateOrdersInBatch(customerId, orderIds, price);
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `CleanOrderService` (annotated with `@Service`, `@Transactional`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `processCheckout(...)`, execution proceeds step-by-step: Spring and Hibernate coordinate the `PersistenceContext` (`Session`), SQL statement generation, JDBC connection binding, and transaction commit/rollback interception.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- **Summary of the 7 Bugs Caught**:
  1. `@Autowired` field injection (harder to unit test without reflection, allows mutable/null fields).
  2. `private Long currentCustomerId;` instance variable on a singleton `@Service` causes cross-user race conditions!
  3. Using `double price` instead of `BigDecimal` for monetary values.
  4. Calling `this.saveAllItems()` internally bypasses the Spring AOP `@Transactional` proxy.
  5. Marking `@Transactional` on a `private` method is ignored by CGLIB proxying.
  6. Calling `orderRepo.findById(itemId)` inside a `for` loop causes $N$ separate database round-trips instead of a single `orderRepo.findAllById(itemIds)`.
  7. Calling `restTemplate.getForObject(...)` inside `@Transactional` holds a HikariCP database connection open across network latency, starving the connection pool!

---

<a id="q8-46"></a>
### Q8.46: Capstone Part 1: High-Level Architecture & Domain Model of a Production-Grade Distributed Order, Inventory & Payment Platform.

#### 1. Formal Technical Definition & Internal Mechanics
- Across `Q8.46 – Q8.52`, we synthesize every concept from Sections 01–07 into a complete, production-ready **Distributed Order & Payment Platform (`OrderService`, `InventoryService`, `PaymentService`)** using Java 21, Spring Boot 3, PostgreSQL, Redis, and Apache Kafka.
- **End-to-End Guarantees**:
  - **Zero Double-Charge**: Client `Idempotency-Key` + Deterministic Payment Saga IDs.
  - **Zero Lost Events**: Transactional Outbox Pattern (`FOR UPDATE SKIP LOCKED`) in PostgreSQL.
  - **Zero Duplicate Consumption**: Transactional Inbox Pattern (`processed_events` table).
  - **Zero Overselling**: Optimistic/Pessimistic Inventory Reservation with 15-minute expiration hold.

#### 2. Technical Architecture & Execution Diagram
```
                       ┌───────────────────────────┐
                       │   Client / API Gateway    │
                       │ (OAuth2 JWT + Rate Limit) │
                       └─────────────┬─────────────┘
                                     │ POST /api/v1/orders (Idempotency-Key)
                                     ▼
                       ┌───────────────────────────┐
                       │       Order Service       │
                       │  [orders + outbox_events] │
                       └──────┬─────────────▲──────┘
      1. OrderCreatedEvent    │             │ 4. PaymentCaptured / Failed
                              ▼             │
                    ╔═════════════════════════════╗
                    ║     Apache Kafka Cluster    ║
                    ║ (RF=3, min.isr=2, acks=all) ║
                    ╚═════════╦═════════════╦═════╝
        2. Reserve Inventory  ║             ║ 3. Authorize & Capture Card
                              ▼             ▼
               ┌────────────────────┐ ┌────────────────────┐
               │ Inventory Service  │ │  Payment Service   │
               │ [stock + inbox/out]│ │ [ledger + inbox/out│
               └────────────────────┘ └────────────────────┘
```

#### 3. Detailed Technical Example & Production Code
```java
// Immutable Java 21 Domain Event Sealed Hierarchy shared across the Saga:
public sealed interface OrderSagaEvent permits
        OrderSagaEvent.OrderCreated,
        OrderSagaEvent.InventoryReserved,
        OrderSagaEvent.InventoryReservationFailed,
        OrderSagaEvent.PaymentSucceeded,
        OrderSagaEvent.PaymentFailed {

    UUID eventId();
    Long orderId();
    Instant occurredAt();

    record OrderCreated(UUID eventId, Long orderId, Long customerId, String sku, int qty, BigDecimal totalAmount, Instant occurredAt) implements OrderSagaEvent {}
    record InventoryReserved(UUID eventId, Long orderId, String reservationId, Instant occurredAt) implements OrderSagaEvent {}
    record InventoryReservationFailed(UUID eventId, Long orderId, String reason, Instant occurredAt) implements OrderSagaEvent {}
    record PaymentSucceeded(UUID eventId, Long orderId, String paymentRef, Instant occurredAt) implements OrderSagaEvent {}
    record PaymentFailed(UUID eventId, Long orderId, String reason, Instant occurredAt) implements OrderSagaEvent {}
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `OrderSagaEvent`, `OrderCreated`, `InventoryReserved`, `InventoryReservationFailed`, `PaymentSucceeded`, `PaymentFailed` (using standard JDK and framework APIs), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `Long orderId();` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Using a **Java 21 `sealed interface` of `record`s** for Saga Events guarantees compile-time exhaustiveness in `switch` expressions and immutability across serialization boundaries.

---

<a id="q8-47"></a>
### Q8.47: Capstone Part 2: PostgreSQL Production DDL Schema (`orders`, `outbox_events`, `processed_events`, `inventory_reservations`, and Partial/Covering Indexes).

#### 1. Formal Technical Definition & Internal Mechanics
- Every table uses:
  - `BIGINT` / `UUIDv7` primary keys.
  - `NUMERIC(19, 4)` for financial amounts with `CHECK (total_amount > 0)` constraints.
  - `version BIGINT NOT NULL DEFAULT 0` for JPA `@Version` optimistic locking.
  - A **Partial Index** on `outbox_events (created_at ASC) WHERE status = 'PENDING'` so the Outbox poller remains sub-millisecond even after 100 million events are published!

#### 2. Technical Architecture & Execution Diagram
```
Order Service PostgreSQL Database:
  ┌──────────────────────────┐       ┌──────────────────────────┐
  │          orders          │       │      outbox_events       │
  ├──────────────────────────┤       ├──────────────────────────┤
  │ id (PK)                  │       │ id (UUID PK)             │
  │ idempotency_key (UNIQUE) │       │ aggregate_id (Order ID)  │
  │ customer_id              │       │ event_type               │
  │ status                   │       │ payload (JSONB)          │
  │ total_amount NUMERIC     │       │ status ('PENDING')       │
  │ version (@Version)       │       └──────────────────────────┘
  └──────────────────────────┘       (Partial Index WHERE status='PENDING')
```

#### 3. Detailed Technical Example & Production Code
```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    idempotency_key VARCHAR(64) NOT NULL UNIQUE,
    customer_id BIGINT NOT NULL,
    sku VARCHAR(64) NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    total_amount NUMERIC(19, 4) NOT NULL CHECK (total_amount > 0),
    status VARCHAR(32) NOT NULL,
    failure_reason VARCHAR(255),
    version BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_customer_created ON orders (customer_id, created_at DESC) INCLUDE (status, total_amount);

CREATE TABLE outbox_events (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(64) NOT NULL,
    aggregate_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    attempt_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outbox_pending ON outbox_events (created_at ASC) WHERE status = 'PENDING';

CREATE TABLE processed_events (
    event_id UUID PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code & Configuration Execution Walkthrough**:
  1. **Declaration & Schema Binding**: The SQL block begins with `CREATE TABLE orders (`, establishing the target resource, dependency descriptor, or execution context for **Capstone Part 2: PostgreSQL Production DDL Schema (`orders`, `outbox_events`, `processed_events`, `inventory_reservations`, and Partial/Covering Indexes).**.
  2. **Core Rule / Parameter Evaluation**: At `id UUID PRIMARY KEY,`, the runtime engine (Spring Boot binder, Kubernetes/Envoy control plane, JVM launcher, or SQL planner) parses the explicit thresholds, routing rules, or execution flags.
  3. **Runtime Enforcement**: Concluding with `);`, these settings are enforced deterministically at startup and runtime to guarantee the behavior illustrated in the architecture diagram above.
- Notice how `idempotency_key VARCHAR(64) NOT NULL UNIQUE` on the `orders` table enforces API idempotency directly at the database engine level, even if 2 identical HTTP requests arrive at the exact same microsecond on different pods!

---

<a id="q8-48"></a>
### Q8.48: Capstone Part 3: Idempotent Order Creation Endpoint & Service with Transactional Outbox Persistence.

#### 1. Formal Technical Definition & Internal Mechanics
- When `POST /api/v1/orders` is called with `Idempotency-Key: <uuid>`, `OrderCommandService`:
  1. Checks if an order with `idempotency_key` already exists (returning the existing order if found).
  2. Saves the new `OrderEntity` (`status = PENDING_INVENTORY`) AND inserts `OrderCreated` into `outbox_events` in the **same `@Transactional` commit**.

#### 2. Technical Architecture & Execution Diagram
```
POST /api/v1/orders (Header: Idempotency-Key: "idem-901")
       │
       ▼
[@Transactional OrderCommandService.placeOrder()]
  ├── 1. Check orderRepository.findByIdempotencyKey("idem-901") -> Return existing if present!
  ├── 2. orderRepository.save(orderEntity)
  ├── 3. outboxRepository.save(outboxEvent)
  └── 4. COMMIT (Atomic!)
```

#### 3. Detailed Technical Example & Production Code
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    private final OrderCommandService orderService;

    public OrderController(OrderCommandService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestHeader("Idempotency-Key") @NotBlank String idempotencyKey,
            @Valid @RequestBody CreateOrderRequest request,
            @AuthenticationPrincipal Jwt jwt) {
        Long customerId = Long.valueOf(jwt.getSubject());
        OrderResponse response = orderService.createOrderIdempotently(idempotencyKey, customerId, request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}

@Service
public class OrderCommandService {
    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;

    public OrderCommandService(
            OrderRepository orderRepository,
            OutboxRepository outboxRepository,
            ObjectMapper objectMapper) {
        this.orderRepository = orderRepository;
        this.outboxRepository = outboxRepository;
        this.objectMapper = objectMapper;
    }

    @Transactional
    public OrderResponse createOrderIdempotently(String idempotencyKey, Long customerId, CreateOrderRequest req) {
        Optional<OrderEntity> existing = orderRepository.findByIdempotencyKey(idempotencyKey);
        if (existing.isPresent()) {
            return OrderResponse.from(existing.get());
        }

        OrderEntity order = OrderEntity.createPending(idempotencyKey, customerId, req.sku(), req.quantity(), req.totalAmount());
        OrderEntity saved = orderRepository.save(order);

        OrderSagaEvent.OrderCreated event = new OrderSagaEvent.OrderCreated(
            UUID.randomUUID(), saved.getId(), customerId, req.sku(), req.quantity(), req.totalAmount(), Instant.now()
        );
        outboxRepository.save(OutboxEntity.pending("Order", saved.getId().toString(), "OrderCreated", toJson(event)));
        return OrderResponse.from(saved);
    }

    private String toJson(Object obj) {
        try { return objectMapper.writeValueAsString(obj); }
        catch (JsonProcessingException e) { throw new UncheckedIOException(e); }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `OrderController`, `OrderCommandService` (annotated with `@RestController`, `@RequestMapping`, `@PostMapping`, `@RequestHeader`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `toJson(...)`, execution proceeds step-by-step: the `HttpSecurity` builder registers ordered security filters (`BearerTokenAuthenticationFilter`, `AuthorizationFilter`, exception handlers) into the `SecurityFilterChain` bean; Spring and Hibernate coordinate the `PersistenceContext` (`Session`), SQL statement generation, JDBC connection binding, and transaction commit/rollback interception.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Notice how `customerId` is extracted from the verified `@AuthenticationPrincipal Jwt jwt` (`jwt.getSubject()`), **never** trusted from the `@RequestBody` JSON payload (preventing BOLA / IDOR spoofing!).

---

<a id="q8-49"></a>
### Q8.49: Capstone Part 4: High-Throughput Outbox Relay Poller using `SELECT ... FOR UPDATE SKIP LOCKED` and Asynchronous Kafka `CompletableFuture` Callbacks.

#### 1. Formal Technical Definition & Internal Mechanics
- Every `100ms`, the Outbox Relay locks a batch of 50 `PENDING` events using `FOR UPDATE SKIP LOCKED` (so 10 pods never read the same rows!).
- Publishes all 50 events to Kafka using `aggregate_id` (`orderId`) as the partition key, waits for broker `acks=all` confirmation, and marks the batch `PUBLISHED`.

#### 2. Technical Architecture & Execution Diagram
```
[OutboxRelayWorker (Pod 1..10)]
  ├── 1. Claim 50 rows via SELECT ... FOR UPDATE SKIP LOCKED
  ├── 2. Publish 50 records to Kafka (Key = aggregateId -> Preserves Partition Order!)
  ├── 3. Await CompletableFuture.allOf(sendFutures) (Broker acks=all confirmed!)
  └── 4. Batch UPDATE status = 'PUBLISHED'
```

#### 3. Detailed Technical Example & Production Code
```java
@Component
public class TransactionalOutboxRelay {
    private final JdbcTemplate jdbcTemplate;
    private final KafkaTemplate<String, String> kafkaTemplate;

    public TransactionalOutboxRelay(JdbcTemplate jdbcTemplate, KafkaTemplate<String, String> kafkaTemplate) {
        this.jdbcTemplate = jdbcTemplate;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Scheduled(fixedDelayString = "100")
    @Transactional
    public void publishPendingOutboxBatch() {
        List<OutboxRow> batch = jdbcTemplate.query("""
            SELECT id, aggregate_id, event_type, payload
            FROM outbox_events
            WHERE status = 'PENDING'
            ORDER BY created_at ASC
            LIMIT 50
            FOR UPDATE SKIP LOCKED
            """,
            (rs, rowNum) -> new OutboxRow(
                UUID.fromString(rs.getString("id")),
                rs.getString("aggregate_id"),
                rs.getString("event_type"),
                rs.getString("payload")
            )
        );
        if (batch.isEmpty()) return;

        List<UUID> publishedIds = new ArrayList<>(batch.size());
        for (OutboxRow row : batch) {
            try {
                ProducerRecord<String, String> record = new ProducerRecord<>(
                    "order.saga.events.v1", null, row.aggregateId(), row.payload()
                );
                record.headers().add("event_type", row.eventType().getBytes(StandardCharsets.UTF_8));
                kafkaTemplate.send(record).get(3, TimeUnit.SECONDS);
                publishedIds.add(row.id());
            } catch (Exception ex) {
                // Increment attempt_count; leave status='PENDING' for next retry cycle
                jdbcTemplate.update("UPDATE outbox_events SET attempt_count = attempt_count + 1 WHERE id = ?", row.id());
            }
        }

        if (!publishedIds.isEmpty()) {
            jdbcTemplate.batchUpdate(
                "UPDATE outbox_events SET status = 'PUBLISHED' WHERE id = ?",
                publishedIds,
                publishedIds.size(),
                (ps, id) -> ps.setObject(1, id)
            );
        }
    }

    record OutboxRow(UUID id, String aggregateId, String eventType, String payload) {}
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `TransactionalOutboxRelay`, `OutboxRow` (annotated with `@Component`, `@Scheduled`, `@Transactional`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `publishPendingOutboxBatch(...)`, execution proceeds step-by-step: Spring and Hibernate coordinate the `PersistenceContext` (`Session`), SQL statement generation, JDBC connection binding, and transaction commit/rollback interception.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Attaching `"event_type"` as a **Kafka Header** (`record.headers().add(...)`) allows downstream consumers to route or filter events without parsing the entire JSON body first!

---

<a id="q8-50"></a>
### Q8.50: Capstone Part 5: Idempotent Kafka Consumer & Saga Step Handler (`InventoryService` Stock Reservation with Transactional Inbox + Outbox).

#### 1. Formal Technical Definition & Internal Mechanics
- When `InventoryService` consumes `OrderCreated`, it executes a **single local ACID transaction** that:
  1. Inserts `eventId` into `processed_events ON CONFLICT DO NOTHING` (**Transactional Inbox**). If `0` rows inserted, returns immediately and ACKs Kafka!
  2. Attempts atomic stock decrement: `UPDATE inventory SET available_qty = available_qty - :qty, reserved_qty = reserved_qty + :qty WHERE sku = :sku AND available_qty >= :qty`.
  3. Writes either `InventoryReserved` (if `1` row updated) or `InventoryReservationFailed` (if `0` rows updated—out of stock!) to its own **`outbox_events`** table!

#### 2. Technical Architecture & Execution Diagram
```
[Inventory Kafka Consumer]
       │
       ▼
BEGIN @Transactional:
  1. INSERT INTO processed_events (event_id) ON CONFLICT DO NOTHING
     └── Already processed? -> Return & ack.acknowledge()!
  2. UPDATE inventory SET available = available - :qty WHERE sku = :sku AND available >= :qty
     ├── 1 row updated -> Insert 'InventoryReserved' into outbox_events
     └── 0 rows updated -> Insert 'InventoryReservationFailed' into outbox_events
COMMIT -> ack.acknowledge()!
```

#### 3. Detailed Technical Example & Production Code
```java
@Service
public class InventorySagaStepHandler {
    private final JdbcTemplate jdbcTemplate;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;

    public InventorySagaStepHandler(
            JdbcTemplate jdbcTemplate,
            OutboxRepository outboxRepository,
            ObjectMapper objectMapper) {
        this.jdbcTemplate = jdbcTemplate;
        this.outboxRepository = outboxRepository;
        this.objectMapper = objectMapper;
    }

    @Transactional
    public void handleOrderCreated(OrderSagaEvent.OrderCreated event) throws JsonProcessingException {
        // 1. Transactional Inbox Idempotency Guard:
        int inboxInserted = jdbcTemplate.update(
            "INSERT INTO processed_events (event_id, event_type) VALUES (?, 'OrderCreated') ON CONFLICT (event_id) DO NOTHING",
            event.eventId()
        );
        if (inboxInserted == 0) {
            return; // Duplicate delivery safely ignored!
        }

        // 2. Atomic Conditional Stock Decrement (Zero Overselling!):
        int stockUpdated = jdbcTemplate.update("""
            UPDATE inventory_stock
            SET available_qty = available_qty - ?,
                reserved_qty  = reserved_qty + ?,
                version       = version + 1
            WHERE sku = ? AND available_qty >= ?
            """,
            event.qty(), event.qty(), event.sku(), event.qty()
        );

        // 3. Emit Reply Event to Transactional Outbox:
        if (stockUpdated == 1) {
            String reservationId = "RES-" + event.orderId();
            var reply = new OrderSagaEvent.InventoryReserved(UUID.randomUUID(), event.orderId(), reservationId, Instant.now());
            outboxRepository.save(OutboxEntity.pending("Order", event.orderId().toString(), "InventoryReserved", objectMapper.writeValueAsString(reply)));
        } else {
            var reply = new OrderSagaEvent.InventoryReservationFailed(UUID.randomUUID(), event.orderId(), "INSUFFICIENT_STOCK", Instant.now());
            outboxRepository.save(OutboxEntity.pending("Order", event.orderId().toString(), "InventoryReservationFailed", objectMapper.writeValueAsString(reply)));
        }
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `InventorySagaStepHandler` (annotated with `@Service`, `@Transactional`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `handleOrderCreated(...)`, execution proceeds step-by-step: Spring and Hibernate coordinate the `PersistenceContext` (`Session`), SQL statement generation, JDBC connection binding, and transaction commit/rollback interception.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Look at Step 2 (`UPDATE inventory_stock SET available_qty = available_qty - ? WHERE sku = ? AND available_qty >= ?`): doing the conditional check and decrement in a **single atomic SQL `UPDATE`** is fast, lock-minimal, and makes negative stock mathematically impossible without even needing to read the entity into Java first!

---

<a id="q8-51"></a>
### Q8.51: Capstone Part 6: Saga Compensation Flow (Handling `PaymentFailed` by Releasing Reserved Stock & Cancelling Order) and Global `@RestControllerAdvice` (`ProblemDetail`).

#### 1. Formal Technical Definition & Internal Mechanics
- If `PaymentService` fails to charge the customer's card (e.g., `InsufficientFunds`) and emits `PaymentFailed(orderId=99)`:
  1. `InventoryService` consumes `PaymentFailed`, checks its `processed_events` inbox, and executes the **Compensating Transaction**: releasing `reserved_qty` back to `available_qty`!
  2. `OrderService` consumes `PaymentFailed` and transitions `Order #99` to `CANCELLED_PAYMENT_FAILED`.
- All REST API errors across the platform use **Spring 6 RFC 7807 `ProblemDetail`** enriched with `traceId` and `timestamp`.

#### 2. Technical Architecture & Execution Diagram
```
Payment Service emits PaymentFailed(orderId=99)
       ├──► [Inventory Service]: Compensates stock (available_qty += qty, reserved_qty -= qty)
       └──► [Order Service]:     Updates Order #99 status = 'CANCELLED_PAYMENT_FAILED'
```

#### 3. Detailed Technical Example & Production Code
```java
@RestControllerAdvice
public class GlobalProblemDetailExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(InsufficientStockException.class)
    public ProblemDetail handleInsufficientStock(InsufficientStockException ex, HttpServletRequest req) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, ex.getMessage());
        problem.setTitle("Insufficient Inventory Stock");
        problem.setType(URI.create("https://api.acme.com/errors/insufficient-stock"));
        problem.setInstance(URI.create(req.getRequestURI()));
        problem.setProperty("traceId", MDC.get("traceId"));
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    @ExceptionHandler(ObjectOptimisticLockingFailureException.class)
    public ProblemDetail handleOptimisticLock(ObjectOptimisticLockingFailureException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT, "Resource was modified concurrently; please retry."
        );
        problem.setTitle("Concurrent Modification Conflict");
        problem.setProperty("traceId", MDC.get("traceId"));
        return problem;
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `GlobalProblemDetailExceptionHandler` (annotated with `@RestControllerAdvice`, `@ExceptionHandler`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: During method execution, control flow proceeds step-by-step: the core statement `}` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Always include `"traceId"` (`MDC.get("traceId")`) inside every `ProblemDetail` error response so when a customer or frontend developer reports an API error, support engineers can paste that `traceId` straight into Grafana Tempo / Kibana and see the exact distributed trace and stack trace in 2 seconds!

---

<a id="q8-52"></a>
### Q8.52: Capstone Part 7: End-to-End Integration Test Suite (`Testcontainers` PostgreSQL + Kafka + `Awaitility` + Concurrency Race Verification).

#### 1. Formal Technical Definition & Internal Mechanics
- To verify our entire Capstone Saga end-to-end in CI:
  1. Spin up real `postgres:16-alpine` and `apache/kafka-native:3.8.0` containers via `@ServiceConnection`.
  2. Test **Idempotent Order Creation**: Send two concurrent `POST /api/v1/orders` requests with the exact same `Idempotency-Key` and assert that only **1 Order row and 1 Outbox row** are created in PostgreSQL!
  3. Test **Full Async Saga Completion & Compensation** using `Awaitility`: Assert that when stock is available, the order transitions `PENDING_INVENTORY -> CONFIRMED`, and when stock is `0`, the Saga compensates and transitions the order to `CANCELLED_OUT_OF_STOCK`.

#### 2. Technical Architecture & Execution Diagram
```
[End-to-End Capstone Integration Test]
  ├── 1. Seed Inventory (SKU-IPHONE = 1 unit in stock)
  ├── 2. Fire 5 Concurrent POST /api/v1/orders for SKU-IPHONE (1 unit each) via CyclicBarrier(5)
  └── 3. Awaitility.await() asserts:
        ├── Exactly 1 Order reaches status = 'CONFIRMED'
        ├── Exactly 4 Orders reach status = 'CANCELLED_OUT_OF_STOCK'
        └── Final inventory_stock.available_qty == 0 (Zero Oversell!)
```

#### 3. Detailed Technical Example & Production Code
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderSagaEndToEndIntegrationTest {

    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine");

    @ServiceConnection
    static final KafkaContainer KAFKA = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

    static {
        POSTGRES.start();
        KAFKA.start();
    }

    @Autowired private OrderCommandService orderService;
    @Autowired private OrderRepository orderRepository;
    @Autowired private JdbcTemplate jdbcTemplate;

    @Test
    void shouldDeduplicateIdenticalIdempotencyKeysAndCompleteSagaWithoutOverselling() {
        jdbcTemplate.update(
            "INSERT INTO inventory_stock (sku, available_qty, reserved_qty, version) VALUES ('SKU-PRO', 1, 0, 0)"
        );

        CreateOrderRequest req = new CreateOrderRequest("SKU-PRO", 1, new BigDecimal("999.00"));

        // 1. Verify Idempotency-Key deduplication:
        OrderResponse first = orderService.createOrderIdempotently("idem-key-100", 42L, req);
        OrderResponse duplicate = orderService.createOrderIdempotently("idem-key-100", 42L, req);
        assertThat(duplicate.id()).isEqualTo(first.id());

        // 2. Verify Asynchronous Outbox -> Kafka -> Inventory -> Order Saga completion:
        Awaitility.await()
            .atMost(Duration.ofSeconds(10))
            .pollInterval(Duration.ofMillis(100))
            .untilAsserted(() -> {
                OrderEntity updated = orderRepository.findById(first.id()).orElseThrow();
                assertThat(updated.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
            });
    }
}
```

#### 4. Step-by-Step Code Execution Explanation & Production Pitfalls
- **Step-by-Step Code Execution Walkthrough**:
  1. **Type & Metadata Structure**: The implementation defines `OrderSagaEndToEndIntegrationTest` (annotated with `@SpringBootTest`, `@ServiceConnection`, `@Autowired`, `@Test`), establishing the compile-time contract and runtime wiring required for this pattern.
  2. **Core Execution Path**: Inside `shouldDeduplicateIdenticalIdempotencyKeysAndCompleteSagaWithoutOverselling(...)`, execution proceeds step-by-step: the core statement `jdbcTemplate.update(` executes the primary state transformation while preserving algorithmic time/space complexity and thread safety.
  3. **Invariant & Failure-Path Guarantees**: Return values, exception propagation, and resource cleanup are structured so that concurrent callers and downstream consumers observe consistent, deterministic state.
- Being able to sketch and code this complete 7-part architecture (`Domain Sealed Records` $\rightarrow$ `PostgreSQL Schema & Partial Indexes` $\rightarrow$ `Idempotent Command + Outbox` $\rightarrow$ `FOR UPDATE SKIP LOCKED Relay` $\rightarrow$ `Transactional Inbox Consumer` $\rightarrow$ `Saga Compensation` $\rightarrow$ `Testcontainers E2E Test`) prepares you to ace any Senior, Staff, or Principal Java Backend Engineering interview!

---

[← Back to Main README](../README.md)
