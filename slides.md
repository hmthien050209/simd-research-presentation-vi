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
  serif: Noto Sans
  mono: JetBrains Mono
class: text-left
hideInToc: true
---

# Tăng tốc chấm điểm trắc nghiệm trực tuyến: Tối ưu hóa SIMD trên kiến trúc x86_64

Hoàng Minh Thiên, Hoàng Bá Tùng, Đinh Nguyên Khoa, Nguyễn Khôi Nguyên - 11 Tin-LN

---
hideInToc: true
---

# Mục lục

<Toc :maxDepth="1" />

---

# Giới thiệu

## Nhóm nghiên cứu

- **Thành viên**: Hoàng Minh Thiên, Hoàng Bá Tùng, Đinh Nguyên Khoa, Nguyễn Khôi Nguyên.
- **Lớp**: 11 Tin-LN.
- **Tư cách**: Nhóm học sinh đam mê Khoa học Máy tính, mong muốn tìm hiểu sâu về kiến trúc máy tính và tối ưu hóa phần mềm.

---

# Bối cảnh

## Phạm vi nghiên cứu

- **Kiến trúc**: Intel x86_64 với các tập lệnh mở rộng AVX2, AVX512BW, AVX512VL, AVX512F, AVX512DQ.
- **Công cụ**: GNU Compiler Collection (GCC) >= 10.0.
- **Ngôn ngữ**: C++20.
- **Môi trường**: Đơn luồng (Single-threaded).

## Tại sao chọn đề tài này?

- Các kỳ thi trực tuyến ngày càng phổ biến.
- Hệ thống thường bị quá tải vào giờ cao điểm do lượng yêu cầu chấm điểm lớn.
- Chấm điểm là một tác vụ quan trọng cần độ chính xác và tốc độ cao.

---

# Xác định vấn đề

- **Nút thắt cổ chai (Bottleneck)**: Logic chấm điểm trắc nghiệm truyền thống xử lý từng câu hỏi một cách tuần tự.
- **Lãng phí tài nguyên**: Các CPU hiện đại có khả năng xử lý song song mạnh mẽ nhưng chưa được tận dụng hết trong các mã nguồn thông thường.
- **Mục tiêu**: Tăng tốc độ chấm điểm để giảm tải cho máy chủ và cải thiện trải nghiệm người dùng.

---

# Tài liệu liên quan (Related Work)

## Cảm hứng từ Daniel Lemire

- **Daniel Lemire**: Giáo sư Khoa học Máy tính, nổi tiếng với các nghiên cứu về tối ưu hóa hiệu năng phần mềm (SIMDJSON, nén số nguyên).
- **Ý tưởng**: Sử dụng SIMD (Single Instruction, Multiple Data) để xử lý dữ liệu lớn.
    - *Intersection of sorted arrays*: Tìm giao của hai danh sách (tương tự như so sánh đáp án bài làm và đáp án đúng).
    - *Bit manipulation*: Sử dụng các phép toán bit để lọc và đếm nhanh.

## Các nghiên cứu khác

- Ứng dụng SIMD trong lọc dữ liệu (Filtering) và tính tổng (Aggregation) trong cơ sở dữ liệu.
- Kỹ thuật "Branchless programming" (Lập trình không rẽ nhánh) để tránh dự đoán sai của CPU.

---

# Đề xuất giải pháp

Chúng tôi đề xuất chuyển đổi từ xử lý tuần tự sang xử lý song song cấp dữ liệu (Data-Level Parallelism) sử dụng SIMD Intrinsics.

---

## Cách tiếp cận ngây thơ (Naive Approach)

- **Mô tả**: Dùng vòng lặp `for` duyệt qua từng câu hỏi.
- **Logic**:
    1. Lấy đáp án của thí sinh tại vị trí `i`.
    2. So sánh với đáp án đúng tại vị trí `i`.
    3. Nếu khớp, cộng điểm vào tổng.
- **Nhược điểm**: Rất chậm, phụ thuộc vào trình biên dịch để tối ưu hóa (thường không hiệu quả).

---

## Tối ưu hóa không rẽ nhánh (Branchless)

- **Vấn đề của `if`**: CPU cố gắng dự đoán kết quả của lệnh `if`. Nếu đoán sai, CPU phải quay lại và làm lại, gây tốn thời gian.
- **Giải pháp**: Loại bỏ lệnh `if`.
    - Thay vì `if (đúng) cộng điểm`, ta dùng công thức: `Tổng += (đúng/sai) * điểm`.
    - Trong C++, `true` là 1, `false` là 0. Phép nhân sẽ tự động loại bỏ điểm nếu sai.

---

## Giải pháp SIMD (AVX2 / AVX512)

Thay vì chấm 1 câu, ta chấm **32 câu (AVX2)** hoặc **64 câu (AVX512)** cùng lúc.

### Quy trình xử lý (Pipeline):

1.  **Load**: Nạp 32/64 đáp án của thí sinh và 32/64 đáp án đúng vào các thanh ghi vector.
2.  **Compare**: So sánh song song hai vector này. Kết quả là một vector mặt nạ (mask) chứa các giá trị `0xFF` (đúng) hoặc `0x00` (sai).
3.  **Mask & Points**: Dùng vector mặt nạ để giữ lại điểm số của các câu đúng (dùng phép AND bitwise).
4.  **Horizontal Sum**: Cộng dồn tất cả các điểm số trong vector kết quả để ra tổng điểm của nhóm câu hỏi đó.

---

## Cải tiến cấu trúc dữ liệu

- **Vấn đề của `std::vector`**: Tiện lợi nhưng truy cập bộ nhớ đôi khi chậm hơn mảng thuần C do các lớp bảo vệ.
- **Giải pháp**: Tự xây dựng cấu trúc `ByteArray` tùy chỉnh.
    - Cấp phát bộ nhớ thẳng hàng (aligned memory) để tối ưu cho việc nạp dữ liệu vào thanh ghi SIMD.
    - Loại bỏ các kiểm tra dư thừa.
    - Tối ưu hóa bộ nhớ đệm (Cache friendly).

---

# So sánh và Đánh giá

Hệ thống đo đạc:
- Ubuntu 24.04 LTS, Intel i5-1135G7.
- So sánh thời gian thực thi (Real time) giữa các giải pháp.

---
layout: intro
---

## Kết quả Benchmark (Cấu trúc cũ)

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
layout: intro
---

## Kết quả Benchmark (Cấu trúc mới)

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

## So sánh với các nghiên cứu khác

- **Kết quả của chúng tôi**: Tăng tốc độ khoảng **3-7 lần** so với mã nguồn gốc.
- **So với Lemire**: Các kỹ thuật của Lemire (như trong SIMDJSON) thường đạt tốc độ xử lý hàng gigabyte/giây. Giải pháp của chúng tôi cũng đạt được hiệu suất cao tương tự nhờ áp dụng nguyên lý:
    - Giảm thiểu rẽ nhánh (Branch misprediction).
    - Tận dụng tối đa băng thông bộ nhớ.
    - Xử lý song song dữ liệu.

---

# Kết luận và Định hướng

## Kết luận
- Việc sử dụng SIMD mang lại hiệu quả vượt trội cho bài toán chấm điểm trắc nghiệm.
- Cần có kiến thức sâu về kiến trúc máy tính để viết mã tối ưu (Intrinsics).
- Không nên phụ thuộc hoàn toàn vào khả năng tự động vector hóa của trình biên dịch.

## Định hướng phát triển
- **GPU**: Thử nghiệm trên GPU (CUDA/OpenCL) để xử lý lượng dữ liệu lớn hơn nữa.
- **Đa luồng**: Kết hợp SIMD với đa luồng (Multi-threading) để tận dụng hết các nhân CPU.
- **Hỗ trợ ARM**: Mở rộng nghiên cứu sang kiến trúc ARM (Apple Silicon, Server ARM) với tập lệnh NEON/SVE.

---
layout: end
hideInToc: true
---

# Xin cảm ơn!