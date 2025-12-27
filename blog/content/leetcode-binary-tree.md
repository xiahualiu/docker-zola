+++
title = "LeetCode Binary Tree Question Summary"
date = 2024-11-09
draft = false
[taxonomies]
  tags = ["LeetCode", "C++", "Algorithm"]
[extra]
  math = true
  math_auto_render = true
  toc = true
	keywords = "LeetCode, C++, Algorithm"
+++

## Type Definition

In LeetCode problems, the binary tree node is standardly defined as:

```cpp
struct TreeNode {
    int val;
    TreeNode* left;
    TreeNode* right;
    TreeNode() : val(0), left(nullptr), right(nullptr) {}
    TreeNode(int _val) : val(_val), left(nullptr), right(nullptr) {}
    TreeNode(int _val, TreeNode* _left, TreeNode* _right)
        : val(_val), left(_left), right(_right) {}
};
```

## Binary Tree Traversal

Tree traversal is the foundation of almost every tree problem. There are three standard Depth-First Search (DFS) orders.

### Pre-Order Traversal

**Order:** Root $\rightarrow$ Left $\rightarrow$ Right.

#### Iterative Style

**Note**: We use a `stack` here. Because a stack is LIFO (Last In, First Out), we push the right child first so that the left child is popped and processed first.

```cpp
void pre_order(TreeNode* root) {
    if (!root) return;
    
    stack<TreeNode*> st;
    st.push(root);
    
    while (!st.empty()) {
        TreeNode* cur = st.top();
        st.pop();
        
        // 1. Process Root
        cout << cur->val << endl;
        
        // 2. Push Right (so it is processed last)
        if (cur->right != nullptr) {
            st.push(cur->right);
        }
        // 3. Push Left (so it is processed next)
        if (cur->left != nullptr) {
            st.push(cur->left);
        }
    }
}
```

#### Recursive Style

```cpp
void pre_order(TreeNode* cur) {
    if (cur == nullptr) {
        return;
    }
    // 1. Process Root
    cout << cur->val << endl; 
    // 2. Go Left
    pre_order(cur->left);
    // 3. Go Right
    pre_order(cur->right);
}
```

### In-Order Traversal

**Order:** Left $\rightarrow$ Root $\rightarrow$ Right.

This order is critical for **Binary Search Trees (BST)** because in-order traversal of a BST yields the values in sorted (ascending) order.

#### Iterative Style

We traverse to the leftmost node, push nodes onto the stack along the way, process the node, and then switch to the right child.

```cpp
void in_order(TreeNode* root) {
    stack<TreeNode*> st;
    TreeNode* cur = root;
    
    while (!st.empty() || cur != nullptr) {
        if (cur != nullptr) {
            // Keep going left
            st.push(cur);
            cur = cur->left;
        } else {
            // Backtrack
            cur = st.top();
            st.pop();
            
            // Process Node
            cout << cur->val << endl;
            
            // Go right
            cur = cur->right;
        }
    }
}
```

#### Recursive Style

```cpp
void in_order(TreeNode* cur) {
    if (cur == nullptr) {
        return;
    }
    in_order(cur->left);
    // Process Node
    cout << cur->val << endl;
    in_order(cur->right);
}
```

### Post-Order Traversal

**Order:** Left $\rightarrow$ Right $\rightarrow$ Root.

This is useful when you need to process children before the parent (e.g., deleting a tree, or calculating height).

#### Iterative Style

A common trick to implement this iteratively is to use **two stacks**.

1. Perform a modified pre-order traversal (Root $\rightarrow$ Right $\rightarrow$ Left).
2. Store the result in a second stack (or reverse the result vector).

```cpp
void post_order(TreeNode* root) {
    if (!root) return;

    stack<TreeNode*> s1, s2;
    s1.push(root);

    while (!s1.empty()) {
        TreeNode* cur = s1.top();
        s1.pop();
        s2.push(cur); // Store for reverse printing

        // Push Left then Right (so Right is processed first in s1)
        if (cur->left) s1.push(cur->left);
        if (cur->right) s1.push(cur->right);
    }

    // Print s2 to get Left -> Right -> Root
    while (!s2.empty()) {
        cout << s2.top()->val << endl;
        s2.pop();
    }
}
```

#### Recursive Style

```cpp
void post_order(TreeNode* cur) {
    if (cur == nullptr) {
        return;
    }
    post_order(cur->left);
    post_order(cur->right);
    // Process Node
    cout << cur->val << endl;
}
```

## Binary Tree BFS (Level Order)

Breadth-First Search (BFS) processes the tree level by level.

### Iterative Style

This is the standard approach using a `queue`.

```cpp
void bfs(TreeNode* root) {
    if (!root) return;
    queue<TreeNode*> q;
    q.push(root);
    
    while (!q.empty()) {
        int levelSize = q.size(); // Snapshot size for current level
        
        for (int i = 0; i < levelSize; i++) {
            TreeNode* cur = q.front();
            q.pop();
            
            // Process Node
            cout << cur->val << " ";
            
            if (cur->left) q.push(cur->left);
            if (cur->right) q.push(cur->right);
        }
        cout << endl; // End of level
    }
}
```

### Recursive Style

Recursive BFS is less common but can be achieved by passing the queue to the recursive function.

```cpp
void bfs_recursive(queue<TreeNode*>& q) {
    if (q.empty()) return;
    
    int levelSize = q.size();
    for (int i = 0; i < levelSize; i++) {
        TreeNode* cur = q.front();
        q.pop();
        
        // Process Node
        cout << cur->val << " ";
        
        if (cur->left) q.push(cur->left);
        if (cur->right) q.push(cur->right);
    }
    cout << endl;
    
    bfs_recursive(q);
}
```

## Examples

### Pre-order Traversal: Path Sum

**Problem:** [LeetCode 112: Path Sum](https://leetcode.com/problems/path-sum/description/)

To check the path sum, we must traverse from parent to child, subtracting the current node's value from the target sum. If we reach a leaf and the remaining sum is 0, we found a path.

#### Iterative Solution

```cpp
bool hasPathSum(TreeNode* root, int targetSum) {
    if (!root) return false;
    
    // Stack stores pairs of {Node, Remaining_Sum}
    stack<pair<TreeNode*, int>> st;
    st.push({root, targetSum});
    
    while (!st.empty()) {
        auto [node, currentSum] = st.top();
        st.pop();
        
        currentSum -= node->val;
        
        // Check if leaf
        if (!node->left && !node->right && currentSum == 0) {
            return true;
        }
        
        if (node->right) st.push({node->right, currentSum});
        if (node->left) st.push({node->left, currentSum});
    }
    return false;
}
```

#### Recursive Solution

```cpp
bool hasPathSum(TreeNode* root, int targetSum) {
    if (root == nullptr) {
        return false;
    }
    
    targetSum -= root->val;
    
    // Check if leaf
    if (root->left == nullptr && root->right == nullptr) {
        return targetSum == 0;
    }
    
    return hasPathSum(root->left, targetSum) || hasPathSum(root->right, targetSum);
}
```

### In-order Traversal: Min Difference in BST

**Problem:** [LeetCode 530. Minimum Absolute Difference in BST](https://leetcode.com/problems/minimum-absolute-difference-in-bst/description/)

**Key Insight:** Since an in-order traversal of a BST visits nodes in sorted order ($v_1 < v_2 < v_3 ...$), the minimum difference must occur between two adjacent nodes in this traversal.

#### Iterative Solution

```cpp
int getMinimumDifference(TreeNode* root) {
    stack<TreeNode*> st;
    TreeNode* cur = root;
    int result = INT_MAX;
    int prev_val = -1; // Use a flag or -1 if values are non-negative
    bool first_node = true;

    while (!st.empty() || cur != nullptr) {
        if (cur != nullptr) {
            st.push(cur);
            cur = cur->left;
        } else {
            cur = st.top();
            st.pop();
            
            if (!first_node) {
                result = min(result, cur->val - prev_val);
            }
            
            prev_val = cur->val;
            first_node = false;
            cur = cur->right;
        }
    }
    return result;
}
```

#### Recursive Solution

```cpp
class Solution {
    int min_diff = INT_MAX;
    TreeNode* prev = nullptr; // Track the previous node pointer
    
public:
    int getMinimumDifference(TreeNode* root) {
        inorder(root);
        return min_diff;
    }

    void inorder(TreeNode* node) {
        if (!node) return;
        
        inorder(node->left);
        
        if (prev != nullptr) {
            min_diff = min(min_diff, node->val - prev->val);
        }
        prev = node;
        
        inorder(node->right);
    }
};
```

### Post-order Traversal: Lowest Common Ancestor

**Problem:** [LeetCode 236: Lowest Common Ancestor](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/description/)

We work from the bottom up. If a node receives `p` from one side and `q` from the other side, that node is the Lowest Common Ancestor (LCA).

```cpp
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    // Base case: root is null, or root is one of the targets
    if (root == nullptr || root == p || root == q) {
        return root;
    }

    // Post-order: visit children first
    TreeNode* left = lowestCommonAncestor(root->left, p, q);
    TreeNode* right = lowestCommonAncestor(root->right, p, q);

    // If both return non-null, root is the LCA
    if (left != nullptr && right != nullptr) {
        return root;
    }
    
    // Otherwise, return the non-null child (propagate up)
    return (left != nullptr) ? left : right;
}
```

### BFS: Zigzag Level Order Traversal

**Problem:** [LeetCode 103: Binary Tree Zigzag Level Order Traversal](https://leetcode.com/problems/binary-tree-zigzag-level-order-traversal/)

We can use two stacks to simulate the zigzag motion.

* **Level Even:** Pop from S1, Push children (Left then Right) to S2.
* **Level Odd:** Pop from S2, Push children (Right then Left) to S1.

```cpp
vector<vector<int>> zigzagLevelOrder(TreeNode* root) {
    if (!root) return {};
    
    vector<vector<int>> result;
    stack<TreeNode*> s1;
    stack<TreeNode*> s2;
    
    s1.push(root);
    
    while (!s1.empty() || !s2.empty()) {
        vector<int> level;
        
        // Process S1 (Left to Right)
        if (!s1.empty()) {
            while (!s1.empty()) {
                TreeNode* cur = s1.top(); s1.pop();
                level.push_back(cur->val);
                
                if (cur->left) s2.push(cur->left);
                if (cur->right) s2.push(cur->right);
            }
        } 
        // Process S2 (Right to Left)
        else {
            while (!s2.empty()) {
                TreeNode* cur = s2.top(); s2.pop();
                level.push_back(cur->val);
                
                // Note order: Right then Left
                if (cur->right) s1.push(cur->right);
                if (cur->left) s1.push(cur->left);
            }
        }
        result.push_back(level);
    }
    return result;
}
```
