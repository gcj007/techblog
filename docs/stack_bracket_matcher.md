# Bracket matching with stack

**更好的写法是：采用“单一出口（Single Exit）”模式。** 用一个标志变量 `valid` 记录状态，一旦发现失效，就 `break` 跳出循环，最后**统一**在函数末尾释放内存并返回结果。

这样做的好处是：**成功和失败路径泾渭分明**，内存管理只写一次，绝不出错。

请看下面重构后的代码，注释里明确标出了 **“失效路径”** 和 **“成功路径””**：

```c
#include <stdbool.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

bool isValid(char *s) {
    int len = strlen(s);
    
    // 【边界成功】空字符串直接视为有效
    if (len == 0) {
        return true;
    }

    // 1. 分配栈空间
    char *stack = (char*)malloc(len * sizeof(char));
    if (stack == NULL) {
        return false; // 内存分配失败（系统级错误）
    }

    int top = -1;           // 栈顶指针
    bool valid = true;      // 【核心标志】默认为有效，只有遇到明确错误才改为 false

    // 2. 遍历字符串
    for (int i = 0; i < len; i++) {
        char c = s[i];

        // --- 分支一：左括号，入栈 ---
        if (c == '(' || c == '[' || c == '{') {
            stack[++top] = c;
            continue; // 处理下一个字符
        }

        // --- 分支二：右括号，开始匹配检查 ---
        // 【失效条件 1】栈为空 => 没有左括号可以匹配
        if (top == -1) {
            valid = false;
            break; // 立即终止遍历
        }

        // 取出栈顶左括号
        char topChar = stack[top];

        // 【失效条件 2】栈顶括号与当前右括号不成对
        if ((c == ')' && topChar != '(') ||
            (c == ']' && topChar != '[') ||
            (c == '}' && topChar != '{')) {
            valid = false;
            break; // 立即终止遍历
        }

        // --- 匹配成功：弹出栈顶 ---
        top--;
    }

    // 3. 【最终判定】遍历结束后，如果 valid 为 true 但栈不为空，说明有多余左括号，依然失效
    if (valid && top != -1) {
        valid = false;
    }

    // 4. 【单一出口】统一释放内存，统一返回结果
    free(stack);
    return valid;
}

// 主函数（用于测试）
int main(int argc, char *argv[]) {
    if (argc != 2) {
        printf("用法: %s \"括号字符串\"\n", argv[0]);
        return 1;
    }

    printf("%s\n", isValid(argv[1]) ? "有效" : "无效");
    return 0;
}
```

---

### 为什么这样写“分得明确”？

1. **失效场景高度集中**  
   代码中明确列出了 **仅有的 3 种失效情况**：
   - 栈空了，却来了右括号（`")"`）
   - 栈顶不匹配（`"(]"`）
   - 遍历完了，栈里还剩东西（`"((()"`）  
   这三种情况一旦触发，`valid` 立即变 `false`，然后 `break` 跳出。

2. **成功场景只有一条路**  
   只有**全程没触发 break**，且遍历结束后 `top == -1`，`valid` 才保持 `true`。

3. **内存释放只有一处**  
   无论成功还是失败，`free(stack)` 都只写在最后，绝无遗漏风险，代码极度整洁。

这种写法在大型工程项目中非常推荐，因为**修改和维护时，你不需要在十几个 `return` 里找遗漏的 `free`**，逻辑一目了然。