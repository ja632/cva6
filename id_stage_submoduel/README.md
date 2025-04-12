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

### 🔄 Rename Commit 控制與指標邏輯
```systemverilog
assign issue_enable[0] = issue_instr_valid_i[0] & issue_ack_o[0] & !no_rename_i[0];
assign issue_enable[1] = issue_instr_valid_i[1] & issue_ack_o[1] & !no_rename_i[1];

assign commit_enable[0] = commit_instr_o[0].valid & commit_ack_i[0] & (commit_instr_o[0].rd != 6'd0);
assign commit_enable[1] = commit_instr_o[1].valid & commit_ack_i[1] & (commit_instr_o[1].rd != 6'd0);

assign Pr_rd_o_rob[0] = issue_ptr[0]; // 實體 register 分配結果
assign Pr_rd_o_rob[1] = issue_ptr[1];

assign commit_ptr[0] = commit_instr_o[0].rd[4:0]; // 被回收的實體暫存器編號
assign commit_ptr[1] = commit_instr_o[1].rd[4:0];
```

### 🔢 分配指標管理邏輯
```systemverilog
// 使用遞增方式取用 pointer_freelist 中的 register
always_ff @(posedge clk_i or negedge rst_ni) begin
    if (~rst_ni) begin
        pointer_freelist <= 'd2; // x0 和 x1 不可使用
    end else if(issue_enable[0] & issue_enable[1]) begin
        pointer_freelist <= (pointer_freelist > 28) ? 2 : pointer_freelist + 2;
    end else if(issue_enable[0] | issue_enable[1]) begin
        pointer_freelist <= (pointer_freelist > 28) ? 2 : pointer_freelist + 1;
    end
end

// 決定 issue_ptr：即將分配出去的實體暫存器編號
always_comb begin
    if(issue_enable[0] & issue_enable[1]) begin
        issue_ptr[0] = pointer_freelist;
        issue_ptr[1] = pointer_freelist + 1;
    end else begin
        issue_ptr[0] = pointer_freelist;
        issue_ptr[1] = pointer_freelist;
    end
end
```
### ♻️ Free List Update 機制

```systemverilog
// ------------------------------------------------------------------------------------------------
//  update free list
// ------------------------------------------------------------------------------------------------
always_comb begin
    // 預設將下一狀態設為目前狀態
    physical_register_freelist_n = physical_register_freelist_q;

    // 若遇到 flush 未發出指令，則從分支快照中還原 freelist 狀態
    if(flush_unissied_instr_i) begin
        for (int unsigned j = 0; j < CVA6Cfg.Nrmaptable; j++) begin
            if(physical_register_freelist_n[j]!=1'd0) begin 
                physical_register_freelist_n[j] = br_snopshot_freelist[br_pop_ptr][j];
            end
        end
    end

    // 新發出的指令要分配 register，將對應 index 設為 1（代表已分配）
    if(issue_enable[0]) begin
        physical_register_freelist_n[issue_ptr[0]] = 1'd1;
    end 
    if(issue_enable[1]) begin
        physical_register_freelist_n[issue_ptr[1]] = 1'd1;
    end 

    // commit 階段的指令釋放 register，設為 0（代表可用）
    if(commit_enable[0]) begin 
        physical_register_freelist_n[commit_ptr[0]] = 1'd0;
    end
    if(commit_enable[1]) begin 
        physical_register_freelist_n[commit_ptr[1]] = 1'd0;
    end
end

// ------------------------------------------------------------------------------------------------
// 以暫存器的方式記住 freelist 狀態（同步更新）
// ------------------------------------------------------------------------------------------------
always_ff @(posedge clk_i or negedge rst_ni) begin
    if (~rst_ni) 
    begin
        // Reset 時，freelist 初始化（只有 x0 無效）
        physical_register_freelist_q <= 'd1;
    end 
    else if(flush_i) begin
        // flush 時也將 freelist 重設（例：發生 exception）
        physical_register_freelist_q <= 'd1;
    end
    else begin
        // 正常情況下將下一狀態覆蓋
        physical_register_freelist_q <= physical_register_freelist_n;
    end
end
```

🔍 **說明摘要：**
- `physical_register_freelist_q` 是一個 bitmap，每個 bit 代表對應實體暫存器是否為空（0=可用，1=已分配）。
- `flush_unissied_instr_i` 會觸發從分支快照記憶體中還原 freelist 狀態。
- 新指令 rename 時會登記為 "已分配"，commit 時會釋放為 "可用"。
- 結合 `pointer_freelist` 指標控制分配順序，在雙發射架構下支援兩筆同時更新。

---
### 🧠 Freelist - 分支快照邏輯（Branch Snapshot in Freelist）

```systemverilog
// ------------------------------------------------------------------------------------------------
// 判斷目前是否發派的是 branch/jalr 指令（需要記錄快照）
// ------------------------------------------------------------------------------------------------
assign issue_is_branch[0] = (br_instr_i[0] & issue_instr_valid_i[0] & issue_ack_o[0]);
assign issue_is_branch[1] = (br_instr_i[1] & issue_instr_valid_i[1] & issue_ack_o[1]);

always_ff @(posedge clk_i or negedge rst_ni) begin
    if (~rst_ni) begin
        // Reset 時將所有 branch 快照資料清空
        for (int unsigned j = 0; j < 16; j++) begin
            br_snopshot_freelist[j] <= 'd0;
        end
    end else begin 
        // 同時發派兩條 branch 指令（需要記錄兩個快照）
        if(issue_is_branch[0] & issue_is_branch[1]) begin
            // 快照 1：紀錄當下的 physical_register_freelist 狀態為 br_push_ptr
            for (int unsigned j = 0; j < CVA6Cfg.Nrmaptable; j++) begin
                if(issue_enable[0] & (j==issue_ptr[0])) // 發派新的實體暫存器
                    br_snopshot_freelist[br_push_ptr][j] <= 1'd1;
                else if(commit_enable[0] & (j==commit_ptr[0]) || commit_enable[1] & (j==commit_ptr[1])) // 寄存器回收
                    br_snopshot_freelist[br_push_ptr][j] <= 1'd0;
                else // 其餘保留原 freelist 狀態
                    br_snopshot_freelist[br_push_ptr][j] <= physical_register_freelist_q[j];
            end

            // 快照 2：下一個 br_push_ptr+1 也同樣更新（for 第二條 branch）
            for (int unsigned j = 0; j < CVA6Cfg.Nrmaptable; j++) begin
                if(issue_enable[0] & (j==issue_ptr[0]) || issue_enable[1] & (j==issue_ptr[1]))
                    br_snopshot_freelist[br_push_ptr+4'd1][j] <= 1'd1;
                else if(commit_enable[0] & (j==commit_ptr[0]) || commit_enable[1] & (j==commit_ptr[1]))
                    br_snopshot_freelist[br_push_ptr+4'd1][j] <= 1'd0;
                else
                    br_snopshot_freelist[br_push_ptr+4'd1][j] <= physical_register_freelist_q[j];
            end
        end 
        // 僅有 issue[0] 是 branch 指令
        else if(issue_is_branch[0]) begin 
            for (int unsigned j = 0; j < CVA6Cfg.Nrmaptable; j++) begin
                if(issue_enable[0] & (j==issue_ptr[0]))
                    br_snopshot_freelist[br_push_ptr][j] <= 1'd1;
                else if(commit_enable[0] & (j==commit_ptr[0]) || commit_enable[1] & (j==commit_ptr[1]))
                    br_snopshot_freelist[br_push_ptr][j] <= 1'd0;
                else
                    br_snopshot_freelist[br_push_ptr][j] <= physical_register_freelist_q[j];
            end
        end 
        // 非 branch 發派，仍需寫入最新狀態避免錯過更新（處理 flush/mispredict 時用）
        else begin 
            for (int unsigned j = 0; j < CVA6Cfg.Nrmaptable; j++) begin
                if(issue_enable[0] & (j==issue_ptr[0]) || issue_enable[1] & (j==issue_ptr[1]))
                    br_snopshot_freelist[br_push_ptr][j] <= 1'd1;
                else if(commit_enable[0] & (j==commit_ptr[0]) || commit_enable[1] & (j==commit_ptr[1]))
                    br_snopshot_freelist[br_push_ptr][j] <= 1'd0;
                else
                    br_snopshot_freelist[br_push_ptr][j] <= physical_register_freelist_q[j];
            end
        end
    end
end
```
### 🔄 Busytable 子模組初始化與定義區塊

```systemverilog
module busytable import ariane_pkg::*;  #(
    parameter config_pkg::cva6_cfg_t CVA6Cfg = config_pkg::cva6_cfg_empty
) (
    input  logic                                                                   clk_i,    // 時鐘訊號
    input  logic                                                                   rst_ni,   // 非同步 reset（低電位有效）
    input  logic                                                                   flush_i,  // 清空全部 busytable 狀態
    input  logic                                                                   flush_unissied_instr_i, // 清空未發派指令的狀態

    // 從 scoreboard 接收 issue 訊號（發派指令）
    input  scoreboard_entry_t    [CVA6Cfg.NrissuePorts-1:0]                        issue_instr_i,         // 發派的 scoreboard 指令內容
    input  logic                 [CVA6Cfg.NrissuePorts-1:0]                        issue_instr_valid_i,   // 發派指令是否有效
    input  logic                 [CVA6Cfg.NrissuePorts-1:0]                        issue_ack_o,           // 發派是否成功（handshake）
    input  logic                 [CVA6Cfg.NrissuePorts-1:0]                        no_rename_i,           // 指令是否不需要 rename

    // 從 scoreboard 接收 commit 訊號（指令寫回）
    input  scoreboard_entry_t    [CVA6Cfg.NrissuePorts-1:0]                        commit_instr_o,        // 寫回的 scoreboard 指令內容
    input  logic                 [CVA6Cfg.NrissuePorts-1:0]                        commit_ack_i,          // 寫回是否成功
    input  logic                 [CVA6Cfg.NrissuePorts-1:0]                        commit_no_rename,      // commit 的指令是否沒被 rename

    // 實體暫存器對應的目的暫存器與來源暫存器編號（不帶 valid bit）
    input                        [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-2:0]     Pr_rd_o_rob,           // rename 指定的 destination register（實體）
    input                        [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-2:0]     Pr_rs1_o,              // rename 對應的 source register 1（實體）
    input                        [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-2:0]     Pr_rs2_o,              // rename 對應的 source register 2（實體）
    input                        [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-2:0]     Pr_rs3_o,              // rename 對應的 source register 3（實體）

    // 加上 valid bit 的輸出來源暫存器（提供給後續使用單元）
    output logic                 [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-1:0]     Pr_rs1_o_rob,          // rs1 + valid bit
    output logic                 [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-1:0]     Pr_rs2_o_rob,          // rs2 + valid bit
    output logic                 [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-1:0]     Pr_rs3_o_rob,          // rs3 + valid bit

    // 分支處理相關資訊（br_tag 快照機制）
    input logic                  [CVA6Cfg.NrissuePorts-1:0]                        br_instr_i,            // 是否為分支指令
    input                        [3:0]                                             br_push_ptr,           // push snapshot 編號
    input                        [3:0]                                             br_pop_ptr             // pop snapshot 編號
);
```

---

### 🧩 Busytable - 註冊區塊與中間變數說明

```systemverilog
localparam int unsigned BITS_MAPTABLE = $clog2(CVA6Cfg.Nrmaptable); // 實體暫存器 bit 數

// busytable 狀態記錄：是否某個實體暫存器正在使用中
logic  physical_register_busytable_n [CVA6Cfg.Nrmaptable-1:0]; // 下一個狀態
logic  physical_register_busytable_q [CVA6Cfg.Nrmaptable-1:0]; // 當前狀態

// 分支 mispredict 時的快照
logic  br_snopshot_busytable [15:0][CVA6Cfg.Nrmaptable-1:0]; // 最多記錄 16 組快照

// 實體暫存器編號
logic [CVA6Cfg.NrissuePorts-1:0][BITS_MAPTABLE-1:0] issue_ptr;   // rename 階段分配的實體暫存器
logic [CVA6Cfg.NrissuePorts-1:0][BITS_MAPTABLE-1:0] commit_ptr;  // commit 階段釋放的實體暫存器

// 啟用訊號
logic [CVA6Cfg.NrissuePorts-1:0] commit_enable; // 是否可釋放實體暫存器
logic [CVA6Cfg.NrissuePorts-1:0] issue_enable;  // 是否可發派實體暫存器

// 是否是分支指令
logic [CVA6Cfg.NrissuePorts-1:0] issue_is_branch;

// 標記是否來源暫存器尚未 ready
logic busy_rs1; 
logic busy_rs2; 

// Busytable 是否已滿（判斷是否有可用暫存器）
logic full;    
```

---

### 📘 功能摘要

| 區塊名稱         | 說明                                                                 |
|------------------|----------------------------------------------------------------------|
| `physical_register_busytable_q` | 記錄所有實體暫存器目前是否忙碌（已分配但尚未 commit）                     |
| `br_snopshot_busytable`         | 在分支發派時，記錄 busytable 的狀態以支援 mispredict 還原                   |
| `issue_ptr` / `commit_ptr`      | 分別代表此 cycle 發派與寫回的目的暫存器編號                                 |
| `issue_enable` / `commit_enable`| 控制訊號判斷當前是否進行發派或釋放動作                                      |
| `busy_rs1/rs2`                  | 用來標示來源暫存器是否尚未就緒（尚在計算中）                                 |
| `full`                          | 當所有暫存器皆為 busy 時，表示 busytable 滿載，可能需 stall pipeline           |

### 🔁 Busytable - Push/Pop 與狀態更新邏輯

```systemverilog
// ------------------------------------------------------------------------------------------------
// push / pop signal
// ------------------------------------------------------------------------------------------------

// 判斷是否發派成功且需要 rename
assign issue_enable[0] = issue_instr_valid_i[0] & issue_ack_o[0] & !no_rename_i[0];
assign issue_enable[1] = issue_instr_valid_i[1] & issue_ack_o[1] & !no_rename_i[1];

// 當前要 mark 成 busy 的暫存器 index
assign issue_ptr[0] = Pr_rd_o_rob[0];
assign issue_ptr[1] = Pr_rd_o_rob[1];

// 判斷是否為有效的寫回（commit）指令
assign commit_enable[0] = commit_instr_o[0].valid & commit_ack_i[0] & (commit_instr_o[0].rd != 6'd0);
assign commit_enable[1] = commit_instr_o[1].valid & commit_ack_i[1] & (commit_instr_o[1].rd != 6'd0);

// 要從 busytable 中釋放的實體暫存器 index
assign commit_ptr[0] = commit_instr_o[0].rd[4:0];
assign commit_ptr[1] = commit_instr_o[1].rd[4:0];

// ------------------------------------------------------------------------------------------------
// 更新 busytable 狀態
// ------------------------------------------------------------------------------------------------
always_comb begin
    physical_register_busytable_n = physical_register_busytable_q;

    // 若發生 flush_unissied_instr_i（如 branch mispredict），從快照還原 busytable 狀態
    if(flush_unissied_instr_i) begin
        for (int unsigned j = 0; j < CVA6Cfg.Nrmaptable; j++) begin
            if(physical_register_busytable_q[j] != 1'd0) begin 
                physical_register_busytable_n[j] = br_snopshot_busytable[br_pop_ptr][j];
            end
        end
    end 

    // 發派時將對應實體暫存器設為 busy
    if(issue_enable[0]) begin
        physical_register_busytable_n[issue_ptr[0]] = 1'd1;
    end
    if(issue_enable[1]) begin
        physical_register_busytable_n[issue_ptr[1]] = 1'd1;
    end 

    // 寫回時將對應暫存器從 busytable 移除（標記為 idle）
    if(commit_enable[0]) begin 
        physical_register_busytable_n[commit_ptr[0]] = 1'd0;
    end
    if(commit_enable[1]) begin 
        physical_register_busytable_n[commit_ptr[1]] = 1'd0;
    end

    // x0 寄存器永遠不可被標記為 busy
    physical_register_busytable_n[0] = 1'd0;
end
```

---

### 📘 小結：Busytable 更新邏輯

| 操作階段 | 動作內容                                       |
|----------|------------------------------------------------|
| 發派 (issue) | 將 `rd` 對應的實體暫存器標記為 busy (1)             |
| 寫回 (commit) | 將 `rd` 回收時從 busytable 中釋放 (設為 0)         |
| flush_unissued | 發生 branch mispredict 時，透過快照還原 busytable |
| 強制清除 | x0 寄存器永遠設為 0，避免誤標 busy                    |

