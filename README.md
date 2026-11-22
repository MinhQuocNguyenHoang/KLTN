# KLTN -- Automotive Central Gateway & Secure FOTA over CAN-FD

- **Đề tài:** Nghiên cứu, thiết kế hệ thống Automotive Central Gateway hỗ trợ cập nhật phần mềm an toàn (Secure FOTA) qua mạng CAN-FD ứng dụng trong ô tô
- **Sinh viên thực hiện:** Nguyễn Hoàng Minh Quốc (MSSV: 23207099) -- Khoa Điện tử - Viễn thông, ĐH KHTN ĐHQG-HCM
- **Giảng viên hướng dẫn:** Đỗ Quốc Minh Đăng
- **Phần cứng & Bảo mật:** `STM32MP157D-DK1` (Gateway) + `STM32G474VET6` (ECU) + `MCP2562FD` (CAN-FD Transceiver) + `SecOC (TinyJAMBU-128)`

## Cấu trúc thư mục

```text
KLTN/
├── docs/
│   ├── pdf/                       # [Tracked on Git] Chỉ chứa các file PDF báo cáo thành phẩm (00..06)
│   ├── latex/                     # [Ignored on Git] Nguồn LaTeX & công cụ build PDF cục bộ
│   │   ├── config/                # info.tex, preamble.tex dùng chung
│   │   ├── figures/               # Hình trích gốc (src/) và logo ĐH KHTN
│   │   ├── reports/               # File .tex từng báo cáo + _build/
│   │   ├── sections/              # Nội dung dạng module (task000.tex .. task006.tex)
│   │   ├── thesis/                # Khung quyển Báo cáo KLTN cuối kỳ (main.tex)
│   │   └── tools/                 # build.ps1, find_ref.ps1, render_fig.ps1
│   ├── references/                # [PDF Ignored, chỉ track README_references.md] Datasheet/RM/UM/RFC gốc
│   └── Yêu cầu và Nhiệm vụ.docx   # [Tracked on Git] Yêu cầu gốc của GVHD
├── gateway/                       # [Tracked on Git] Code Central Gateway (STM32MP157D-DK1)
│   ├── linux/                     # Cấu hình Yocto/Buildroot, Linux Device Tree (.dts)
│   └── app/                       # FOTA Client, UDS Master, SecOC (TinyJAMBU), SocketCAN
├── ecu/                           # [Tracked on Git] Firmware Target ECU (STM32G474VET6)
│   ├── bootloader/                # Custom Bootloader (A/B Partitioning, Verify, Rollback)
│   └── app/                       # ECU Application (UDS Server, SecOC TinyJAMBU, FDCAN)
├── shared/                        # [Tracked on Git] Thư viện dùng chung giữa Gateway & ECU
│   ├── crypto_tinyjambu/          # Cài đặt TinyJAMBU-128 AEAD
│   └── uds_protocol/              # Định nghĩa UDS (ISO 14229) & ISO-TP
├── server/                        # [Tracked on Git] FOTA Server giả lập (Host PC)
└── hardware/                      # [Tracked on Git] Sơ đồ nối dây, pinout, schematic, BOM
```
<<<<<<< HEAD
=======

## Biên dịch lại báo cáo từ LaTeX sang `docs/pdf/`

Chạy từ thư mục gốc `KLTN`:

```powershell
# Biên dịch toàn bộ báo cáo và tự xuất PDF vào docs/pdf/
.\docs\latex\tools\build.ps1 -All

# Biên dịch riêng 1 báo cáo
.\docs\latex\tools\build.ps1 -Name 00_HardwareCompatibility
```
>>>>>>> f908d9bbadd8b49e2bebdea307a9e5b6ca0b9dc2
