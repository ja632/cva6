# cva6

##ariane_pkg
## 🧾 scoreboard_entry_t 結構說明
```systemverilog
typedef struct packed {
  logic [riscv::VLEN-1:0] pc;
  logic [TRANS_ID_BITS-1:0] trans_id;
  fu_t fu;
  fu_op op;
  logic [REG_ADDR_SIZE-1:0] rs1;
  logic [REG_ADDR_SIZE-1:0] rs2;
  logic [REG_ADDR_SIZE-1:0] rd;
  riscv::xlen_t result;
  logic valid;
  logic use_imm;
  logic use_zimm;
  logic use_pc;
  exception_t ex;
  branchpredict_sbe_t bp;
  logic is_compressed;
  riscv::xlen_t rs1_rdata;
  riscv::xlen_t rs2_rdata;
  logic [riscv::VLEN-1:0] lsu_addr;
  logic [(riscv::XLEN/8)-1:0] lsu_rmask;
  logic [(riscv::XLEN/8)-1:0] lsu_wmask;
  riscv::xlen_t lsu_wdata;
  logic vfp;
} scoreboard_entry_t;
```
### 📝 scoreboard_entry_t 欄位對應說明

| 欄位名稱         | 資料型態                                | 說明 |
|------------------|------------------------------------------|------|
| `pc`             | `logic [riscv::VLEN-1:0]`                | 指令的程式計數器（Program Counter） |
| `trans_id`       | `logic [TRANS_ID_BITS-1:0]`             | transaction ID，用於 scoreboard 對應 |
| `fu`             | `fu_t`                                   | 功能單元代碼，例如 ALU、LSU、BRU 等 |
| `op`             | `fu_op`                                  | 對應功能單元的 operation code |
| `rs1`            | `logic [REG_ADDR_SIZE-1:0]`             | 第一個來源暫存器索引 |
| `rs2`            | `logic [REG_ADDR_SIZE-1:0]`             | 第二個來源暫存器索引 |
| `rd`             | `logic [REG_ADDR_SIZE-1:0]`             | 寫入目的暫存器索引 |
| `result`         | `riscv::xlen_t`                          | 計算結果或 immediate、浮點元件中的 rs2 |
| `valid`          | `logic`                                  | 結果是否有效 |
| `use_imm`        | `logic`                                  | 是否使用立即數作為 Operand B |
| `use_zimm`       | `logic`                                  | 是否使用零擴展的立即數作為 Operand A |
| `use_pc`         | `logic`                                  | 是否以 PC 作為 Operand A（如 auipc） |
| `ex`             | `exception_t`                            | 指令是否發生例外 |
| `bp`             | `branchpredict_sbe_t`                    | 分支預測資訊結構 |
| `is_compressed`  | `logic`                                  | 是否為壓縮指令 |
| `rs1_rdata`      | `riscv::xlen_t`                          | rs1 的實際值，供 RVFI 使用 |
| `rs2_rdata`      | `riscv::xlen_t`                          | rs2 的實際值，供 RVFI 使用 |
| `lsu_addr`       | `logic [riscv::VLEN-1:0]`                | Load/Store 使用的目標地址 |
| `lsu_rmask`      | `logic [(riscv::XLEN/8)-1:0]`            | 記憶體讀遮罩 |
| `lsu_wmask`      | `logic [(riscv::XLEN/8)-1:0]`            | 記憶體寫遮罩 |
| `lsu_wdata`      | `riscv::xlen_t`                          | 要寫入記憶體的資料 |
| `vfp`            | `logic`                                  | 是否為向量浮點運算指令 |

RVFI 是 RISC-V Formal Interface 的縮寫

## 🧾 fetch_entry_t 結構說明
```
typedef struct packed {
  logic [riscv::VLEN-1:0] address;
  logic [31:0] instruction;
  branchpredict_sbe_t     branch_predict;
  exception_t             ex;
} fetch_entry_t;
```

### 欄位名稱	資料型態	說明
address	logic [riscv::VLEN-1:0]	該指令的虛擬地址，代表其在記憶體中的位置。通常來源為 I-Cache or Frontend PC。
instruction	logic [31:0]	指令內容本身，是已經過 re-align 的 32-bit 指令（即使是壓縮指令也會展開成 32-bit）。
branch_predict	branchpredict_sbe_t	此指令的分支預測資訊，包含是否預測為分支、預測目標地址等。是 branch prediction 單元（例如 BTB/BHT）產出的結果。
ex	exception_t	指令在取指階段（如 page fault、access fault）所發生的例外。用於確保指令執行時能正確處理 earlier exceptions。

## 🧠 為什麼 ID Stage 需要 16-entry Buffer？

在 CVA6 dual-issue 的架構中，`id_stage` 設計了一個長度為 16 的 buffer（環狀 FIFO），目的是為了提高 pipeline 吞吐率、容錯能力並支援雙發射（dual-issue）指令流程。

---

### 📌 主要功能與設計目的：

1. **支援 Dual-Issue 流程**
   - Frontend 每個 cycle 可能提供 2 條指令 (`fetch_entry_i[0]` 和 `fetch_entry_i[1]`)
   - Backend 若暫時只能 issue 1 條或 0 條指令，這個 buffer 就能暫存未送出的 instruction
   - ✅ 解決「進快出慢」的節奏不一致問題

2. **解耦解碼與發射邏輯**
   - Decode 可持續運作，不會被 Issue 階段的 stall 卡住
   - ✅ 提高整體 pipeline 運作效率

3. **支援分支錯誤處理與 flush**
   - 指令若錯誤（如 misprediction）需 flush，buffer 可提供 rollback 能力
   - ✅ 支援 speculative execution + recovery

4. **提升 pipeline 吞吐率與靈活性**
   - Decode 預先處理、先解碼再等待發射
   - ✅ 減少 frontend 空轉、增加有效指令密度

---

### ⚙️ 為什麼選 16-entry？

| 原因 | 說明 |
|------|------|
| Dual-Issue 對應 buffer 空間 | 16 entries ≈ 可暫存 8 個 cycle 的指令流（2 條/cycle） |
| Pipeline flush 邏輯 | Flush 時清空 buffer 效率高 |
| 資源成本平衡 | 若 buffer 太小會頻繁阻塞，太大又會增加面積與複雜度 |
| 可推動高 IPC | 維持 decode 一直 work，支援 backend 計算資源飽和運作 |

---

### 🔍 原始程式碼關鍵：

```systemverilog
// 決定是否允許 fetch_entry_i 同時進入兩條
assign dual_fetch = ((id_ins_num - issue_instr_ack_i[0] - issue_instr_ack_i[1]) < 5'd15);

// 決定是否已滿，不允許再放入
assign issue_full = (id_ins_num > 5'd16);
```
## 🧠 `re_name` 模組概觀（Register Renaming Stage）

`re_name` 是 CVA6 中負責將解碼後的虛擬暫存器映射至實體暫存器的模組，支援 dual-issue 和 Tomasulo 架構。

---

### 📥 輸入介面
| Port 名稱 | 資料型別 | 說明 |
|-----------|-----------|------|
| `clk_i` / `rst_ni` | `logic` | 時脈與非同步重置信號 |
| `flush_i` | `logic` | flush pipeline 中所有資料 |
| `flush_unissied_instr_i` | `logic` | 清除尚未送出至 issue 的指令（如分支錯誤） |

### 🔁 來自 ID 階段的資料
| Port 名稱 | 說明 |
|-----------|------|
| `rename_instr_i` | 解碼後的 scoreboard entry（指令資訊） |
| `rename_instr_valid_i` | 該筆指令是否有效 |
| `rename_ack_o` | 回應 ID 是否接收成功 |
| `is_ctrl_flow_i` | 是否為控制流程指令（例如 branch、jump） |

### 📤 輸出到 Issue 階段
| Port 名稱 | 說明 |
|-----------|------|
| `issue_instr_o` | 已完成 renaming 的指令資訊 |
| `issue_instr_valid_o` | 該筆輸出指令是否有效 |
| `issue_ack_i` | issue stage 是否成功接收 |
| `is_ctrl_flow_o` | 是否為控制流程指令 |

### ✅ Commit 資訊（釋放實體暫存器）
| Port 名稱 | 說明 |
|-----------|------|
| `commit_instr_o` | Commit 完成的指令（供 rename buffer 清除） |
| `commit_ack_i` | Commit 確認訊號 |
| `physical_waddr_i` | 要釋放的實體目的暫存器（從 commit 來） |
| `virtual_waddr_o` | 對應釋放的虛擬目的暫存器（傳給 map table） |

### 📥 Operand 映射
| Port 名稱 | 說明 |
|-----------|------|
| `rs1_physical_i` / `rs2_physical_i` / `rs3_physical_i` | operand 實體暫存器編號（由 map table 提供） |
| `rs1_virtual_o` / `rs2_virtual_o` / `rs3_virtual_o` | operand 對應的虛擬編號（供 forwarding 使用） |

### ⛳ 分支處理相關
| Port 名稱 | 說明 |
|-----------|------|
| `target_branch_addr_i` | 分支預測目標地址 |
| `mispredict_pc` | 錯誤分支對應 PC |
| `resolve_branch_i` | 宣告某個分支已 resolve（即使預測錯） |
| `mispredict_branch_i` | 該分支是否為錯誤預測 |

### 🧮 浮點運算支援
| Port 名稱 | 說明 |
|-----------|------|
| `we_fpr_i` | 對應 commit port 是否寫回浮點暫存器 |

---

### 🔗 Rename 功能涵蓋元件（內部）
- `maptable`：建立虛實映射
- `busytable`：追蹤哪些實體暫存器仍在使用中
- `freelist`：管理空閒的實體暫存器（供分配）
- `rename buffer`：暫存已 rename、等待 issue 的指令資訊
- `br_tag memory`：儲存分支快照，用於 rollback

---

## 🧠 Rename Stage - Rename FIFO 與暫存器指標解釋

```systemverilog
localparam int unsigned MEM_ENTRY_BIT = $clog2(CVA6Cfg.Nrrename);
localparam int unsigned BRENTRY_BIT   = $clog2(16);
```
- `MEM_ENTRY_BIT`：根據 rename buffer 大小 `Nrrename`，計算需要多少位元來編址。
- `BRENTRY_BIT`：Branch snapshot buffer 固定支援 16 個 entries。

```systemverilog
typedef struct packed {
    logic                           valid; 
    logic                           no_rename_rs1; 
    logic                           is_ctrl_flow; 
    ariane_pkg::scoreboard_entry_t  data;
} rename_mem_t;
```
- `rename_mem_t` 是儲存在 rename FIFO 的每筆資料：是否有效、是否需要 rename rs1、是否為控制流程、指令資訊。

```systemverilog
rename_mem_t [CVA6Cfg.Nrrename-1:0] rename_data_q, rename_data_n;
```
- 實際 rename FIFO：一組寄存器與 combinational buffer。

---

## 🧾 Rename 處理用的輔助訊號

```systemverilog
// 實體與虛擬 register 映射的結果 (由 maptable / busytable / freelist 輸入或輸出)
ariane_pkg::scoreboard_entry_t  [CVA6Cfg.NrissuePorts-1:0]  rename_entry;
logic [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-2:0]         Pr_rd_o_rob;
logic [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-1:0]         Pr_rs1_o;
logic [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-1:0]         Pr_rs2_o;
logic [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-1:0]         Pr_rs3_o;
logic [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-1:0]         Pr_rs1_o_rob;
logic [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-1:0]         Pr_rs2_o_rob;
logic [CVA6Cfg.NrissuePorts-1:0][REG_ADDR_SIZE-1:0]         Pr_rs3_o_rob;
```
- 這些訊號對應到 physical register allocator 結果，供後續指令 operand 依據進行 mapping。

```systemverilog
logic [CVA6Cfg.NrissuePorts-1:0]                            no_rename;
logic [CVA6Cfg.NrissuePorts-1:0]                            br_instr;
logic [CVA6Cfg.NrissuePorts-1:0]                            is_csr_imm;
logic [CVA6Cfg.NrissuePorts-1:0]                            rs1_no_rename;
logic [CVA6Cfg.NrissuePorts-1:0]                            virtual_waddr_valid;
logic [CVA6Cfg.NrissuePorts-1:0]                            rd_0_no_rename;
```
- 控制相關旗標：判斷哪些 register 要 rename、是否為分支、是否 CSR 使用立即值、不需 rename rs1、rd 為 x0 等。

```systemverilog
logic [MEM_ENTRY_BIT-1:0] issue_pointer_n, issue_pointer_q;
logic [MEM_ENTRY_BIT-1:0] commit_pointer_n, commit_pointer_q;
logic [MEM_ENTRY_BIT-1:0] issue_num, commit_num;
logic [BRENTRY_BIT-1:0]   br_push_ptr, br_pop_ptr;
logic [MEM_ENTRY_BIT:0]   mem_cnt;
logic                    mem_full;
```
- rename FIFO 的指標與狀態管理，控制資料 push/pop。

---

## 🚦 Rename Stage 發送與回傳（Handshake 與輸出）

```systemverilog
assign mem_full = ((mem_cnt-issue_ack_i[0]-issue_ack_i[1])>3'd1);
```
- 檢查 rename FIFO 是否接近滿（保守策略，留 1 筆空間）。

```systemverilog
assign rs1_no_rename[0] = (is_csr_use_imm(rename_instr_i[0].op) & rename_instr_i[0].use_zimm);
assign rs1_no_rename[1] = (is_csr_use_imm(rename_instr_i[1].op) & rename_instr_i[1].use_zimm);
```
- 如果是 CSR 且使用 zimm，代表 rs1 無需被 rename。

```systemverilog
assign issue_instr_o[0] = rename_data_q[commit_pointer_q].data;
assign issue_instr_valid_o[0] = rename_data_q[commit_pointer_q].valid;
assign is_ctrl_flow_o[0] = rename_data_q[commit_pointer_q].is_ctrl_flow;
assign issue_instr_o[1] = rename_data_q[commit_pointer_q+2'd1].data;
assign issue_instr_valid_o[1] = rename_data_q[commit_pointer_q+2'd1].valid;
assign is_ctrl_flow_o[1] = rename_data_q[commit_pointer_q+2'd1].is_ctrl_flow;
```
- rename FIFO 送出對應資料給 issue stage，透過 `commit_pointer_q` 指標取資料。

---


---

## 🔁 Rename FIFO 控制邏輯（always_comb）

這段程式碼描述 rename stage 的 FIFO 操作邏輯，包括：
- 指令發送至 issue stage 時如何出隊
- 指令從 ID stage 進入 rename FIFO 的條件與行為

```systemverilog
always_comb begin
    rename_data_n             = rename_data_q;          // 預設保持不變
    issue_pointer_n           = issue_pointer_q;
    commit_pointer_n          = commit_pointer_q;
    rename_ack_o              = 2'd0;                   // 預設不發出 ack
    issue_num                 = 2'd0;                   // 預設不計入新進指令
    commit_num                = 2'd0;                   // 預設不計入已送出指令

    // ========================================================================
    // ❶ 將已經 rename 完畢的指令發送給 issue stage（由 commit_pointer_q 指向）
    // ========================================================================
    if (issue_ack_i[0] & issue_ack_i[1]) begin 
        rename_data_n[commit_pointer_q     ].valid          = 1'b0;
        rename_data_n[commit_pointer_q     ].no_rename_rs1  = 1'b0;
        rename_data_n[commit_pointer_q+2'd1].valid          = 1'b0;
        rename_data_n[commit_pointer_q+2'd1].no_rename_rs1  = 1'b0;
        commit_pointer_n                                    = commit_pointer_n + 3'd2;
        commit_num                                          = 2'd2; // 記錄發出數量供 mem_cnt 更新
    end else if (issue_ack_i[0]) begin 
        rename_data_n[commit_pointer_q     ].valid          = 1'b0;
        rename_data_n[commit_pointer_q     ].no_rename_rs1  = 1'b0;
        commit_pointer_n                                    = commit_pointer_n + 3'd1;
        commit_num                                          = 2'd1;
    end

    // ========================================================================
    // ❷ 接收從 ID stage 傳入的兩條指令並寫入 rename FIFO（由 issue_pointer_q 指向）
    // ========================================================================
    if(((mem_cnt-issue_ack_i[0]-issue_ack_i[1])<3'd1) & (rename_instr_valid_i==2'b11)) begin 
        issue_pointer_n                                   = issue_pointer_n + 2'd2;
        issue_num                                         = 2'd2; // 記錄進入數量供 mem_cnt 更新
        rename_ack_o [0]                                  = 1'b1;
        rename_ack_o [1]                                  = 1'b1;
        rename_data_n[issue_pointer_q     ].valid         = 1'b1;
        rename_data_n[issue_pointer_q     ].data          = rename_entry      [0];
        rename_data_n[issue_pointer_q     ].is_ctrl_flow  = is_ctrl_flow_i    [0];
        rename_data_n[issue_pointer_q     ].no_rename_rs1 = rs1_no_rename     [0];
        rename_data_n[issue_pointer_q+2'd1].valid         = 1'b1;
        rename_data_n[issue_pointer_q+2'd1].data          = rename_entry      [1];
        rename_data_n[issue_pointer_q+2'd1].is_ctrl_flow  = is_ctrl_flow_i    [1];
        rename_data_n[issue_pointer_q+2'd1].no_rename_rs1 = rs1_no_rename     [1];
    end else if (rename_instr_valid_i[0] & !mem_full) begin
        issue_pointer_n                                   = issue_pointer_n + 2'd1;
        issue_num                                         = 2'd1;
        rename_ack_o [0]                                  = 1'b1;
        rename_data_n[issue_pointer_q     ].valid         = 1'b1;
        rename_data_n[issue_pointer_q     ].data          = rename_entry      [0];
        rename_data_n[issue_pointer_q     ].is_ctrl_flow  = is_ctrl_flow_i    [0];
        rename_data_n[issue_pointer_q     ].no_rename_rs1 = rs1_no_rename     [0];
    end
end
```

---

### 📝 解釋重點
| 行為情境                             | 說明 |
|--------------------------------------|------|
| `issue_ack_i` 表示 rename buffer 資料被 issue | 將對應資料從 rename FIFO "清空"（valid = 0）並更新 commit pointer |
| `rename_instr_valid_i` 表示有來自 ID 的新指令 | 將兩條指令寫入 rename FIFO，並拉高對應的 `rename_ack_o` 告知 ID stage |
| `mem_cnt` 保持 rename FIFO 的佔用狀態     | 實際更新動作在時脈區塊（`always_ff`），此處只記錄即將變化的數量 |

---

---

## 🌀 同步更新 Rename Buffer 內容（Commit 後回寫 RS）

這段程式碼實現了：
- 當指令 commit（送出）後，若其寫入的目的暫存器是其他指令的來源 rs1/rs2，則**同步更新那些等待中的指令在 rename buffer 裡的 rs1/rs2** 值。
- 避免使用過時的虛擬暫存器別名（physical register aliasing）。

```systemverilog
for (int unsigned j = 0; j < CVA6Cfg.Nrrename; j++) begin 
    // ---------- rs1 更新 ----------
    if ((commit_instr_o[0].rd == rename_data_q[j].data.rs1) && commit_ack_i[0] && rename_data_q[j].valid && !rename_data_q[j].no_rename_rs1 &&
        ((is_rs1_fpr(rename_data_q[j].data.op) && we_fpr_i[0]) ||
         ((commit_instr_o[0].rd != '0) && !is_rs1_fpr(rename_data_q[j].data.op) && !we_fpr_i[0]))) begin 
        rename_data_n[j].data.rs1 = {1'd1, virtual_waddr_o[0][REG_ADDR_SIZE-2:0]};
    end else if ((commit_instr_o[1].rd == rename_data_q[j].data.rs1) && commit_ack_i[1] && rename_data_q[j].valid && !rename_data_q[j].no_rename_rs1 &&
                 ((is_rs1_fpr(rename_data_q[j].data.op) && we_fpr_i[1]) ||
                  ((commit_instr_o[1].rd != '0) && !is_rs1_fpr(rename_data_q[j].data.op) && !we_fpr_i[1]))) begin
        rename_data_n[j].data.rs1 = {1'd1, virtual_waddr_o[1][REG_ADDR_SIZE-2:0]};
    end

    // ---------- rs2 更新 ----------
    if ((commit_instr_o[0].rd == rename_data_q[j].data.rs2) && commit_ack_i[0] && rename_data_q[j].valid &&
        ((is_rs2_fpr(rename_data_q[j].data.op) && we_fpr_i[0]) ||
         ((commit_instr_o[0].rd != '0) && !is_rs2_fpr(rename_data_q[j].data.op) && !we_fpr_i[0]))) begin 
        rename_data_n[j].data.rs2 = {1'd1, virtual_waddr_o[0][REG_ADDR_SIZE-2:0]};
    end else if ((commit_instr_o[1].rd == rename_data_q[j].data.rs2) && commit_ack_i[1] && rename_data_q[j].valid &&
                 ((is_rs2_fpr(rename_data_q[j].data.op) && we_fpr_i[1]) ||
                  ((commit_instr_o[1].rd != '0) && !is_rs2_fpr(rename_data_q[j].data.op) && !we_fpr_i[1]))) begin
        rename_data_n[j].data.rs2 = {1'd1, virtual_waddr_o[1][REG_ADDR_SIZE-2:0]};
    end

    // ---------- 特殊: result 欄位用來存放浮點 immediate 寫入值 ----------
    if (is_imm_fpr(rename_data_q[j].data.op)) begin 
        if ((commit_instr_o[0].rd == rename_data_q[j].data.result[5:0]) && commit_ack_i[0] && rename_data_q[j].valid && we_fpr_i[0]) begin 
            rename_data_n[j].data.result = {58'd0, 1'd1, virtual_waddr_o[0][REG_ADDR_SIZE-2:0]};
        end else if ((commit_instr_o[1].rd == rename_data_q[j].data.result[5:0]) && commit_ack_i[1] && rename_data_q[j].valid && we_fpr_i[1]) begin
            rename_data_n[j].data.result = {58'd0, 1'd1, virtual_waddr_o[1][REG_ADDR_SIZE-2:0]};
        end 
    end
end

// -------- Flush 條件下清空 FIFO --------
if (flush_i | flush_unissied_instr_i) begin 
    for (int unsigned i = 0; i < CVA6Cfg.Nrrename; i++) begin
        rename_data_n[i].valid = 1'b0;
    end
end
```

---

## 🧠 寄存器（always_ff）區塊

這段程式碼描述 rename buffer 與相關指標的寄存器邏輯：
- 當 flush 產生時重設
- 否則更新指標與記錄內容

```systemverilog
always_ff @(posedge clk_i or negedge rst_ni) begin
    if (~rst_ni) begin 
        mem_cnt             <= '0;
        rename_data_q       <= '0;
        issue_pointer_q     <= '0;
        commit_pointer_q    <= '0;
    end else if (flush_i | flush_unissied_instr_i) begin 
        mem_cnt             <= '0;
        rename_data_q       <= '0;
        issue_pointer_q     <= '0;
        commit_pointer_q    <= '0;
    end else begin    
        rename_data_q       <= rename_data_n;
        issue_pointer_q     <= issue_pointer_n;
        commit_pointer_q    <= commit_pointer_n;
        mem_cnt             <= mem_cnt + issue_num - commit_num;
    end
end
```

---

### 🔍 小結
| 機制 | 說明 |
|------|------|
| 🔁 更新 rename 中指令的 source | 目的：確保早期進入 rename buffer 的指令可以即時獲得最新的暫存器映射資訊 |
| 📥 mem_cnt + / - | 為確保 buffer 不會溢位，透過計數器追蹤 rename buffer 使用量 |
| 🔄 flush 清除條件 | 發生 mispredict 或 interrupt 等情況時，需清空 rename buffer 內容與指標 |


---

## 🧠 Rename Stage 子模組整理

以下為 rename stage 中使用的三個關鍵模組說明：freelist、busytable、maptable。

---

### 🧾 `freelist`
```systemverilog
freelist #(
    .CVA6Cfg ( CVA6Cfg )
) i_freelist(
    .clk_i, .rst_ni,
    .flush_i,
    .flush_unissied_instr_i,
    .issue_instr_valid_i ( rename_instr_valid_i ),
    .issue_ack_o          ( rename_ack_o ),
    .no_rename_i          ( no_rename ),
    .commit_instr_o       ( commit_instr_o ),
    .commit_ack_i         ( commit_ack_i ),
    .Pr_rd_o_rob          ( Pr_rd_o_rob ),
    .br_instr_i           ( br_instr ),
    .br_push_ptr          ( br_push_ptr ),
    .br_pop_ptr           ( br_pop_ptr )
);
```
#### 📌 功能：
- 管理空閒的 **physical register (PRF)**
- 提供給 map table 使用的寫入位址（`Pr_rd_o_rob`）
- 若分支錯誤（mis-predict），可透過 `br_push_ptr` 和 `br_pop_ptr` 回收對應 snapshot 中的暫存器
- 支援 flush 功能，清空目前 rename 中尚未發出的 rename 結果

---

### 🧾 `busytable`
```systemverilog
busytable #(
    .CVA6Cfg ( CVA6Cfg )
) i_busytable(
    .clk_i, .rst_ni,
    .flush_i,
    .flush_unissied_instr_i,
    .issue_instr_i        ( rename_instr_i ),
    .issue_instr_valid_i  ( rename_instr_valid_i ),
    .issue_ack_o          ( rename_ack_o ),
    .no_rename_i          ( no_rename ),
    .commit_instr_o       ( commit_instr_o ),
    .commit_ack_i         ( commit_ack_i ),
    .Pr_rd_o_rob          ( Pr_rd_o_rob ),
    .Pr_rs1_o             ( Pr_rs1_o ),
    .Pr_rs2_o             ( Pr_rs2_o ),
    .Pr_rs3_o             ( Pr_rs3_o ),
    .Pr_rs1_o_rob         ( Pr_rs1_o_rob ),
    .Pr_rs2_o_rob         ( Pr_rs2_o_rob ),
    .Pr_rs3_o_rob         ( Pr_rs3_o_rob ),
    .br_instr_i           ( br_instr ),
    .br_push_ptr          ( br_push_ptr ),
    .br_pop_ptr           ( br_pop_ptr )
);
```
#### 📌 功能：
- 管理 physical register 的 **忙碌狀態（busy bit）**
- 發出給 Issue 單元使用的 ready 判斷依據
- 控制 `rs1/rs2/rs3` 是否可從 PRF 讀值
- 支援 rollback snapshot，針對分支錯誤的暫存器狀態復原

---

### 🧾 `maptable`
```systemverilog
maptable #(
    .CVA6Cfg ( CVA6Cfg )
) i_maptable(
    .clk_i, .rst_ni,
    .flush_i,
    .flush_unissied_instr_i,
    .issue_instr_i         ( rename_instr_i ),
    .issue_instr_valid_i   ( rename_instr_valid_i ),
    .issue_ack_o           ( rename_ack_o ),
    .no_rename_i           ( no_rename ),
    .commit_instr_o        ( commit_instr_o ),
    .commit_ack_i          ( commit_ack_i ),
    .Pr_rd_o_rob           ( Pr_rd_o_rob ),
    .physical_waddr_i      ( physical_waddr_i ),
    .Pr_rs1_o              ( Pr_rs1_o ),
    .Pr_rs2_o              ( Pr_rs2_o ),
    .Pr_rs3_o              ( Pr_rs3_o ),
    .virtual_waddr_o       ( virtual_waddr_o ),
    .virtual_waddr_valid   ( virtual_waddr_valid ),
    .rs1_physical_i        ( rs1_physical_i ),
    .rs2_physical_i        ( rs2_physical_i ),
    .rs3_physical_i        ( rs3_physical_i ),
    .rs1_virtual_o         ( rs1_virtual_o ),
    .rs2_virtual_o         ( rs2_virtual_o ),
    .rs3_virtual_o         ( rs3_virtual_o ),
    .br_instr_i            ( br_instr ),
    .br_push_ptr           ( br_push_ptr ),
    .br_pop_ptr            ( br_pop_ptr )
);
```
#### 📌 功能：
- 實作 register renaming 對應：`ARF -> PRF`
- 寫入時給出新的 `virtual_waddr` 與 `Pr_rd`
- 同時也維護 snapshot 結構支援分支復原
- 提供對應 `rs1/rs2/rs3` 的映射結果給後段（如 issue stage）使用

---

👉 這三個模組是整個 Tomasulo 架構中 rename/dispatch 的關鍵支柱，用來維持資料一致性、資源管理，以及支援精確例外與分支回復。

---

## 🧪 Rename Entry 生成與資料寫入

這段邏輯主要負責將 ID 階段解碼後的 `scoreboard_entry_t` 進行目的暫存器映射、來源暫存器讀出對應實體暫存器值後，封裝成 `rename_entry` 發送給後續發射階段。

```systemverilog
// -------------------------------------------
// 將原始欄位直接傳遞到 rename_entry
// -------------------------------------------
assign rename_entry[0].pc             = rename_instr_i[0].pc;
assign rename_entry[0].trans_id       = rename_instr_i[0].trans_id;
assign rename_entry[0].fu             = rename_instr_i[0].fu;
assign rename_entry[0].op             = rename_instr_i[0].op;
assign rename_entry[0].use_imm        = rename_instr_i[0].use_imm;
assign rename_entry[0].valid          = rename_instr_i[0].valid;
assign rename_entry[0].use_zimm       = rename_instr_i[0].use_zimm;
assign rename_entry[0].use_pc         = rename_instr_i[0].use_pc;
assign rename_entry[0].ex             = rename_instr_i[0].ex;
assign rename_entry[0].bp             = rename_instr_i[0].bp;
assign rename_entry[0].is_compressed  = rename_instr_i[0].is_compressed;
assign rename_entry[0].vfp            = rename_instr_i[0].vfp;
assign rename_entry[0].rs1_rdata      = rename_instr_i[0].rs1_rdata;
assign rename_entry[0].rs2_rdata      = rename_instr_i[0].rs2_rdata;
assign rename_entry[0].lsu_addr       = rename_instr_i[0].lsu_addr;
assign rename_entry[0].lsu_rmask      = rename_instr_i[0].lsu_rmask;
assign rename_entry[0].lsu_wmask      = rename_instr_i[0].lsu_wmask;
assign rename_entry[0].lsu_wdata      = rename_instr_i[0].lsu_wdata;
assign rename_entry[0].rd             = (no_rename[0]) ? rename_instr_i[0].rd : {1'b0, Pr_rd_o_rob[0]};

// 處理 rs1 實體對應邏輯
always_comb begin 
    rename_entry[0].rs1 = '0;
    if (rs1_no_rename[0]) begin
        rename_entry[0].rs1 = {1'd1, rename_instr_i[0].rs1};
    end else if ((virtual_waddr_o[0] == rename_instr_i[0].rs1) && commit_ack_i[0] && virtual_waddr_valid[0] &&
                ((is_rs1_fpr(rename_instr_i[0].op) && we_fpr_i[0]) || (!is_rs1_fpr(rename_instr_i[0].op) && !we_fpr_i[0]))) begin
        rename_entry[0].rs1 = {1'd1, virtual_waddr_o[0][REG_ADDR_SIZE-2:0]};
    end else if ((virtual_waddr_o[1] == rename_instr_i[0].rs1) && commit_ack_i[1] && virtual_waddr_valid[1] &&
                ((is_rs1_fpr(rename_instr_i[0].op) && we_fpr_i[1]) || (!is_rs1_fpr(rename_instr_i[0].op) && !we_fpr_i[1]))) begin
        rename_entry[0].rs1 = {1'd1, virtual_waddr_o[1][REG_ADDR_SIZE-2:0]};
    end else begin
        rename_entry[0].rs1 = Pr_rs1_o_rob[0];
    end
end

// 處理 rs2 實體對應邏輯
always_comb begin 
    rename_entry[0].rs2 = '0;
    if ((virtual_waddr_o[0] == rename_instr_i[0].rs2) && commit_ack_i[0] && virtual_waddr_valid[0] &&
        ((is_rs2_fpr(rename_instr_i[0].op) && we_fpr_i[0]) || (!is_rs2_fpr(rename_instr_i[0].op) && !we_fpr_i[0]))) begin
        rename_entry[0].rs2 = {1'd1, virtual_waddr_o[0][REG_ADDR_SIZE-2:0]};
    end else if ((virtual_waddr_o[1] == rename_instr_i[0].rs2) && commit_ack_i[1] && virtual_waddr_valid[1] &&
        ((is_rs2_fpr(rename_instr_i[0].op) && we_fpr_i[1]) || (!is_rs2_fpr(rename_instr_i[0].op) && !we_fpr_i[1]))) begin
        rename_entry[0].rs2 = {1'd1, virtual_waddr_o[1][REG_ADDR_SIZE-2:0]};
    end else begin
        rename_entry[0].rs2 = Pr_rs2_o_rob[0];
    end
end

// 若 result 欄位為立即數浮點暫存器位址，也需要進行更新
always_comb begin 
    if (is_imm_fpr(rename_instr_i[0].op)) begin
        if ((virtual_waddr_o[0] == {1'd0, rename_instr_i[0].result[REG_ADDR_SIZE-2:0]}) && commit_ack_i[0] && virtual_waddr_valid[0] && we_fpr_i[0]) begin
            rename_entry[0].result = {58'd0, 1'd1, virtual_waddr_o[0][REG_ADDR_SIZE-2:0]};
        end else if ((virtual_waddr_o[1] == {1'd0, rename_instr_i[0].result[REG_ADDR_SIZE-2:0]}) && commit_ack_i[1] && virtual_waddr_valid[1] && we_fpr_i[1]) begin
            rename_entry[0].result = {58'd0, 1'd1, virtual_waddr_o[1][REG_ADDR_SIZE-2:0]};
        end else begin
            rename_entry[0].result = {58'd0, Pr_rs3_o_rob[0]};
        end
    end else begin
        rename_entry[0].result = rename_instr_i[0].result;
    end
end
```

---

✅ 這段程式碼可讓 rename_entry[0] 完整映射 rename 後的結果，正確傳遞給後續 issue stage。

### rename_entry[1]
```systemverilog
assign rename_entry[1].pc             = rename_instr_i[1].pc;
assign rename_entry[1].trans_id       = rename_instr_i[1].trans_id;
assign rename_entry[1].fu             = rename_instr_i[1].fu;
assign rename_entry[1].op             = rename_instr_i[1].op;
assign rename_entry[1].use_imm        = rename_instr_i[1].use_imm;
assign rename_entry[1].valid          = rename_instr_i[1].valid;
assign rename_entry[1].use_zimm       = rename_instr_i[1].use_zimm;
assign rename_entry[1].use_pc         = rename_instr_i[1].use_pc;
assign rename_entry[1].ex             = rename_instr_i[1].ex;
assign rename_entry[1].bp             = rename_instr_i[1].bp;
assign rename_entry[1].is_compressed  = rename_instr_i[1].is_compressed;
assign rename_entry[1].vfp            = rename_instr_i[1].vfp;
assign rename_entry[1].rs1_rdata      = rename_instr_i[1].rs1_rdata;
assign rename_entry[1].rs2_rdata      = rename_instr_i[1].rs2_rdata;
assign rename_entry[1].lsu_addr       = rename_instr_i[1].lsu_addr;
assign rename_entry[1].lsu_rmask      = rename_instr_i[1].lsu_rmask;
assign rename_entry[1].lsu_wmask      = rename_instr_i[1].lsu_wmask;
assign rename_entry[1].lsu_wdata      = rename_instr_i[1].lsu_wdata;
assign rename_entry[1].rd             = (no_rename[1]) ? rename_instr_i[1].rd : {1'b0, Pr_rd_o_rob[1]};

// 處理 rs1 映射（避免與 rename[0] 發生寫後讀衝突）
always_comb begin 
    rename_entry[1].rs1 = '0;
    if (rs1_no_rename[1]) begin
        rename_entry[1].rs1 = {1'd1, rename_instr_i[1].rs1[REG_ADDR_SIZE-2:0]};
    end else if ((virtual_waddr_o[0] == rename_instr_i[1].rs1) && commit_ack_i[0] && virtual_waddr_valid[0] &&
                !((virtual_waddr_o[0] == rename_instr_i[0].rd) && ((is_rs1_fpr(rename_instr_i[1].op)) == is_rd_fpr(rename_instr_i[0].op)) && !no_rename[0]) &&
                ((is_rs1_fpr(rename_instr_i[1].op) && we_fpr_i[0]) || (!is_rs1_fpr(rename_instr_i[1].op) && !we_fpr_i[0]))) begin
        rename_entry[1].rs1 = {1'd1, virtual_waddr_o[0][REG_ADDR_SIZE-2:0]};
    end else if ((virtual_waddr_o[1] == rename_instr_i[1].rs1) && commit_ack_i[1] && virtual_waddr_valid[1] &&
                !((virtual_waddr_o[1] == rename_instr_i[0].rd) && ((is_rs1_fpr(rename_instr_i[1].op)) == is_rd_fpr(rename_instr_i[0].op)) && !no_rename[0]) &&
                ((is_rs1_fpr(rename_instr_i[1].op) && we_fpr_i[1]) || (!is_rs1_fpr(rename_instr_i[1].op) && !we_fpr_i[1]))) begin
        rename_entry[1].rs1 = {1'd1, virtual_waddr_o[1][REG_ADDR_SIZE-2:0]};
    end else begin
        rename_entry[1].rs1 = Pr_rs1_o_rob[1];
    end
end

// 處理 rs2 映射（避免與 rename[0] 發生寫後讀衝突）
always_comb begin 
    rename_entry[1].rs2 = '0;
    if ((virtual_waddr_o[0] == rename_instr_i[1].rs2) && commit_ack_i[0] && virtual_waddr_valid[0] &&
        !((virtual_waddr_o[0] == rename_instr_i[0].rd) && ((is_rs2_fpr(rename_instr_i[1].op)) == is_rd_fpr(rename_instr_i[0].op)) && !no_rename[0]) &&
        ((is_rs2_fpr(rename_instr_i[1].op) && we_fpr_i[0]) || (!is_rs2_fpr(rename_instr_i[1].op) && !we_fpr_i[0]))) begin
        rename_entry[1].rs2 = {1'd1, virtual_waddr_o[0][REG_ADDR_SIZE-2:0]};
    end else if ((virtual_waddr_o[1] == rename_instr_i[1].rs2) && commit_ack_i[1] && virtual_waddr_valid[1] &&
        !((virtual_waddr_o[1] == rename_instr_i[0].rd) && ((is_rs2_fpr(rename_instr_i[1].op)) == is_rd_fpr(rename_instr_i[0].op)) && !no_rename[0]) &&
        ((is_rs2_fpr(rename_instr_i[1].op) && we_fpr_i[1]) || (!is_rs2_fpr(rename_instr_i[1].op) && !we_fpr_i[1]))) begin
        rename_entry[1].rs2 = {1'd1, virtual_waddr_o[1][REG_ADDR_SIZE-2:0]};
    end else begin
        rename_entry[1].rs2 = Pr_rs2_o_rob[1];
    end

    if (rename_entry[1].fu == 4'd6) begin
        rename_entry[1].rs2 = '0;
    end
end

// 處理浮點立即數（例如 fmv.x.w）目的暫存器為 rs3 的情況
always_comb begin 
    if (is_imm_fpr(rename_instr_i[1].op)) begin
        if ((virtual_waddr_o[0] == {1'd0, rename_instr_i[1].result[REG_ADDR_SIZE-2:0]}) && commit_ack_i[0] && virtual_waddr_valid[0] &&
            !((virtual_waddr_o[0] == rename_instr_i[0].rd) && ((is_imm_fpr(rename_instr_i[1].op)) == is_rd_fpr(rename_instr_i[0].op)) && !no_rename[0]) && we_fpr_i[0]) begin
            rename_entry[1].result = {58'd0, 1'd1, virtual_waddr_o[0][REG_ADDR_SIZE-2:0]};
        end else if ((virtual_waddr_o[1] == {1'd0, rename_instr_i[1].result[REG_ADDR_SIZE-2:0]}) && commit_ack_i[1] && virtual_waddr_valid[1] &&
            !((virtual_waddr_o[1] == rename_instr_i[0].rd) && ((is_imm_fpr(rename_instr_i[1].op)) == is_rd_fpr(rename_instr_i[0].op)) && !no_rename[0]) && we_fpr_i[1]) begin
            rename_entry[1].result = {58'd0, 1'd1, virtual_waddr_o[1][REG_ADDR_SIZE-2:0]};
        end else begin
            rename_entry[1].result = {58'd0, Pr_rs3_o_rob[1]};
        end
    end else begin
        rename_entry[1].result = rename_instr_i[1].result;
    end
end
```

---

✅ `rename_entry[1]` 處理的重點在於避免與前一條 rename_entry[0] 發生 RAW hazard，同時針對浮點立即數進行轉換處理。

---

## 🧪 Rename Entry 生成與資料寫入

這段邏輯主要負責將 ID 階段解碼後的 `scoreboard_entry_t` 進行目的暫存器映射、來源暫存器讀出對應實體暫存器值後，封裝成 `rename_entry` 發送給後續發射階段。

（略過 rename_entry[0], rename_entry[1] 詳細設定）

---

## 🚫 No Rename Instruction 判斷邏輯

```systemverilog
assign rd_0_no_rename[0] = (is_rd_fpr(rename_instr_i[0].op)) ? 1'd0 : (rename_instr_i[0].rd == 6'd0);
assign rd_0_no_rename[1] = (is_rd_fpr(rename_instr_i[1].op)) ? 1'd0 : (rename_instr_i[1].rd == 6'd0);

assign no_rename[0] = (
    (rename_instr_i[0].op == 8'h00 && rename_instr_i[0].fu == 4'b0100 && rename_instr_i[0].rd == 6'd0) ||
    op_is_branch(rename_instr_i[0].op) ||
    is_csr_no_rename(rename_instr_i[0].op) ||
    (is_no_rename_rd_zero(rename_instr_i[0].op) && rename_instr_i[0].rd == 6'd0) ||
    is_store(rename_instr_i[0].op) ||
    rd_0_no_rename[0] || flush_unissied_instr_i || flush_i
);

assign no_rename[1] = (
    (rename_instr_i[1].op == 8'h00 && rename_instr_i[1].fu == 4'b0100 && rename_instr_i[1].rd == 6'd0) ||
    op_is_branch(rename_instr_i[1].op) ||
    is_csr_no_rename(rename_instr_i[1].op) ||
    (is_no_rename_rd_zero(rename_instr_i[1].op) && rename_instr_i[1].rd == 6'd0) ||
    is_store(rename_instr_i[1].op) ||
    rd_0_no_rename[1] || flush_unissied_instr_i || flush_i
);
```

📌 說明：
- 判斷該指令是否需要 register rename。
- 若屬於 branch、CSR、store、或目的暫存器為 x0，則不需 rename。
- `flush_unissied_instr_i` 或 `flush_i` 為清除 pipeline 的訊號，也會導致不 rename。

---

## 🔁 Branch Tag Snapshot

該邏輯用於追蹤正在 rename 階段中發出的 branch instruction，以便之後發生錯誤預測（mispredict）時能夠正確回溯。

```systemverilog
assign br_instr[0] = (rename_instr_i[0].fu == 4'd4);
assign br_instr[1] = (rename_instr_i[1].fu == 4'd4);

assign issue1_is_branch = (rename_instr_i[0].fu == 4'd4 && rename_instr_valid_i[0] && rename_ack_o[0] && !flush_unissied_instr_i);
assign issue2_is_branch = (rename_instr_i[1].fu == 4'd4 && rename_instr_valid_i[1] && rename_ack_o[1] && !flush_unissied_instr_i);

always_ff @(posedge clk_i or negedge rst_ni) begin
    if (~rst_ni) begin
        br_push_ptr <= 4'd0;
    end else if (issue1_is_branch && issue2_is_branch) begin
        br_push_ptr <= br_push_ptr + 4'd2;
    end else if (issue1_is_branch || issue2_is_branch) begin
        br_push_ptr <= br_push_ptr + 4'd1;
    end
end

always_ff @(posedge clk_i or negedge rst_ni) begin
    if (~rst_ni) begin
        br_pop_ptr <= 4'd0;
    end else if (flush_unissied_instr_i) begin
        br_pop_ptr <= br_push_ptr;
    end else if (resolve_branch_i) begin
        br_pop_ptr <= br_pop_ptr + 4'd1;
    end
end
```

📌 說明：
- `br_push_ptr`：遇到 branch 指令就記錄，目前 rename 階段中有多少 branch 指令。
- `br_pop_ptr`：分支 resolve 或 mispredict 發生時依據該指標回溯狀態。
- 實作上可配合 `maptable`, `freelist`, `busytable` 等模組來進行 rollback。

---

