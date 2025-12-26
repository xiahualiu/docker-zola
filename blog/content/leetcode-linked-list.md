+++
title = "LeetCode Linked List Question Summary"
date = 2024-11-09
draft = false
[taxonomies]
  tags = ["LeetCode", "C++", "Algorithm", "Linked List"]
[extra]
  toc = true
  keywords = "LeetCode, C++, Algorithm, Linked List"
+++

After solving around 20 LeetCode linked-list problems, here is a concise summary of patterns and templates.

## Type Definition

The standard singly linked list definition used by LeetCode is:

```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode() : val(0), next(nullptr) {}
    ListNode(int _val) : val(_val), next(nullptr) {}
    ListNode(int _val, ListNode* _next) : val(_val), next(_next) {}
};
```

## Linked List Traversal

Unlike arrays, you cannot access random elements in a linked list. You must traverse it sequentially. The direction of processing determines the logic.

### 1. Forward Traversal (Pre-Order)

We process the current node before moving to the next.

**Iterative style** (most common):

```cpp
for (auto p = head; p != nullptr; p = p->next) {
    // Process p
}
```

If you need to manipulate links (delete a node or reverse a link), keep track of the previous node:

```cpp
ListNode* prev = nullptr;
for (auto p = head; p != nullptr; p = p->next) {
    // Process prev and p
    prev = p;
}
```

**Recursive style:** performing action before the recursive call:

```cpp
void traverse(ListNode* cur) {
    if (!cur) return;
    // Process cur
    cout << cur->val << endl;
    traverse(cur->next);
}
```

If you need the `prev` pointer in recursion, pass it as an argument:

```cpp
void traverse(ListNode* prev, ListNode* cur) {
    if (!cur) return;
    // Process prev and cur
    traverse(cur, cur->next);
}

// Initial call
// traverse(nullptr, head);
```

### 2. Backward Traversal (Post-Order)

Process nodes in reverse order (tail → head). Use a stack or recursion.

**Iterative (using stack)** — uses O(N) extra space:

```cpp
stack<ListNode*> st;
for (auto p = head; p != nullptr; p = p->next) st.push(p);
while (!st.empty()) {
    ListNode* cur = st.top(); st.pop();
    // Process cur (Tail -> Head)
}
```

**Recursive (post-order):** perform action after the recursive call returns:

```cpp
void traverse(ListNode* cur) {
    if (!cur) return;
    traverse(cur->next);
    // Process cur as recursion unwinds
    cout << cur->val << endl;
}
```

## Example: Reverse Linked List (LeetCode 206)

Several common methods to reverse a singly linked list.

**Method 1 — Iterative (forward logic):**

```cpp
ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;
    ListNode* cur = head;
    while (cur != nullptr) {
        ListNode* nextTemp = cur->next; // Save next
        cur->next = prev;               // Reverse link
        prev = cur;                     // Move prev
        cur = nextTemp;                 // Move cur
    }
    return prev; // New head
}
```

**Method 2 — Recursive (tail recursion / forward logic):**

```cpp
ListNode* reverseRecur(ListNode* prev, ListNode* cur) {
    if (!cur) return prev;
    ListNode* nextTemp = cur->next;
    cur->next = prev;
    return reverseRecur(cur, nextTemp);
}

ListNode* reverseList(ListNode* head) {
    return reverseRecur(nullptr, head);
}
```

**Method 3 — Recursive (standard post-order):**

```cpp
ListNode* reverseList(ListNode* head) {
    if (head == nullptr || head->next == nullptr) return head;
    ListNode* newHead = reverseList(head->next);
    head->next->next = head;
    head->next = nullptr;
    return newHead;
}
```

## Important Patterns

### 1. Dummy Node

When modifying list structure (insert/delete/merge), the head may change. Use a **dummy node** to simplify edge cases:

```cpp
ListNode* dummy = new ListNode(0);
dummy->next = head;
ListNode* cur = dummy;
// ... perform operations using cur ...
return dummy->next; // Actual head
```

### 2. Fast & Slow Pointers (Tortoise & Hare)

Useful for detecting cycles and finding the middle of the list. **Slow** moves 1 step; **Fast** moves 2 steps.

```cpp
ListNode* slow = head;
ListNode* fast = head;
while (fast != nullptr && fast->next != nullptr) {
    slow = slow->next;
    fast = fast->next->next;
    if (slow == fast) {
        // Cycle detected
        break;
    }
}
// If no cycle, 'slow' is at the middle when loop ends
```
