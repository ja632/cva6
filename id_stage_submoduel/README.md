
## 📦 freelist 子模組功能說明

`freelist` 是 `re_name` 模組中用來管理**實體暫存器分配與回收**的子模組。

### 功能重點

- 維護一份 32-entry 的實體暫存器狀態表 (`physical_register_freelist_q`)，0 表示**使用中**，1 表示**可用**。
- 提供 **rename 階段分配 register**（`Pr_rd_o_rob`）與 **commit 階段回收 register** 的邏輯。
- 支援 **分支指令快照備份與還原**（snapshot/restore）。

---

### 📥 模組介面（I/O）
```systemverilog
input logic clk_i,         // 時鐘輸入
input logic rst_ni,        // 非同步 Reset (active low)
input logic flush_i,       // flush 整體 rename 狀態
input logic flush_unissied_instr_i, // flush 未發出的指令

input logic [NrIssuePorts-1:0] issue_instr_valid_i, // issue port 指令有效
input logic [NrIssuePorts-1:0] issue_ack_o,         // issue port 被接受
input logic [NrIssuePorts-1:0] no_rename_i,         // 是否不需要 rename（例如 store）

input scoreboard_entry_t [NrIssuePorts-1:0] commit_instr_o, // commit 階段的指令
input logic [NrIssuePorts-1:0] commit_ack_i,                // commit 被接受
input logic [NrIssuePorts-1:0] commit_no_rename,            // commit 階段不需 rename

output logic [NrIssuePorts-1:0][REG_ADDR_SIZE-2:0] Pr_rd_o_rob, // 分配到的實體暫存器

input logic [NrIssuePorts-1:0] br_instr_i,  // 是否為分支指令（snapshot 用）
input [3:0] br_push_ptr, br_pop_ptr         // 分支 snapshot 的指標
```

---

### 🔐 內部狀態變數
```systemverilog
localparam int unsigned BITS_MAPTABLE = $clog2(CVA6Cfg.Nrmaptable);
logic [CVA6Cfg.Nrmaptable-1:0] physical_register_freelist_q; // 現在的 freelist 狀態
logic [CVA6Cfg.Nrmaptable-1:0] physical_register_freelist_n; // 下一狀態
logic [CVA6Cfg.Nrmaptable-1:0] br_snopshot_freelist [15:0];  // 最多 16 個分支 snapshot

logic [NrIssuePorts-1:0][BITS_MAPTABLE-1:0] issue_ptr;  // rename 階段的實體 reg 分配
logic [NrIssuePorts-1:0][BITS_MAPTABLE-1:0] commit_ptr; // commit 階段要釋放的實體 reg
logic [NrIssuePorts-1:0] commit_enable; // 是否允許 commit
logic [NrIssuePorts-1:0] issue_enable;  // 是否允許 rename 發生
logic [NrIssuePorts-1:0] issue_is_branch; // 是否是分支指令

logic [BITS_MAPTABLE-1:0] pointer_freelist; // 下一個要分配的 register 編號
```

---

✅ 此模組與 `maptable` / `busytable` 共同構成 rename 階段中 register 分配與追蹤系統的一環。
