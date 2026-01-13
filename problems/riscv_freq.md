| Cause 值 | 异常类型                           | 含义           |
| ------- | ------------------------------ | ------------ |
| 0       | Instruction address misaligned | 指令地址未对齐      |
| 1       | Instruction access fault       | 指令访问异常       |
| 2       | Illegal instruction            | 非法指令         |
| 3       | Breakpoint                     | 断点           |
| 4       | Load address misaligned        | 加载地址未对齐      |
| 5       | Load access fault              | 加载访问异常       |
| 6       | Store/AMO address misaligned   | 存储地址未对齐      |
| 7       | Store/AMO access fault         | 存储访问异常       |
| 8       | Environment call from U-mode   | 来自用户态的 ecall |
| 9       | Environment call from S-mode   | 来自内核态的 ecall |
| 11      | Environment call from M-mode   | 来自机器态的 ecall |
| 12      | Instruction page fault         | 指令页错误        |
| 13      | Load page fault                | 加载页错误        |
| 15      | **Store/AMO page fault**       | 🚨 存储（写入）页错误 |
