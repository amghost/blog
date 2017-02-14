# 思路
构建一棵树不是理想的方案，连题目也说了：“Find an algorithm without reconstructing the tree.”。
本题有点像校验一条数学式是否合法，用栈来解决。主要利用到了叶子节点`#`，这里有两个处理要点：
以`"9,3,4,#,#,1,#,#,2,#,6,#,#"`为例

1. `#`本身就是一个叶子节点，如4后面的两个叶子节点#，以及2后的左叶子节点#
2. 左右节点完整的中间节点可以塌缩成叶子节点，这样在栈中可以用`#`表示，比如以6为根的子树收缩为叶子节点

所以整个算法是围绕`#`来进行

# 压栈和出栈
只有一种情况不需要将读到的元素压栈，那就是读到了`#`且栈顶元素为叶子节点，因为这个时候可以弹出栈了。
弹出第一个`#`，此时两个叶子节点`#`的父节点作为一颗“完整”的子树，可以塌缩为一个叶子节点`#`

注意这里的栈顶叶子节点可能“前身”是一个中间节点，因为左右节点完整所以塌缩为一个叶子节点

由于根节点没有兄弟节点和父节点，所以它不会出栈，但是最后会塌缩成一个`#`（因为左右子树完整），反之如果根节点被弹出栈了，那就是输入有问题，不是合法的树的preorder。

# 图解说明
以`"9,3,4,#,#,1,#,#,2,#,6,#,#"`为例：
原始树结构为：
```
     _9_
    /   \
   3     2
  / \   / \
 4   1  #  6
/ \ / \   / \
# # # #   # #
```


读到第一个`#`时，树是长这个样子的：
```
         _9_
        / 
      3 
     /  
    4 
   / \
  #
```
栈的内容是：`9, 3, 4, #`

读到第二个`#`时，树长这个样子：
```
         _9_
        / 
      3 
     /  
    4 
   / \
  #   #
```
此时检测到栈顶是一个叶子节点`#`，于是将其弹出，同时将父节点`4`塌缩为叶子节点`#`，于是树变这样：
```
         _9_
        / 
      3 
     /  
    #
```
栈的内容变成这样：`9, 3, #`

同理，当`1`塌缩为叶子节点时，树长这样：
```
         _9_
        / 
      3 
     / \ 
    #   #
```
此时我们要加一个循环，继续塌缩父节点`3`（栈顶连续两个元素都是叶子节点`#`），直到塌缩条件不符合为止
所以树变成：
```
     _9_
    / 
   #
```

**中间节点塌缩为叶子节点的好处是**简化了算法中的操作对象，只有叶子节点，也方便我们出栈

# 循环条件和判定条件
使用栈就是一个不断压栈、出栈的过程，要明确**循环继续的条件**以及返回结果的**判定条件**

循环中我们不断向前读取元素，循环条件为读到了最后一个元素；或许你可能会觉得加上栈非空作为循环条件，有道理，可以尝试，不过我的解法里面不需要。
判定结果首先必须是所有元素已经读完，同时栈中仅剩一个叶子节点，该节点是根节点塌缩的。

# 代码

```cpp
class Solution {
    public:
        bool isValidSerialization(string preorder) {
            stack<string> node_stack;
            string::size_type prev_pos = 0, next_pos = 0;

            while (next_pos != string::npos) {
                next_pos = preorder.find_first_of(',', prev_pos);
                string t = preorder.substr(prev_pos, next_pos - prev_pos);

                if (t == "#" && !node_stack.empty() && node_stack.top() == "#") {
                    node_stack.pop();
                    if (node_stack.empty())
                        return false;
                    node_stack.top() = "#";

                    while (node_stack.size() > 2 && node_stack.top() == "#") {
                        node_stack.pop();       // You need to pop one to get another one
                        if (node_stack.top() == "#") {
                            node_stack.pop();
                            node_stack.top() = "#";
                        }
                        else {
                            node_stack.push("#");
                            break;
                        }
                    }
                }
                else {
                    node_stack.push(t);
                }

                prev_pos = next_pos + 1;
            }

            return ( next_pos == string::npos && node_stack.size() == 1 && node_stack.top() == "#" );
        }
};
```

