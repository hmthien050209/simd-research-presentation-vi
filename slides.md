---
theme: seriph
title: 'Tăng tốc chấm điểm trắc nghiệm trực tuyến: Tối ưu hóa SIMD trên kiến trúc x86_64'
info: Demonstrate ways to improve performance of MCQs scoring systems using x86_64 SIMD
author: Hoàng Minh Thiên <hoangminhthien05022009@gmail.com>
lineNumbers: true
colorSchema: dark
drawings:
  persist: false
transition: slide-left
mdc: true
fonts:
  sans: Noto Sans
  serif: Noto Sans
  mono: JetBrains Mono
class: text-left
hideInToc: true
---

# Tăng tốc chấm điểm trắc nghiệm trực tuyến: Tối ưu hóa SIMD trên kiến trúc x86_64

Pokerface: Hoàng Minh Thiên, Hoàng Bá Tùng, Đinh Nguyên Khoa, Nguyễn Khôi Nguyên - 11 Tin-LN

---

# Giới thiệu nhóm nghiên cứu

- **Thành viên**: Hoàng Minh Thiên, Hoàng Bá Tùng, Đinh Nguyên Khoa, Nguyễn Khôi Nguyên.
- **Lớp**: 11 Tin-LN.
- **Mục tiêu**: Tìm hiểu sâu về kiến trúc máy tính và tối ưu hóa phần mềm.

---

# Bối cảnh (5W-1H)

- **What**: Tối ưu hóa hệ thống chấm điểm trắc nghiệm.
- **Why**: Hệ thống quá tải giờ cao điểm, cần tốc độ & chính xác.
- **Who**: Các nền tảng thi trực tuyến, trường học.
- **Where**: Máy chủ kiến trúc x86_64 (AVX2, AVX512).
- **When**: Khi xử lý lượng lớn yêu cầu đồng thời.
- **How**: Sử dụng SIMD Intrinsics, C++20, Đơn luồng.

---

# Vấn đề & Nghiên cứu liên quan

## Vấn đề
- **Bottleneck**: Xử lý tuần tự lãng phí tài nguyên CPU.
- **Mục tiêu**: Tăng tốc độ, giảm tải máy chủ.

## Nghiên cứu liên quan
- **Daniel Lemire**: SIMDJSON, nén số nguyên.
- **Kỹ thuật**: Branchless programming, Bit manipulation.

---

# Giải pháp đề xuất

- **Naive**: Duyệt tuần tự `for`, so sánh từng câu (Chậm).
- **Branchless**: Loại bỏ `if`, dùng toán tử bit (Khá).
- **SIMD (AVX2/512)**:
    - Xử lý **32/64 câu** cùng lúc.
    - **Pipeline**: Load -> Compare -> Mask -> Sum.
- **Data Structure**: `ByteArray` tùy chỉnh, aligned memory, cache-friendly.

---
layout: image
image: /img/benchmark_results_avx512.json_100000.png
---

---
layout: image
image: /img/benchmark_results_avx512.json_5000000.png
---

---
layout: image
image: /img/benchmark_results_avx512.json_10000000.png
---

---
layout: image
image: /img/benchmark_results_avx512_new_struct.json_100000.png
---

---
layout: image
image: /img/benchmark_results_avx512_new_struct.json_5000000.png
---

---
layout: image
image: /img/benchmark_results_avx512_new_struct.json_10000000.png
---

---

# Kết luận & Định hướng

- **Kết quả**: Tăng tốc **3-7 lần** so với mã nguồn gốc.
- **Bài học**: SIMD hiệu quả nhưng phức tạp, cần hiểu rõ phần cứng.
- **Định hướng**:
    - Mở rộng sang **GPU** (CUDA).
    - Kết hợp **Đa luồng** (Multi-threading).
    - Hỗ trợ **ARM** (NEON/SVE).

---

# Tài liệu tham khảo

- Intel&reg; Intrinsics Guide.
- Intel&reg; 64 and IA-32 Architectures Software Developer's Manual Volume 1: Basic Architecture.

---
layout: end
hideInToc: true
---

# Xin cảm ơn!