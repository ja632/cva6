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

