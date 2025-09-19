# Clean Architecture MASTER Prompt - Phiên Bản Cuối Cùng

## Tổng Quan
Đây là bộ tài liệu hoàn chỉnh về Clean Architecture với các phiên bản khác nhau, từ cơ bản đến nâng cao, bao gồm cả phiên bản tiếng Việt.

## Cấu Trúc Tài Liệu

### 📁 Các Phiên Bản Có Sẵn

1. **`clean_arch_MASTER_prompt_v2_with_naming_rules.txt`** - Phiên bản gốc với naming rules
2. **`clean_arch_MASTER_prompt_v3_with_constants_rules_EN.txt`** - Phiên bản với constants rules (tiếng Anh)
3. **`clean_arch_prompt_v4.txt`** - Phiên bản cải tiến với cấu trúc tối ưu
4. **`clean_arch_prompt_v5.txt`** - Phiên bản cuối cùng (tiếng Anh) với cấu trúc hoàn thiện
5. **`clean_arch_prompt_v5_VI.txt`** - Phiên bản tiếng Việt hoàn chỉnh

### 🎯 Khuyến Nghị Sử Dụng

#### Cho Developers Quốc Tế:
- **Sử dụng**: `clean_arch_prompt_v5.txt`
- **Lý do**: Cấu trúc tối ưu, đầy đủ tính năng, dễ đọc

#### Cho Developers Việt Nam:
- **Sử dụng**: `clean_arch_prompt_v5_VI.txt`
- **Lý do**: Dịch thuật hoàn chỉnh, giữ nguyên thuật ngữ kỹ thuật

### 📋 Tính Năng Chính

#### ✅ Phiên Bản v5 (Khuyến Nghị)
- **Mục lục** với navigation links
- **Cấu trúc tối ưu** với 13 sections rõ ràng
- **Naming rules chi tiết** cho từng layer
- **Error handling** với error catalog
- **Constants management** với quy tắc rõ ràng
- **Execution playbook** 7 bước
- **Enterprise essentials** đầy đủ
- **Feature switchboard** linh hoạt

#### 🔧 Các Cải Tiến So Với Phiên Bản Trước
1. **Tái cấu trúc** sections theo logic workflow
2. **Gộp nội dung** trùng lặp
3. **Chia nhỏ** sections dài thành subsections
4. **Thêm mục lục** và navigation
5. **Dịch thuật** hoàn chỉnh sang tiếng Việt

### 🚀 Cách Sử Dụng

#### 1. Chọn Phiên Bản Phù Hợp
```bash
# Cho dự án quốc tế
cp clean_arch_prompt_v5.txt your_project/

# Cho dự án Việt Nam
cp clean_arch_prompt_v5_VI.txt your_project/
```

#### 2. Tích Hợp Vào Dự Án
- Đặt file prompt vào thư mục `docs/` hoặc `prompts/`
- Tham khảo khi thiết kế architecture
- Sử dụng làm checklist cho code review

#### 3. Customization
- Có thể chỉnh sửa theo nhu cầu dự án cụ thể
- Thêm/bớt rules theo team conventions
- Tích hợp với CI/CD pipeline

### 📚 Tài Liệu Tham Khảo

#### Clean Architecture Concepts
- **Domain Layer**: Pure business logic, no dependencies
- **Application Layer**: Use cases, ports, DTOs
- **Infrastructure Layer**: External concerns, adapters
- **Interfaces Layer**: UI, API, CLI entry points

#### Key Principles
- **SOLID** principles
- **Ports & Adapters** pattern
- **Anti-Corruption Layer** (ACL)
- **Dependency Injection** (DI)
- **12-Factor App** methodology

### 🔄 Lịch Sử Phiên Bản

| Version | Language | Key Features | Status |
|---------|----------|--------------|--------|
| v2 | EN | Naming rules | Legacy |
| v3 | EN | Constants rules | Legacy |
| v4 | EN | Restructured | Legacy |
| v5 | EN | Final optimized | ✅ Current |
| v5 | VI | Vietnamese translation | ✅ Current |

### 📞 Hỗ Trợ

Nếu cần hỗ trợ hoặc có câu hỏi về Clean Architecture:
1. Tham khảo tài liệu trong thư mục này
2. Kiểm tra examples trong code
3. Liên hệ team architecture

---

**Lưu ý**: Đây là tài liệu master tổng hợp, được cập nhật thường xuyên. Luôn sử dụng phiên bản mới nhất cho dự án.
