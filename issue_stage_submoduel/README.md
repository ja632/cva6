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

---

### 🧠 ROB: Register Order Buffer（內部暫存區記憶體）定義與控制信號


```systemverilog
localparam int unsigned BITS_ENTRIES = $clog2(NR_ENTRIES);  // 根據 rob 大小決定編號位元數

// ------------------------------------------------------------------------------------
// ROB memory 結構，每筆資料包含發派與浮點資訊，以及 scoreboard entry 資訊
// ------------------------------------------------------------------------------------
typedef struct packed {
  logic issued;                         // 是否已經發派給功能單元
  logic is_rd_fpr_flag;                // 寫回目的暫存器是否為浮點暫存器
  ariane_pkg::scoreboard_entry_t sbe; // 指令資訊（scoreboard entry）
} sb_mem_t;

sb_mem_t [NR_ENTRIES-1:0] mem_q, mem_n;  // ROB 的記憶體陣列（目前狀態與下一狀態）

// ------------------------------------------------------------------------------------
// 控制變數
// ------------------------------------------------------------------------------------
logic issue_full;                // 指示 ROB 是否已滿（即無法再發派）
logic issue_en_0;                // 是否允許發派 port 0
logic issue_en_1;                // 是否允許發派 port 1
logic flush_instr;               // 是否需 flush 特定指令
logic case_flush;                // 根據 pointer 決定 flush 的方式（wrap around）
logic enable_dual_issue;         // 是否允許雙發派（根據 ROB 空間）

// ------------------------------------------------------------------------------------
// 指標與計數器（ROB 指標）
// ------------------------------------------------------------------------------------
logic [BITS_ENTRIES:0] issue_cnt_n, issue_cnt_q;      // 發派指令數（帶進位）
logic [NR_ENTRIES-1:0] num_flush_q;                   // flush 資訊
logic [NR_ENTRIES-1:0] flush_entry_n;                 // 下個 flush 寫入
logic [NR_ENTRIES-1:0] num_flush_flag;                // 當前 flush flag
logic [NR_ENTRIES-1:0] num_flush_flag_n;              // 下一週期的 flush flag
logic [BITS_ENTRIES:0] flush_branch_trans_id;         // 分支 trans id

// ------------------------------------------------------------------------------------
// commit / issue pointer 計算
// ------------------------------------------------------------------------------------
logic [BITS_ENTRIES-1:0] num_commit;                      // 當前 commit 數量
logic [BITS_ENTRIES-1:0] flush_number;                    // flush 數量
logic [BITS_ENTRIES-1:0] issue_pointer_n, issue_pointer_q;// 發派指標（目前 / 下一）
logic [BITS_ENTRIES-1:0] issue_pointer_plus;              // 發派指標+1
logic [BITS_ENTRIES-1:0] flush_branch_mispredict;         // 分支 mispredict 起點
logic [BITS_ENTRIES-1:0] flush_branch_mispredict_plus;    // 分支 mispredict+1

// ------------------------------------------------------------------------------------
// flush 資訊記錄（FU、trans id、LSU 地址）
// ------------------------------------------------------------------------------------
logic [NR_ENTRIES-1:0][3:0 ] flush_fu_n;
logic [NR_ENTRIES-1:0][3:0 ] flush_trans_id_n;
logic [NR_ENTRIES-1:0][63:0] flush_lsu_addr_n;
logic [NR_ENTRIES-1:0][BITS_ENTRIES-1:0] num_flush_branch;
logic [NR_ENTRIES-1:0][BITS_ENTRIES-1:0] num_flush_branch_n;

// ------------------------------------------------------------------------------------
// commit port 對應的 commit 指標與分支指令旗標
// ------------------------------------------------------------------------------------
logic [CVA6Cfg.NrCommitPorts-1:0] commit_branch_instr;
logic [CVA6Cfg.NrCommitPorts-1:0][BITS_ENTRIES-1:0] commit_pointer_n;
logic [CVA6Cfg.NrCommitPorts-1:0][BITS_ENTRIES-1:0] commit_pointer_q;
logic [CVA6Cfg.NrCommitPorts-1:0][BITS_ENTRIES-1:0] commit_pointer_plus;

// ------------------------------------------------------------------------------------
// 傳入的 source register index（提供給 issue_read_operands）
// ------------------------------------------------------------------------------------
logic [CVA6Cfg.NrissuePorts-1:0][ariane_pkg::REG_ADDR_SIZE-1:0] rs1;
logic [CVA6Cfg.NrissuePorts-1:0][ariane_pkg::REG_ADDR_SIZE-1:0] rs2;

// ------------------------------------------------------------------------------------
// 資訊 assign（ROB 滿、flush 處理、pointer 計算）
// ------------------------------------------------------------------------------------
assign sb_full_o                    = issue_full;                              // ROB 是否已滿輸出
assign issue_full                   = (issue_cnt_q[BITS_ENTRIES] == 1'b1);     // 超過上限就算 full
assign case_flush                   = (issue_pointer_q > commit_pointer_q[0]) ? 1'd0 : 1'd1; // flush 是否繞圈
assign enable_dual_issue            = (issue_cnt_q < 5'd13);                   // 少於 13 條可雙發派
assign issue_pointer_plus           = (issue_pointer_q==4'd15) ? 4'd0 : issue_pointer_q+4'd1; // 發派位置加一
assign commit_pointer_plus[0]       = (commit_pointer_q[0]==4'd15) ? 3'd0 : (commit_pointer_q[0]+4'd1); // commit 指標+1
assign commit_pointer_plus[1]       = (commit_pointer_q[1]==4'd15) ? 3'd0 : (commit_pointer_q[1]+4'd1);
assign flush_branch_mispredict      = trans_id_i[6];                           // 第六個 trans_id 做為 flush 起點
assign flush_branch_mispredict_plus = (trans_id_i[6]==4'd15) ? 3'd0 : (trans_id_i[6]+4'd1);

// ------------------------------------------------------------------------------------
// 暫存解碼指令
// ------------------------------------------------------------------------------------
ariane_pkg::scoreboard_entry_t [1:0] decoded_instr;  // 暫存 port 0 與 port 1 的指令

```
### 🔄 ROB Flush 邏輯與 RVFI 整合詳細解析

```systemverilog
// ------------------------------------------------------------------------------------------------
// 🧠 ROB Instruction Preparation and RVFI Hookup
// ------------------------------------------------------------------------------------------------
// 將 decode 階段傳來的 scoreboard_entry 儲存進 ROB buffer，若啟用 RVFI 模式會補充 rs1/rs2 的數值資料
// 以及將 LSU 欄位預設清空，確保後續 RVFI trace 資訊正確。
always_comb begin
    decoded_instr[0] = decoded_instr_i[0];
    decoded_instr[1] = decoded_instr_i[1];

    // 如果啟用 RVFI 模式，會將讀取資料與 LSU 資訊初始化
    if (IsRVFI) begin
        decoded_instr[0].rs1_rdata = rs1_forwarding_i[0];
        decoded_instr[0].rs2_rdata = rs2_forwarding_i[0];
        decoded_instr[0].lsu_addr  = '0;
        decoded_instr[0].lsu_rmask = '0;
        decoded_instr[0].lsu_wmask = '0;
        decoded_instr[0].lsu_wdata = '0;

        decoded_instr[1].rs1_rdata = rs1_forwarding_i[1];
        decoded_instr[1].rs2_rdata = rs2_forwarding_i[1];
        decoded_instr[1].lsu_addr  = '0;
        decoded_instr[1].lsu_rmask = '0;
        decoded_instr[1].lsu_wmask = '0;
        decoded_instr[1].lsu_wdata = '0;
    end
end

// ------------------------------------------------------------------------------------------------
// ✅ Commit Ports: 將要寫回的指令從 ROB 中讀出對應位置內容
// ------------------------------------------------------------------------------------------------
// 根據 commit_pointer_q 決定從 mem_q 中讀出哪條已完成的指令，
// 並補上 trans_id，輸出給 commit 單元
always_comb begin : commit_ports
    for (int unsigned i = 0; i < CVA6Cfg.NrCommitPorts; i++) begin
        commit_instr_o[i] = mem_q[commit_pointer_q[i]].sbe;
        commit_instr_o[i].trans_id = commit_pointer_q[i];
    end
end

// ------------------------------------------------------------------------------------------------
// 🔁 Flush Entry 決策邏輯（標示 flush 條目數量）
// ------------------------------------------------------------------------------------------------
// 判斷目前 commit 的條目是否包含被標記為 mispredict 的分支，若是則回傳應該 flush 幾條指令。
always_comb begin
    num_flush_q = 4'd0;

    if ((|commit_branch_instr) & (num_flush_flag[commit_pointer_q[0]] | num_flush_flag[commit_pointer_q[1]])) begin
        if (num_flush_flag[commit_pointer_q[0]])
            num_flush_q = num_flush_branch[commit_pointer_q[0]];

        if (num_flush_flag[commit_pointer_q[1]])
            num_flush_q = num_flush_branch[commit_pointer_q[1]];
    end
end

// ------------------------------------------------------------------------------------------------
// 🚩 Flush Flag 與 Flush 數量記錄更新邏輯
// ------------------------------------------------------------------------------------------------
// - 若 flush 發生，會記錄是哪一條指令引起，以及需要 flush 幾條。
// - 若該指令已被 commit，則清除該筆記錄。
always_comb begin
    num_flush_flag_n = num_flush_flag;
    num_flush_branch_n = num_flush_branch;

    if (flush_i) begin
        num_flush_flag_n = 16'd0;
        for (int unsigned i = 0; i < NR_ENTRIES; i++) begin
            num_flush_branch_n[i] = '0;
        end
    end else if (resolved_branch_i.is_mispredict & flush_instr) begin
        // 分支錯誤時設定 flush flag 與對應條數
        num_flush_flag_n[flush_branch_mispredict] = 1'd1;
        num_flush_branch_n[flush_branch_mispredict] = flush_number;
    end

    // 當分支指令寫回後，清除對應 flush flag 記錄
    if (|commit_branch_instr & !((commit_pointer_q[0] == flush_branch_mispredict) & resolved_branch_i.is_mispredict & flush_instr)) begin
        num_flush_flag_n[commit_pointer_q[0]] = 1'd0;
        num_flush_branch_n[commit_pointer_q[0]] = '0;
    end

    if (|commit_branch_instr & !((commit_pointer_q[1] == flush_branch_mispredict) & resolved_branch_i.is_mispredict & flush_instr)) begin
        num_flush_flag_n[commit_pointer_q[1]] = 1'd0;
        num_flush_branch_n[commit_pointer_q[1]] = '0;
    end
end

// ------------------------------------------------------------------------------------------------
// ⏺ Flush 記錄暫存於時脈邊緣
// ------------------------------------------------------------------------------------------------
// 在每個時脈邊緣更新 num_flush_flag 與 num_flush_branch，
// 保持這些 flush 資訊跨 cycle 穩定存在
always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
        num_flush_flag <= 16'd0;
        for (int unsigned i = 0; i < NR_ENTRIES; i++) begin
            num_flush_branch[i] <= '0;
        end
    end else begin
        num_flush_flag <= num_flush_flag_n;
        for (int unsigned i = 0; i < NR_ENTRIES; i++) begin
            num_flush_branch[i] <= num_flush_branch_n[i];
        end
    end
end

// ------------------------------------------------------------------------------------------------
// 📊 Flush Entry 數量計算：計算有幾個條目被標記為 flush
// ------------------------------------------------------------------------------------------------
// 統計 flush_entry_n 陣列中有幾個條目需要 flush，回傳 flush_number
always_comb begin
    flush_number = '0;
    for (int unsigned i = 0; i < NR_ENTRIES; i++) begin
        flush_number = flush_number + flush_entry_n[i];
    end
end
```

---

### 📘 說明摘要

這段程式碼實作的是 Reorder Buffer（ROB）的後半部功能：

1. **RVFI 資訊補全**：若開啟 RVFI 模式，會記錄指令執行前的 rs1/rs2 值，幫助外部工具驗證指令正確性。
2. **Commit 輸出邏輯**：將 ROB 中對應條目的指令輸出給 commit 階段。
3. **Flush 機制**：紀錄並追蹤每次 branch mispredict 所需 flush 的指令數量，並根據 commit 狀態清除該紀錄。
4. **時脈邊緣更新機制**：確保 flush flag 和 flush entry 數量跨週期正確保存。
5. **Flush 數量統計**：將所有需 flush 的條目加總供後續單元參考。

此邏輯確保當分支預測錯誤時，能夠正確識別哪些指令需要清除並避免非法寫回。

---

### 🚀 Issue 入口與 ROB 更新邏輯

```systemverilog
// ------------------------------------------------------------------------------------------------
// 🧠 Issue Path Preparation and Trans ID Allocation
// ------------------------------------------------------------------------------------------------
// 這段 always_comb 負責將 rename 階段送進來的指令準備好輸出給 issue_read_operands 模組，
// 並根據是否允許 dual issue 決定是否同時發派兩條指令。

always_comb begin
    rs1[0]                        = '0;
    rs1[1]                        = '0;
    rs2[0]                        = '0;
    rs2[1]                        = '0;

    issue_instr_o[0]              = '0;
    issue_instr_o[0].trans_id     = '0;
    issue_instr_valid_o[0]        = '0;
    decoded_instr_ack_o[0]        = '0;

    issue_instr_o[1]              = '0;
    issue_instr_o[1].trans_id     = '0;
    issue_instr_valid_o[1]        = '0;
    decoded_instr_ack_o[1]        = '0;

    if(enable_dual_issue) begin 
      // 若允許雙發派，準備兩條指令
      issue_instr_o[0]            = decoded_instr_i[0];
      issue_instr_o[0].rs1        = (rs1[0]!='0) ? rs1[0] : decoded_instr_i[0].rs1;
      issue_instr_o[0].rs2        = (rs2[0]!='0) ? rs2[0] : decoded_instr_i[0].rs2;
      issue_instr_valid_o[0]      = decoded_instr_valid_i[0] & ~unresolved_branch_i;
      decoded_instr_ack_o[0]      = issue_ack_i[0];
      issue_instr_o[0].trans_id   = issue_pointer_q;
          
      issue_instr_o[1]            = decoded_instr_i[1];
      issue_instr_o[1].rs1        = (rs1[1]!='0) ? rs1[1] : decoded_instr_i[1].rs1;
      issue_instr_o[1].rs2        = (rs2[1]!='0) ? rs2[1] : decoded_instr_i[1].rs2;
      issue_instr_valid_o[1]      = decoded_instr_valid_i[1] & ~unresolved_branch_i;
      decoded_instr_ack_o[1]      = issue_ack_i[1];
      issue_instr_o[1].trans_id   = issue_pointer_q + 1;
    end else begin 
      // 單發派邏輯
      issue_instr_o[0]            = decoded_instr_i[0];
      issue_instr_o[0].rs1        = (rs1[0]!='0) ? rs1[0] : decoded_instr_i[0].rs1;
      issue_instr_o[0].rs2        = (rs2[0]!='0) ? rs2[0] : decoded_instr_i[0].rs2;
      issue_instr_o[0].trans_id   = issue_pointer_q;
      issue_instr_valid_o[0]      = decoded_instr_valid_i[0] & ~unresolved_branch_i & ~issue_full;
      decoded_instr_ack_o[0]      = issue_ack_i[0] & ~issue_full;
    end
end

// ------------------------------------------------------------------------------------------------
// 🧾 Reorder Buffer 記憶體更新邏輯
// ------------------------------------------------------------------------------------------------
// 根據是否成功發派 instruction，將其存入 mem_n（下一個 ROB buffer 狀態）
// 若啟用 FPU 功能並判定為 FPR 寫入，會標示 is_rd_fpr_flag。

always_comb begin
    mem_n                   = mem_q;  // 預設保留現況
    issue_en_0              = 1'b0;
    issue_en_1              = 1'b0;
    commit_branch_instr     = 2'd0;
    flush_instr             = 1'd0;
    flush_entry_n           = 16'd0;

    for (int unsigned i = 0; i < NR_ENTRIES; i++) begin
      flush_fu_n      [i] = '0;
      flush_trans_id_n[i] = '0;
      flush_lsu_addr_n[i] = '0;
    end

    // ------------------------------------------------------------------------------------
    // Issue Port
    // ------------------------------------------------------------------------------------
    if (((decoded_instr_valid_i[0] && decoded_instr_ack_o[0]) & (decoded_instr_valid_i[1] && decoded_instr_ack_o[1])) & !flush_unissued_instr_i) begin
      issue_en_0 = 1'b1;
      issue_en_1 = 1'b1;
      
      mem_n[issue_pointer_q] = {
        1'b1, 
        (CVA6Cfg.FpPresent && ariane_pkg::is_rd_fpr(
          decoded_instr_i[0].op
        )),  
        decoded_instr[0]  
      };
      mem_n[issue_pointer_plus] = {
        1'b1,
        (CVA6Cfg.FpPresent && ariane_pkg::is_rd_fpr(
          decoded_instr_i[1].op
        )),  
        decoded_instr[1] 
      };
    end else if ((decoded_instr_valid_i[0] && decoded_instr_ack_o[0]) & !flush_unissued_instr_i) begin
      issue_en_0 = 1'b1;

      mem_n[issue_pointer_q] = {
        1'b1, 
        (CVA6Cfg.FpPresent && ariane_pkg::is_rd_fpr(
          decoded_instr_i[0].op
        )),
        decoded_instr[0]
      };
    end
end
```

---

### 📘 說明摘要

這段程式碼負責將 decode 階段送入的指令準備轉發至 Issue 單元，並同步更新 ROB（Reorder Buffer）：

1. **Issue 發派邏輯**：
   - 若支援 Dual Issue，並且兩條指令都有效且可接受，則同時發派兩條。
   - 否則只發派第一條指令。
   - 同時會對 rs1/rs2 處理 forwarding，避免依賴錯誤值。

2. **trans_id 記錄**：每條指令會記錄在 ROB 中的索引作為其 transaction ID。

3. **ROB Buffer 寫入**：
   - 寫入時標記 `issued=1`，並根據是否為浮點寫入，設定 `is_rd_fpr_flag`。
   - 寫入目的地址為 `issue_pointer_q` 或其遞增版。

這些機制確保後續執行單元可以正確讀取對應的寄存器值與指令內容，也為 commit 與 flush 提供基礎依據。

---

### 🔁 ROB 寫回與 Commit 更新邏輯

```systemverilog
// ------------------------------------------------------------------------------------------------
// ✅ FU 為 NONE 的指令視為立即有效
// ------------------------------------------------------------------------------------------------
// 如果某條指令的功能單元 (FU) 是 NONE，代表它不需要等待資源即可執行，
// 此時直接將該指令標記為 valid。
for (int unsigned i = 0; i < NR_ENTRIES; i++) begin
  if (mem_q[i].sbe.fu == ariane_pkg::NONE && mem_q[i].issued)
    mem_n[i].sbe.valid = 1'b1;
end

// ------------------------------------------------------------------------------------------------
// 🧾 Write Back（含 RVFI Trace 記錄）
// ------------------------------------------------------------------------------------------------
// 如果啟用 RVFI 模式，當 LSU 有讀/寫行為，記錄其地址與掩碼（mask）資訊
if (IsRVFI) begin
  if (lsu_rmask_i != 0) begin
    mem_n[lsu_addr_trans_id_i].sbe.lsu_addr  = lsu_addr_i;
    mem_n[lsu_addr_trans_id_i].sbe.lsu_rmask = lsu_rmask_i;
  end else if (lsu_wmask_i != 0) begin
    mem_n[lsu_addr_trans_id_i].sbe.lsu_addr  = lsu_addr_i;
    mem_n[lsu_addr_trans_id_i].sbe.lsu_wmask = lsu_wmask_i;
    mem_n[lsu_addr_trans_id_i].sbe.lsu_wdata = wbdata_i[1];
  end
end

// 根據 write-back 資訊更新對應指令狀態（來自執行單元寫回）
for (int unsigned i = 0; i < CVA6Cfg.NrWbPorts; i++) begin
  if (wt_valid_i[i] && mem_q[trans_id_i[i]].issued) begin
    mem_n[trans_id_i[i]].sbe.valid  = 1'b1;
    mem_n[trans_id_i[i]].sbe.result = wbdata_i[i];

    // 若有啟用 Debug 功能則記錄預測地址
    if (CVA6Cfg.DebugEn) begin
      mem_n[trans_id_i[i]].sbe.bp.predict_address = resolved_branch_i.target_address;
    end

    // CVXIF 特殊處理，若 x_we_i 為 0 表示該指令無 rd，強制設為 0
    if (mem_n[trans_id_i[i]].sbe.fu == ariane_pkg::CVXIF && ~x_we_i) begin
      mem_n[trans_id_i[i]].sbe.rd = 5'b0;
    end

    // 儲存 exception 資訊
    if (ex_i[i].valid) begin
      mem_n[trans_id_i[i]].sbe.ex = ex_i[i];
    end else if (CVA6Cfg.FpPresent &&
                 mem_q[trans_id_i[i]].sbe.fu inside {ariane_pkg::FPU, ariane_pkg::FPU_VEC}) begin
      // FPU 例外額外處理
      mem_n[trans_id_i[i]].sbe.ex.cause = ex_i[i].cause;
    end
  end
end

// ------------------------------------------------------------------------------------------------
// ✅ Commit Port：收到 commit_ack 後釋放該條目
// ------------------------------------------------------------------------------------------------
// 將 ROB 條目釋放，並標記為已完成；若是分支類型的指令，設為 commit_branch_instr 以供 flush 使用
for (int i = 0; i < CVA6Cfg.NrCommitPorts; i++) begin
  if (commit_ack_i[i]) begin
    mem_n[commit_pointer_q[i]].issued    = 1'b0;
    mem_n[commit_pointer_q[i]].sbe.valid = 1'b0;

    // 若為分支指令（FU == BRANCH = 0b0100）則標記
    if ((mem_n[commit_pointer_q[i]].sbe.fu == 4'b0100)) begin
      commit_branch_instr = 1'b1;
    end
  end
end
```

---

### 📘 說明摘要

這段程式碼處理了三個 Reorder Buffer 相關的更新動作：

1. **立即有效指令判斷**：若功能單元為 NONE，代表不需資源競爭，可直接設為 valid。
2. **Write-back 操作**：接收執行單元的結果（包含資料與 exception 狀態），並更新對應條目的資訊。
   - RVFI 模式下會補 LSU trace 資訊（讀/寫地址與遮罩）。
   - CVXIF 指令若無寫入會清除 rd 欄位。
3. **Commit 動作處理**：當指令成功寫回時，從 ROB 清除該條目，若為分支指令則記錄其資訊以供 flush 使用。

這部分確保 ROB 條目的更新與釋放符合 CPU 動態指令執行模型，對維持指令準確性和支援分支回溯至關重要。
