### 🧠 ROB 子模組介面說明（Reorder Buffer）

```systemverilog
module rob #(
    parameter config_pkg::cva6_cfg_t CVA6Cfg = config_pkg::cva6_cfg_empty,
    parameter bit IsRVFI = bit'(0),
    parameter type rs3_len_t = logic,
    parameter int unsigned NR_ENTRIES = 8  // ROB 大小（必須為 2 的冪次）
) (
    input  logic clk_i,                   // 時鐘
    input  logic rst_ni,                  // 非同步 reset，低有效
    input  logic flush_unissued_instr_i,  // Flush 所有未發派的指令（如 branch mispredict）
    input  logic unresolved_branch_i,     // 未解決的分支（保留）
    input  logic flush_i,                 // Flush 所有狀態

    output logic sb_full_o,               // scoreboard 是否已滿（代表 ROB 滿了）

    // Register write-after-write hazard 檢查
    output ariane_pkg::fu_t [2**ariane_pkg::REG_ADDR_SIZE-1:0] rd_clobber_gpr_o, // 每個 GPR 寄存器目前的寫入來源功能單元
    output ariane_pkg::fu_t [2**ariane_pkg::REG_ADDR_SIZE-1:0] rd_clobber_fpr_o, // 每個 FPR 寄存器目前的寫入來源功能單元

    // 發派時輸入實體來源暫存器（rs1/rs2/rs3），ROB 內記錄這些值
    input  logic [CVA6Cfg.NrissuePorts-1:0][ariane_pkg::REG_ADDR_SIZE-1:0] rs1_i,
    input  logic [CVA6Cfg.NrissuePorts-1:0][ariane_pkg::REG_ADDR_SIZE-1:0] rs2_i,
    input  logic [CVA6Cfg.NrissuePorts-1:0][ariane_pkg::REG_ADDR_SIZE-1:0] rs3_i,

    // 給 Issue Read Operand 階段讀回 ROB 中 rs1/rs2/rs3 的值
    output riscv::xlen_t [CVA6Cfg.NrissuePorts-1:0] rs1_o,
    output riscv::xlen_t [CVA6Cfg.NrissuePorts-1:0] rs2_o,
    output rs3_len_t     [CVA6Cfg.NrissuePorts-1:0] rs3_o,

    output logic [CVA6Cfg.NrissuePorts-1:0] rs1_valid_o, // rs1 資料是否 valid
    output logic [CVA6Cfg.NrissuePorts-1:0] rs2_valid_o, // rs2 資料是否 valid
    output logic [CVA6Cfg.NrissuePorts-1:0] rs3_valid_o, // rs3 資料是否 valid

    // Commit 階段：ROB 輸出一批可被 commit 的指令
    output scoreboard_entry_t [CVA6Cfg.NrCommitPorts-1:0] commit_instr_o,
    input  logic [CVA6Cfg.NrCommitPorts-1:0] commit_ack_i,

    // Rename 階段寫入新的指令（Decoded）
    input  scoreboard_entry_t [CVA6Cfg.NrCommitPorts-1:0] decoded_instr_i,
    input  logic [CVA6Cfg.NrCommitPorts-1:0] decoded_instr_valid_i,
    output logic [CVA6Cfg.NrCommitPorts-1:0] decoded_instr_ack_o,

    // 發派到 IRO（Issue/ReadOperands）階段的指令
    output scoreboard_entry_t [CVA6Cfg.NrCommitPorts-1:0] issue_instr_o,
    output logic [CVA6Cfg.NrCommitPorts-1:0] issue_instr_valid_o,
    input  logic [CVA6Cfg.NrCommitPorts-1:0] issue_ack_i,

    // 寫回階段資訊（依據 trans_id 判定哪筆指令完成）
    input  logic [CVA6Cfg.NrWbPorts-1:0][TRANS_ID_BITS-1:0] trans_id_i,
    input  logic [CVA6Cfg.NrWbPorts-1:0][riscv::XLEN-1:0]   wbdata_i,
    input  exception_t [CVA6Cfg.NrWbPorts-1:0]              ex_i,
    input  logic [CVA6Cfg.NrWbPorts-1:0]                    wt_valid_i,
    input  bp_resolve_t                                     resolved_branch_i,
    input  logic                                            x_we_i,

    // Load/Store Address 回傳資訊（for exception flush）
    input  [riscv::VLEN-1:0]                 lsu_addr_i,
    input  [(riscv::XLEN/8)-1:0]             lsu_rmask_i,
    input  [(riscv::XLEN/8)-1:0]             lsu_wmask_i,
    input  [TRANS_ID_BITS-1:0]              lsu_addr_trans_id_i,

    // Forwarding 資料（XLEN 寬度）
    input riscv::xlen_t                     rs1_forwarding_i,
    input riscv::xlen_t                     rs2_forwarding_i,

    // Flush 資訊輸出
    output logic [NR_ENTRIES-1:0]           flush_entry,
    output logic [NR_ENTRIES-1:0][3:0]      flush_fu,
    output logic [NR_ENTRIES-1:0][3:0]      flush_trans_id,
    output logic [NR_ENTRIES-1:0][63:0]     flush_lsu_addr
);
```

---

### 📘 小結：ROB 模組職責整理

| 階段         | 功能描述                                                                 |
|--------------|--------------------------------------------------------------------------|
| Rename       | 新增指令進入 ROB                                                         |
| Issue        | 將 valid 指令送入 Issue & Read Operands 階段                            |
| Read Operands| 提供來源暫存器值與 valid 標記                                            |
| Writeback    | 根據 `trans_id` 與 `wbdata` 將結果寫回至 ROB 對應 entry                 |
| Commit       | 將已完成指令送出（符合 commit 條件）                                     |
| Exception    | 接收分支解決與 LSU 地址，用於 trigger flush 或 exception 處理            |
| Forwarding   | 回傳 forwarding 值給 ALU、LSU、FPU 等單元                                |
| Flush 處理    | 輸出 `flush_entry` 等相關資訊以支援整體 pipeline 清空/回復機制          |
