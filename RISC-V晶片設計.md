---
title: RISC-V晶片設計

---


# 計算機晶片設計筆記整理







## risc-v 晶片設計
本篇主要是透過給定的ISA(RV32IM)和RISC-V架構，並利用verilog設計出處理器架構，以下是此處理器的架構:
### **1. 基礎架構:**

* **指令集架構 (ISA):** RISC-V RV32IM
* **RV32I:** 32 位元基礎整數指令集 (Base Integer Instruction Set)。
* **M Extension:** 標準整數乘除法擴充 (Integer Multiplication and Division)。
* **資料路徑寬度 (Data Path Width):** 32-bit。
* **執行模式 (Execution Model):** 單核心 (Single Core)、單執行緒 (Single Thread)、單發射 (Single Issue) - 每個時脈週期執行一條指令。


---


### **2. 微架構設計:**
> Q: 微架構是啥?
> A: Microarchitecture指的是電腦處理器在硬體上的具體實現設計方式。

**3 級流水線**: 
* IF (Instruction Fetch): 取指、PC 更新。
* ID (Instruction Decode): 譯碼、讀取暫存器、分支判斷 (Branch Comp)、資料轉發偵測。
* EX (Execute / Write Back): 運算 (ALU/MUL/DIV)、記憶體存取 (LSU)、寫回結果。

**分支預測:**
* 靜態預測 (Static): 預設順序執行 (PC+4)。
* 分支解析 (Resolution): 在 ID 階段 透過 BRANCH COMP 判斷。
* 分支懲罰 (Penalty): 若發生跳躍 (Branch Taken)，需沖刷 (Flush) IF/ID 流水線暫存器，損失 1 個週期。


---


### **3. 記憶體系統:**

* 定址空間 (Addressing Space): 32-bit 地址線，支援最大 4GB 定址範圍。
* 架構類型: Harvard Architecture -指令與資料分開存取。
* 指令記憶體 (Instruction Memory): 唯讀 (ROM)，容量規格支援 4GB。
* 資料記憶體 (Data Memory): 讀寫 (RAM)，容量規格支援 4GB。
* 存取對齊: 支援 Byte, Half-word, Word 存取。


---


### **4. 暫存器配置:**
* 通用暫存器 (GPRs): 32 個 32-bit 暫存器 (x0 ~ x31)且x0硬體固定為 0。
> Q:什麼是通用暫存器(GPR)?
> 
> A:處理器最核心的高速數據存儲區，負責存放正在處理中的數據，而不是長期存放資料。通常用來存運算的運算元與結果、回傳地址、比較條件...。

* 程式計數器 (PC): 32-bit。


---

### **5. 執行單元:**
* ALU (算術邏輯單元): 負責加減 (ADD/SUB)、邏輯 (AND/OR/XOR)、移位 (SLL/SRL/SRA)、比較 (SLT)。
* MUL (乘法單元): 支援 MUL,MULH,MULHU,MULHSU。
* DIV (除法單元): 支援 DIV, DIVU, REM, REMU (除法與取餘數)。
* LSU (載入儲存單元): 負責 Load / Store 指令的地址計算與記憶體介面控制。



---

### **6. 危障處理:**
* Forwarding Unit 解決 Data Hazard: 
支援 EX-to-ID Forwarding：當前指令在 ID 階段即可取得上一條指令在 EX 階段剛計算出的結果，無需暫停流水線。

* 管線沖刷解決Control Hazard:
當分支預測錯誤時，透過 Flush 機制清除管線中的錯誤指令。

(目前版本還無法解決load-use hazard)

---

## **開始實作!!!**

架構圖如下(簡報裡面有畫質較好的):

![image](https://hackmd.io/_uploads/HJDu3_oBWx.png)

包含了許多基本元件，待會程式講解會有細部拆解的圖片。
程式部分主要是由一個rv32im_core.v主程式進行所有零組件的封裝、和許多個圖中的元件的子程式所組成。

也就是說(對應架構圖拆分):
1. pc_reg.v (程式計數器)
2. program_rom.v (指令記憶體)

---

3. stage_if_id.v (IF/ID 流水線暫存器)

---

4. inst_dec.v (指令解碼器)
5. register_file.v (通用暫存器堆)
6. controller.v (含分支比較器)
7. forwarding_unit.v (轉發單元)

---


8. stage_id_ex.v (ID/EX 流水線暫存器)

---

9. alu.v (算術邏輯單元)
10. mul_unit.v (乘法單元)
11. div_unit.v (除法單元)
12. lsu.v (資料記憶體單元)


---

以上12個小元件程式全部封裝進rv32im_core.v這個主程式裡面。
以下會逐一解說，每個元件的功能、那些輸入輸出參數代表什麼、輸入從哪裡來、輸出到哪裡去、程式碼細部講解。

## 1. pc_reg.v (程式計數器)
主要功能:
實作program counter 來更新要取的指令的位置。如果要JUMP，則JUMP運算後的結果也會存放在pc next，否則正常情況下就是pc=pc+4，但是不論以上是兩種情況的哪一種，計算PC位置都不是program counter的工作(在這個處理器的程式中，我們把branch運算放在rv32im_core.v這個主程式)，program counter只會根據clk或rst用來"更新"或"reset" pc_out裡面所記錄的位置，接著ROM才會去讀取pc_out，並取出正確的指令。

輸入訊號: 
clk:時脈訊號
rst:重置訊號
pc_next:下一個想要讀取的指令位置。

輸出訊號:
pc_out:pc更新後的指令位置。

程式碼:
![image](https://hackmd.io/_uploads/HJP_e3srWe.png)

程式碼解說:
在sensitive list中當"時脈正緣或rst正緣出發時"更新pc_out的值。

## 2. program_rom.v (指令記憶體)
主要功能:實作read only memory 來存放指令，接收剛剛從PC出來的訊號(pc_out)來做為這邊的address，透過address可以知道要從ROM中取出哪一條指令。

輸入訊號:
addr(剛剛的pc_out)

輸出訊號:
inst (從ROM取出來的指令)

程式碼:
![image](https://hackmd.io/_uploads/Byr9l3orWe.png)

程式碼解說:
為甚麼addr要取[11:2]?
因為10bits剛好是1KB(ROM可以存放1KB條指令)，又每一條指令address都一定是4的倍數，後面兩個位數一定是00，接下來我們要把address換成index所以要除以4，也就是右移兩個位置。所以最後就是大概就是[0:9]變成[2:11]接著把[2:11]作為ROM的index。
在initial 區塊可以編寫自己的程式放入rom陣列去做讀取。

## 3. stage_if_id.v (IF/ID 流水線暫存器)
主要功能:
實作IF/ID管線(暫存器)，這個暫存器紀錄PC address和指令，如此一來他可以允許指令先偷跑，在前一條指令還沒做完之前，就先fetch下一條指令，並儲存在管線中，等到下一個CLK來的時候再進入ID階段。如此一來可以讓指令重疊進行加速。
IF/ID 的管線沖刷:
![image](https://hackmd.io/_uploads/SyyG5igI-l.png)
在reset或是flush_IFID 訊號為1時就會沖刷管線。

> 什麼時候會 flush_IFID ?
> 執行JUMP指令時(Branch Taken分支成立時會造成control hazard，因此需要flush IFID管線來解決。)

輸入訊號:


輸出訊號:


程式碼:
![image](https://hackmd.io/_uploads/SkZhehoHWg.png)

程式碼解說:

## 4. inst_dec.v (指令解碼器)
主要功能:
![image](https://hackmd.io/_uploads/HkmxwctIZx.png)

![image](https://hackmd.io/_uploads/HkPUoMZ8Zl.png)

這個程式主要是把送進來的inst拆解成不同的片段(MicroInstruction)，來做為下一階段controller和其他地方做使用(拆解出來的分段不一定都會用到)。

接著，以下介紹不同Microinstruction在此架構中的意義:
    opcode:
        指令分類標籤
    addr_rd:
        目標暫存器位址，指定運算完後的結果最終要寫回 REG_FILE 中的哪一個暫存器。
    funct3:
        在相同的opcode下，用來區分具體動作。例如在Branch下區分是BEQ還是BNE。
    addr_rs1:
        告訴 REG_FILE 要取出哪一個暫存器的數值作為第一個運算元。
    addr_rs2:
        REG_FILE 要取出哪一個暫存器的數值作為第二個運算元。在 S-Type 中，這也代表要存入記憶體的資料。
    funct7:
        通常與 funct3 配合使用，用於區分相似運算。例如區分算術右移 (SRA) 與邏輯右移 (SRL)，或是識別 RV32M 的乘除法指令。

輸入訊號:
輸出訊號:
程式碼:
```
module inst_dec (
    input  wire [31:0] inst,
    output wire [6:0]  opcode, funct7,
    output wire [4:0]  addr_rd, addr_rs1, addr_rs2,
    output wire [2:0]  funct3,
    output reg  [31:0] imm
);
    assign opcode   = inst[6:0];
    assign addr_rd  = inst[11:7];
    assign funct3   = inst[14:12];
    assign addr_rs1 = inst[19:15];
    assign addr_rs2 = inst[24:20];
    assign funct7   = inst[31:25];

    always @(*) begin
        case (opcode)
            7'b0010011, 7'b0000011, 7'b1100111: // I-type
                imm = {{20{inst[31]}}, inst[31:20]};
            7'b0100011: // S-type
                imm = {{20{inst[31]}}, inst[31:25], inst[11:7]};
            7'b1100011: // B-type
                imm = {{19{inst[31]}}, inst[31], inst[7], inst[30:25], inst[11:8], 1'b0};
            7'b1101111: // J-type
                imm = {{11{inst[31]}}, inst[31], inst[19:12], inst[20], inst[30:21], 1'b0};
            7'b0110111, 7'b0010111: // U-type
                imm = {inst[31:12], 12'b0};
            default: imm = 32'b0;
        endcase
    end
endmodule
```
程式碼解說:首先拆解指令片段指派給不同Microinstruction給後續元件使用，接著，根據不同指令把立即值imm取出並把取出的立即值imm擴充成32bits。

## 5. register_file.v (通用暫存器堆)

主要功能:**根據位置取出該處的值** 或是 **把進入的值寫入目標暫存器**。

輸入訊號:

clk: 時脈
addr_rs1, addr_rs2, addr_rd: 兩個來源暫存器，一個目標暫存器。
write_data: 欲寫入的資料。
write_en: 寫入的enable。

輸出訊號:

rs1_value,rs2_value :兩個來源暫存器內所存放的實際值。

程式碼:
![image](https://hackmd.io/_uploads/BJVfZhsSZg.png)

程式碼解說:
宣告一個gpr[32][32] (大小1KB)，並在initial區塊進行初始化。
若記憶體位置不是x0則寫入或讀取。

## 6. controller.v (含分支比較器)
主要功能:

根據不同類型的指令種類或是funct7、funct3來決定寫入的enable、alu運算元的選擇、alu要進行哪種運算、alu運算結果的選擇、PC的選擇線(branch or PC+4)、是否要沖刷管線。

輸入訊號:

由decoder送進來的各種MicroInstruction...
opcode、funct7、funct3

輸出訊號:
程式碼:
```
module controller (
    input  wire [6:0] opcode, funct7,
    input  wire [2:0] funct3,
    input  wire [31:0] rs1_value, rs2_value,
    output reg  write_regf_en_r,
    output reg  [1:0] sel_alu_a_r, sel_alu_b_r,
    output reg  [3:0] alu_op_r,
    output reg  [1:0] sel_rd_value_r,
    output reg  write_read_r, sel_pc_r, flush_IFID_r, flush_IDEX_r
);
    reg branch_taken;
    // Branch Comparator
    always @(*) begin
        case (funct3)
            3'b000: branch_taken = (rs1_value == rs2_value); // BEQ
            3'b001: branch_taken = (rs1_value != rs2_value); // BNE
            3'b100: branch_taken = ($signed(rs1_value) < $signed(rs2_value)); // BLT
            3'b101: branch_taken = ($signed(rs1_value) >= $signed(rs2_value)); // BGE
            3'b110: branch_taken = (rs1_value < rs2_value); // BLTU
            3'b111: branch_taken = (rs1_value >= rs2_value); // BGEU
            default: branch_taken = 0;
        endcase
    end

    // Main Control Signals
    always @(*) begin
        // Defaults
        write_regf_en_r = 0; sel_alu_a_r = 2'b01; sel_alu_b_r = 2'b00;
        alu_op_r = 0; sel_rd_value_r = 0; write_read_r = 0;
        sel_pc_r = 0; flush_IFID_r = 0; flush_IDEX_r = 0;

        case (opcode)
            7'b0110011: begin // R-type
                write_regf_en_r = 1;
                if (funct7 == 1) sel_rd_value_r = (funct3[2]) ? 2'b11 : 2'b01; // M-ext
                else alu_op_r = {funct7[5], funct3};
            end
            7'b0010011: begin // I-type
                write_regf_en_r = 1; sel_alu_b_r = 2'b01; alu_op_r = {1'b0, funct3};
            end
            7'b0000011: begin // Load
                write_regf_en_r = 1; sel_alu_b_r = 2'b01; sel_rd_value_r = 2'b10;
            end
            7'b0100011: begin // Store
                sel_alu_b_r = 2'b01; write_read_r = 1;
            end
            7'b1100011: begin // Branch
                if (branch_taken) begin sel_pc_r = 1; flush_IFID_r = 1; end
            end
            7'b1101111: begin // JAL
                write_regf_en_r = 1; sel_alu_b_r = 2'b10; sel_pc_r = 1; flush_IFID_r = 1;
            end
        endcase
    end
endmodule
```
程式碼解說:



| 訊號名稱 | 類型 | 1 的時候代表什麼？ |
| -------- | -------- | -------- |
| write_regf_en_r     | 寫回|"運算結果要存回暫存器 (如 ADD, LW)"|
| sel_alu_a_r     | ALU來源|"0: PC, 1: RS1"|
| sel_alu_b_r     |ALU來源|"0: RS2, 1: 立即值, 2: 常數4"|
| alu_op_r     |ALU功能|具體的運算代碼 (加、減、移位...)|
| sel_rd_value_r     |寫回來源|"0: ALU, 1: MUL, 2: Mem, 3: DIV"|
| write_read_r     |記憶體|要寫入記憶體 (SW 指令)|
| sel_pc_r     | 跳躍|要跳躍！PC 不加 4 了，改去別的地方|
| flush_IFID_r     |管線|殺掉 IF 階段剛抓到的指令|
| flush_IDEX_r     |管線|殺掉 ID 階段的指令|

## 7. forwarding_unit.v (轉發單元)
主要功能:比較前一條指令的目的暫存器跟現在這條指令的來源暫存器，解決RAW hazard，確保後面那條指令是取到正確的值。

輸入訊號:
addr_rs1:現在這條指令的來源暫存器1位址(ID階段)
addr_rs2:現在這條指令的來源暫存器2位址(ID階段)
addr_rd_ex:上一條指令的目的暫存器位址(EXE階段，正準備寫回)


輸出訊號:
sel_rs1_forward:確定將上一條指令EXE完的計算結果前饋現在這條指令，代替現在來源暫存器1的值做使用。

sel_rs2_forward:確定將上一條指令EXE完的計算結果前饋現在這條指令，代替現在來源暫存器2的值做使用。

在實作的架構中sel_rs1_forward、sel_rs2_forward是reg file 輸出的選擇線。

程式碼:
![image](https://hackmd.io/_uploads/BkAobhsSbl.png)

## 8. stage_id_ex.v (ID/EX 流水線暫存器)
主要功能:
負責在時脈正緣，將解碼後的數值與控制訊號同步傳遞給 ALU；並支援沖刷（Flush）功能，能在遇到data hazard時清空狀態、插入nop。除此之外還能讓現在這條指令讀取前一條指令的目的暫存器，方便forwardind unit做使用。

程式碼:
```
module stage_id_ex (
    input  wire clk, rst, flush_IDEX_in,
    input  wire [31:0] pc_in, rs1_value_in, rs2_value_in, imm_in,
    input  wire [4:0]  addr_rd_in,
    input  wire [2:0]  funct3_in,
    input  wire write_regf_en_in, write_read_in, sel_alu_a_in, 
    input  wire [1:0]  sel_alu_b_in, sel_rd_value_in,
    input  wire [3:0]  alu_op_in,
    
    output reg [31:0] pc_rr, rs1_value_r, rs2_value_r, imm_r,
    output reg [4:0]  addr_rd_r,
    output reg [2:0]  funct3_r,
    output reg write_regf_en_r, write_read_r, sel_alu_a_r,
    output reg [1:0]  sel_alu_b_r, sel_rd_value_r,
    output reg [3:0]  op_r
);
    always @(posedge clk or posedge rst) begin
        if (rst || flush_IDEX_in) begin
             pc_rr <= 0; write_regf_en_r <= 0; write_read_r <= 0;
             addr_rd_r <= 0; op_r <= 0; 
        end else begin
             pc_rr <= pc_in; 
             rs1_value_r <= rs1_value_in; 
             rs2_value_r <= rs2_value_in;
             imm_r <= imm_in; 
             addr_rd_r <= addr_rd_in; 
             funct3_r <= funct3_in;
             write_regf_en_r <= write_regf_en_in; 
             write_read_r <= write_read_in;
             sel_alu_a_r <= sel_alu_a_in; 
             sel_alu_b_r <= sel_alu_b_in;
             sel_rd_value_r <= sel_rd_value_in;
             op_r <= alu_op_in;
        end
    end
endmodule
```

## 9. alu.v (算術邏輯單元)
主要功能:
根據opcode進行不同運算
程式碼:
![image](https://hackmd.io/_uploads/S1sIM3iSWx.png)

## 10. mul_unit.v (乘法單元)
主要功能:
根據function code進行不同乘法運算，在這邊用簡單乘法符號代表乘法器實作(雖然比較慢)
程式碼:
![image](https://hackmd.io/_uploads/Skf9GhoHbl.png)

程式碼解說:
## 11. div_unit.v (除法單元)
主要功能:
根據function code進行不同除法運算，在這邊用簡單除法符號代表除法器實作(雖然比較慢)
程式碼:
![image](https://hackmd.io/_uploads/r1Eif3sBZe.png)


12. lsu.v (資料記憶體單元)
主要功能:根據function code實作LW跟SW指令，並提供記憶體空間暫存。
輸入訊號:
addr:base address
write_data:欲寫入的資料
write_en:是否能寫入ram的開關，確保讀跟寫兩個動作是互斥的。

輸出訊號:
read data:執行load word完指令所讀出來的資料。
程式碼:
![image](https://hackmd.io/_uploads/HJF6G2iHWg.png)

程式碼解說:

13. rv32im_core.v (top module)
主要功能:rv32im的top module，素則串接各種元件和子模型，

程式碼:
```

module rv32im_core (
    input wire clk,
    input wire rst
);
    // =========================================================================
    // 1. 內部訊號線宣告 (Wires)
    // =========================================================================

    // IF 階段訊號
    wire [31:0] pc, pc_next, pc_plus4, jump_addr;
    wire [31:0] inst;
    
    // IF/ID 輸出訊號
    wire [31:0] pc_id, inst_id;

    // ID 階段訊號
    wire [6:0]  opcode, funct7;
    wire [2:0]  funct3;
    wire [4:0]  addr_rd, addr_rs1, addr_rs2;
    wire [31:0] imm;
    wire [31:0] rs1_data_raw, rs2_data_raw; // 從 RegFile 讀出的原始值
    wire [31:0] rs1_value, rs2_value;       // 經過 Forwarding 選擇後的值

    // 控制器訊號 (ID 階段)
    wire        write_regf_en;
    wire [1:0]  sel_alu_a;    // 0:PC, 1:RS1
    wire [1:0]  sel_alu_b;    // 0:RS2, 1:IMM, 2:4
    wire [3:0]  alu_op;
    wire [1:0]  sel_rd_value; // 0:ALU, 1:MUL, 2:LSU, 3:DIV
    wire        write_read_lsu;
    wire        sel_pc;       // 1: Jump/Branch, 0: PC+4
    wire        flush_IFID, flush_IDEX;
    
    // Forwarding 訊號
    wire        sel_rs1_fwd, sel_rs2_fwd;

    // EX 階段訊號 (ID/EX 輸出)
    wire [31:0] pc_ex;
    wire [31:0] rs1_val_ex, rs2_val_ex, imm_ex;
    wire [4:0]  addr_rd_ex;
    wire [2:0]  funct3_ex;
    
    // EX 控制訊號
    wire        write_regf_en_ex, write_read_lsu_ex;
    wire [1:0]  sel_alu_a_ex, sel_alu_b_ex;
    wire [3:0]  alu_op_ex;
    wire [1:0]  sel_rd_value_ex;
    
    // 運算結果訊號
    wire [31:0] alu_in_a, alu_in_b;
    reg  [31:0] alu_in_b_mux; // 暫存變數用
    wire [31:0] alu_out, mul_out, div_out, lsu_read_data;
    
    // 最終寫回值 (Write Back)
    reg  [31:0] rd_value_wb;

    // =========================================================================
    // 2. 元件組裝 (Instantiation)
    // =========================================================================

    // --- IF Stage (取指) ---
    assign pc_plus4 = pc + 32'd4;
    assign pc_next  = (sel_pc) ? jump_addr : pc_plus4;

    pc_reg PC_UNIT (
        .clk(clk), .rst(rst), .pc_next(pc_next), .pc_out(pc)
    );

    program_rom ROM_UNIT (
        .addr(pc), .inst(inst)
    );

    stage_if_id IF_ID_REG (
        .clk(clk), .rst(rst || flush_IFID), // 發生跳躍時需沖刷
        .pc_in(pc), .inst_in(inst),
        .pc_out(pc_id), .inst_out(inst_id)
    );

    // --- ID Stage (譯碼) ---
    inst_dec DEC_UNIT (
        .inst(inst_id),
        .opcode(opcode), .funct7(funct7), .funct3(funct3),
        .addr_rd(addr_rd), .addr_rs1(addr_rs1), .addr_rs2(addr_rs2),
        .imm(imm)
    );

    register_file REG_FILE_UNIT (
        .clk(clk), 
        .addr_rs1(addr_rs1), .addr_rs2(addr_rs2),
        .addr_rd(addr_rd_ex),      // 寫回地址 (EX階段)
        .write_data(rd_value_wb),  // 寫回資料 (EX階段)
        .write_en(write_regf_en_ex), 
        .rs1_value(rs1_data_raw), .rs2_value(rs2_data_raw)
    );

    // Forwarding Mux (資料轉發選擇)
    assign rs1_value = (sel_rs1_fwd) ? rd_value_wb : rs1_data_raw;
    assign rs2_value = (sel_rs2_fwd) ? rd_value_wb : rs2_data_raw;

    forwarding_unit FWD_UNIT (
        .addr_rs1(addr_rs1), .addr_rs2(addr_rs2),
        .addr_rd_ex(addr_rd_ex), .reg_write_ex(write_regf_en_ex),
        .sel_rs1_forward(sel_rs1_fwd), .sel_rs2_forward(sel_rs2_fwd)
    );

    controller CTRL_UNIT (
        .opcode(opcode), .funct3(funct3), .funct7(funct7),
        .rs1_value(rs1_value), .rs2_value(rs2_value),
        .write_regf_en_r(write_regf_en),
        .sel_alu_a_r(sel_alu_a), .sel_alu_b_r(sel_alu_b),
        .alu_op_r(alu_op), .sel_rd_value_r(sel_rd_value),
        .write_read_r(write_read_lsu),
        .sel_pc_r(sel_pc), .flush_IFID_r(flush_IFID), .flush_IDEX_r(flush_IDEX)
    );

    assign jump_addr = pc_id + imm; // 計算跳躍目標

    stage_id_ex ID_EX_REG (
        .clk(clk), .rst(rst), .flush_IDEX_in(flush_IDEX),
        .pc_in(pc_id), .rs1_value_in(rs1_value), .rs2_value_in(rs2_value),
        .imm_in(imm), .addr_rd_in(addr_rd), .funct3_in(funct3),
        .write_regf_en_in(write_regf_en), .write_read_in(write_read_lsu),
        .sel_alu_a_in(sel_alu_a), .sel_alu_b_in(sel_alu_b),
        .alu_op_in(alu_op), .sel_rd_value_in(sel_rd_value),
        
        .pc_rr(pc_ex), .rs1_value_r(rs1_val_ex), .rs2_value_r(rs2_val_ex),
        .imm_r(imm_ex), .addr_rd_r(addr_rd_ex), .funct3_r(funct3_ex),
        .write_regf_en_r(write_regf_en_ex), .write_read_r(write_read_lsu_ex),
        .sel_alu_a_r(sel_alu_a_ex), .sel_alu_b_r(sel_alu_b_ex),
        .op_r(alu_op_ex), .sel_rd_value_r(sel_rd_value_ex)
    );

    // --- EX Stage (執行) ---
    // ALU Input Multiplexers
    assign alu_in_a = (sel_alu_a_ex == 2'b00) ? pc_ex : rs1_val_ex; // 0:PC, 1:RS1

    always @(*) begin
        case (sel_alu_b_ex)
            2'b00: alu_in_b_mux = rs2_val_ex;
            2'b01: alu_in_b_mux = imm_ex;
            2'b10: alu_in_b_mux = 32'd4; // For JAL (PC+4)
            default: alu_in_b_mux = 32'b0;
        endcase
    end
    assign alu_in_b = alu_in_b_mux;

    alu ALU_UNIT (
        .a(alu_in_a), .b(alu_in_b), .alu_op(alu_op_ex), .alu_out(alu_out)
    );

    mul_unit MUL_UNIT (
        .a(rs1_val_ex), .b(rs2_val_ex), .funct3(funct3_ex), .mul_out(mul_out)
    );

    div_unit DIV_UNIT (
        .a(rs1_val_ex), .b(rs2_val_ex), .funct3(funct3_ex), .div_out(div_out)
    );

    lsu LSU_UNIT (
        .clk(clk), .addr(alu_out), .write_data(rs2_val_ex),
        .write_en(write_read_lsu_ex), .funct3(funct3_ex), .read_data(lsu_read_data)
    );

    // 最終寫回 Mux
    always @(*) begin
        case (sel_rd_value_ex)
            2'b00: rd_value_wb = alu_out;
            2'b01: rd_value_wb = mul_out;
            2'b10: rd_value_wb = lsu_read_data;
            2'b11: rd_value_wb = div_out;
            default: rd_value_wb = 32'b0;
        endcase
    end

endmodule

```


---
部分放大:

![image](https://hackmd.io/_uploads/SyPWrtoH-g.png)

![image](https://hackmd.io/_uploads/SJmNSKiHZe.png)

![image](https://hackmd.io/_uploads/H1RvHYor-x.png)

![image](https://hackmd.io/_uploads/BJkAVKoS-x.png)

