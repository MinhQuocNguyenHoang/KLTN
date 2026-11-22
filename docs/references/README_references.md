# Danh mục tài liệu tham khảo chuẩn (dùng thống nhất cho mọi báo cáo)

Mọi file nằm trong `references/`. Số trang dùng khi trích dẫn = **số trang PDF** (tra bằng `tools/find_ref.ps1`,
trùng với số trang in ở footer `x/N` của ST/TI/Microchip).

| Key LaTeX | File | Chuỗi trích dẫn chuẩn (IEEE) |
|---|---|---|
| `rm0440` | rm0440-stm32g4-series-advanced-armbased-32bit-mcus-stmicroelectronics.pdf | STMicroelectronics, *RM0440 Reference manual: STM32G4 Series advanced Arm®-based 32-bit MCUs*, Rev. 9, Mar. 2025. |
| `ds12288` | DS_stm32g474ve.pdf | STMicroelectronics, *DS12288 Datasheet: STM32G474xB STM32G474xC STM32G474xE*, Rev. 6, Nov. 2021. |
| `pm0214` | pm0214-stm32-cortexm4-mcus-and-mpus-programming-manual-stmicroelectronics.pdf | STMicroelectronics, *PM0214 Programming manual: STM32 Cortex®-M4 MCUs and MPUs programming manual*, Rev. 10, Mar. 2020. |
| `an2606` | an2606-stm32-microcontroller-system-memory-boot-mode-stmicroelectronics.pdf | STMicroelectronics, *AN2606 Application note: Introduction to system memory boot mode on STM32 MCUs*, Rev. 71, Nov. 2026. |
| `ds12504` | DS_stm32mp157d.pdf | STMicroelectronics, *DS12504 Datasheet: STM32MP157A/D*, Rev. 9, Sep. 2024. |
| `um2534` | um2534-discovery-kits-with-stm32mp157-mpus-stmicroelectronics.pdf | STMicroelectronics, *UM2534 User manual: Discovery kits with STM32MP157 MPUs*, Rev. 1, Mar. 2019. (LƯU Ý: Rev 1 chỉ mô tả STM32MP157A-DK1 / STM32MP157C-DK2) |
| `tcan334` | TCAN334_Datasheet.pdf | Texas Instruments, *TCAN33x 3.3-V CAN Transceivers with CAN FD (Flexible Data Rate)*, SLLSEQ7F, Dec. 2015 – Rev. May 2025. |
| `mcp2562fd` | MCP2562FD_Datasheet.pdf | Microchip Technology Inc., *MCP2561/2FD High-Speed CAN Flexible Data Rate Transceiver*, DS20005284A, 2014. |
| `tis_elf` | TIS_ELF_Specification_v1.2.pdf | TIS Committee, *Tool Interface Standard (TIS) Executable and Linking Format (ELF) Specification*, Version 1.2, May 1995. |
| `aaelf32` | ARM_ELF_for_Arm_Architecture_aaelf32.pdf | Arm Ltd., *ELF for the Arm® Architecture (AAELF32)*, 2023Q3. |

Quy tắc:
1. Mỗi thông số kỹ thuật (địa chỉ, dung lượng, tên thanh ghi, tên bit, số Figure/Table) phải tra được trong file trên.
2. Khi trích: ghi `\cite{key}` kèm vị trí, ví dụ `\cite[Section 2.6, Table 5, p. 91]{rm0440}`.
3. Kiến thức tổng quát không có trong các file trên thì viết như phân tích của tác giả, KHÔNG gắn trích dẫn giả.
