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
