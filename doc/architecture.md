# Architecture Document: RISC-V RV32I Pipelined Processor

## 1. Module Hierarchy
rv32i_top
├── if_stage
│   ├── program_counter
│   └── pc_mux
├── if_id_reg
├── id_stage
│   ├── register_file
│   ├── immediate_generator
│   └── control_unit
├── id_ex_reg
├── ex_stage
│   ├── alu
│   ├── branch_unit
│   └── forwarding_mux (x2)
├── forwarding_unit
├── hazard_detection_unit
├── ex_mem_reg
├── mem_stage
│   └── data_memory_interface
├── mem_wb_reg
└── wb_stage
└── wb_mux
