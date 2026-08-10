# So sánh Bloom Filter và Counting Bloom Filter trong lọc dữ liệu tốc độ cao — Báo cáo học thuật kèm mã LaTeX và mã thực nghiệm Python

**Bloom Filter (BF) là lựa chọn tối ưu về bộ nhớ và tốc độ cho lọc dữ liệu tốc độ cao trên tập hợp *tĩnh*, còn Counting Bloom Filter (CBF) chỉ nên dùng khi bắt buộc phải *xóa* phần tử động — vì CBF phải trả giá bằng bộ nhớ gấp khoảng 4 lần (theo phân tích của Fan et al., 2000, "4 bits per counter should suffice for most applications") và có nguy cơ phát sinh false negative mà BF không bao giờ mắc phải.**

## TL;DR
- **BF vs CBF:** BF tiết kiệm bộ nhớ (1 bit/ô) và không bao giờ có false negative, nhưng *không xóa được*; CBF thay bit bằng counter 4-bit nên *xóa được* nhưng tốn ~4× bộ nhớ và có thể sinh false negative khi counter bão hòa/xóa nhầm. Cả hai có cùng công thức false positive nếu cùng số ô m, số hàm băm k và số phần tử n.
- **Khi nào dùng cái gì:** Chọn BF cho tập gần như tĩnh (bộ lọc mỗi SSTable trong Cassandra/RocksDB/HBase, Cache Digest của Squid, ký tự signature cố định trong DPI/Snort). Chọn CBF khi tập thay đổi liên tục (router xóa flow cũ, cache thay thế nội dung, deduplication có thu hồi). Nếu cần xóa *và* FPR mục tiêu ε < 3%, Cuckoo Filter thường tốt hơn CBF (ít không gian hơn ~ dưới mức Bloom tối ưu-không-gian).
- **Sản phẩm kèm theo:** Báo cáo cung cấp (a) tài liệu LaTeX hoàn chỉnh compile được bằng `pdflatex`/`xelatex` với mã giả insert/query/delete và bảng so sánh `booktabs`, và (b) mã Python đầy đủ (mmh3 + bitarray + numpy + matplotlib) tự cài đặt BF/CBF, đo FPR thực nghiệm vs lý thuyết, throughput và bộ nhớ.

## Key Findings

1. **Công thức FPR chuẩn (Bloom 1970, phổ biến qua Mullin/Broder–Mitzenmacher):** xác suất một bit còn 0 là (1−1/m)^{kn} ≈ e^{−kn/m}; xác suất false positive p ≈ (1 − e^{−kn/m})^k. Số hàm băm tối ưu k* = (m/n) ln 2, khi đó fraction bit 0 = 1/2 (entropy cực đại). Bộ nhớ tối ưu m = −n ln p / (ln 2)². Quy tắc thực dụng: m = 8n cho FPR ≈ 2%, m = 10n cho FPR < 1% (Berkeley CS170: "Already m = 8n reduces the chance of error to roughly 2%, and m = 10n to less than 1%").

2. **CBF tốn ~4× bộ nhớ:** Broder & Mitzenmacher, *Network Applications of Bloom Filters: A Survey* (Internet Mathematics 1(4):485–509, 2004) nêu rõ: "The analysis from [Fan et al. 00] reveals that 4 bits per counter should suffice for most applications." Một CBF b-bit tiêu tốn gấp b lần bộ nhớ của BF cùng số ô; với b = 4 đó là hệ số 4.

3. **Chứng minh 4 bit là đủ (nguồn gốc Fan et al. 2000):** Số phần tử băm vào một counter xấp xỉ Poisson với kỳ vọng kn/m = ln 2 ở cấu hình tối ưu. Xác suất một counter đạt 16 (gây tràn 4-bit) ≈ e^{−ln2}(ln2)¹⁶/16! ≈ 6.78×10⁻¹⁷. Fan et al. chứng minh với k ≤ (ln 2)m/n, Pr(max counter ≥ 16) ≤ m·1.37×10⁻¹⁵, và kết luận verbatim: "if we allow 4 bits per count, the probability of overflow for practical values of [n] during the initial insertion in the table is minuscule … it seems that 4 bits per count would be amply sufficient."

4. **CBF có thể sinh false negative:** Fan et al. xử lý tràn bằng saturating counter — "if the count ever exceeds 15, we can simply let it stay at 15; after many deletions this might lead to a situation where the Bloom filter allows a false negative (the count becomes 0 when it shouldn't be)". Ngoài ra, xóa một phần tử *chưa từng chèn* (hoặc bị nhận nhầm là thành viên do false positive) cũng làm giảm sai counter và gây false negative cho phần tử thật.

5. **Biến thể tiết kiệm hơn CBF khi cần xóa:**
   - **Cuckoo Filter** (Fan, Andersen, Kaminsky, Mitzenmacher, CoNEXT 2014): "it requires less space than a space-optimized Bloom filter when the target false positive rate ε is less than 3%", đo được dùng "30% less space for the same false positive rate". Ở ε = 3%, Cuckoo/Bloom cần ~7 bit/entry; Bloom đạt ε = 1% cần ~10 bit/item.
   - **d-left CBF** (Bonomi et al., ESA 2006): "uses less space, generally saving a factor of two or more"; theo Rothenberg et al. (arXiv:1005.0352) dlCBF "requires about half the bit-space m of a 4-bit CBF".

## Details

### 1. Bối cảnh: lọc dữ liệu tốc độ cao

Bài toán chung là **kiểm tra thành viên tập hợp (approximate set-membership)**: cho tập S và một khóa x (gói tin, chuỗi ký tự signature, khóa CSDL, fingerprint khối dữ liệu), trả lời nhanh "x *chắc chắn không* thuộc S" hay "x *có thể* thuộc S", với ngân sách bộ nhớ nhỏ để nằm gọn trong SRAM/cache tốc độ cao. Việc chấp nhận một tỉ lệ nhỏ false positive (Bloom gọi là "errors of commission") cho phép giảm mạnh không gian lưu trữ so với bảng băm chính xác — đây chính là ý tưởng đánh đổi space/time của Bloom (1970): "allowing a small number of test messages to be falsely identified as members of the given set will permit a much smaller hash area to be used without increasing reject time."

Các miền ứng dụng tiêu biểu:
- **Network packet filtering & DPI/IDS:** kiểm tra payload chứa signature mã độc (Snort). Dharmapurikar, Krishnamurthy, Sproull, Lockwood, *Deep Packet Inspection using Parallel Bloom Filters* (Hot Interconnects 11, 2003; IEEE Micro 24:52–61, 2004): dùng BF cứng trên FPGA Xilinx Virtex 2000E, hỗ trợ 10.000 chuỗi, đạt ~600 Mbps (prototype) và về sau các thiết kế >10 Gbps với <50 KB on-chip memory + vài MB SRAM ngoài trên tập signature của Snort.
- **Database query filtering (LSM-tree):** Cassandra, RocksDB, HBase, LevelDB, ScyllaDB gắn một BF cho mỗi SSTable để bỏ qua các SSTable "definitely not present", tránh I/O đĩa. Cassandra để lộ tham số `bloom_filter_fp_chance` (mặc định 1–2%).
- **Cache filtering / cooperative web caching:** Squid *Cache Digests* và giao thức *Summary Cache* dùng BF/CBF để tóm tắt nội dung cache.
- **Deduplication:** dùng BF trên fingerprint khối/tệp để loại nhanh các khối trùng mà không phải nạp chỉ mục từ đĩa (ví dụ MAD2 dùng Bloom Filter Array).

### 2. Cơ sở lý thuyết

#### 2.1 Bloom Filter (Burton H. Bloom, 1970)
BF là vector B gồm m bit, khởi tạo toàn 0, với k hàm băm độc lập, phân bố đều h₁,…,h_k: U → {0,…,m−1}.
- **Insert(x):** tính h₁(x),…,h_k(x) và đặt B[h_i(x)] = 1.
- **Query(x):** nếu tất cả B[h_i(x)] = 1 → trả "có thể có" (maybe); nếu có bất kỳ bit nào = 0 → "chắc chắn không" (definitely no).

BF **không có false negative** (bit đã set không bao giờ tự về 0) nhưng **có false positive** (do va chạm băm). BF **không hỗ trợ xóa**: reset các bit của x có thể xóa nhầm bit đang được phần tử khác dùng chung, gây false negative.

Nguồn gốc: B. H. Bloom, "Space/time trade-offs in hash coding with allowable errors," *Communications of the ACM*, 13(7):422–426, 1970, DOI 10.1145/362686.362692.

#### 2.2 Counting Bloom Filter (Fan et al., 2000)
CBF thay mỗi bit bằng một **counter nhỏ** (thường 4 bit). Gọi c(i) là giá trị counter thứ i.
- **Insert(x):** tăng c(h_i(x)) lên 1 với mọi i.
- **Delete(x):** giảm c(h_i(x)) đi 1 với mọi i.
- **Query(x):** nếu tất cả c(h_i(x)) > 0 → "có thể có"; ngược lại → "chắc chắn không".

CBF cho phép xóa vì mỗi counter đếm số phần tử băm vào ô đó. Nhược điểm: **tràn counter (overflow)** khi nhiều phần tử cùng băm vào một ô vượt 2^b − 1, và **false negative** nếu áp dụng saturating counter hoặc xóa sai.

Nguồn gốc: L. Fan, P. Cao, J. Almeida, A. Z. Broder, "Summary Cache: A Scalable Wide-Area Web Cache Sharing Protocol," *IEEE/ACM Transactions on Networking*, 8(3):281–293, 2000, DOI 10.1109/90.851975 (bản hội nghị: SIGCOMM'98). Trong đó "the directory representations are very economical, as low as 8 bits per entry", giảm số thông điệp liên proxy "by a factor of 25 to 60" và băng thông "by over 50%". Cấu hình thử nghiệm dùng load factor m/n ∈ {8, 16, 32} với 4 hàm băm; "A load factor between 8 and 16 works well".

#### 2.3 Công thức toán học đầy đủ

**(a) Xác suất false positive.** Sau khi chèn n phần tử với k hàm băm, xác suất một bit cụ thể còn 0:
p₀ = (1 − 1/m)^{kn} ≈ e^{−kn/m}.
Xác suất false positive (tất cả k bit của khóa lạ đều là 1):
**p ≈ (1 − e^{−kn/m})^k** (dạng chặt hơn: (1 − (1 − 1/m)^{kn})^k).

**(b) Số hàm băm tối ưu.** Lấy đạo hàm p theo k và cho bằng 0 (tương đương yêu cầu fraction bit 0 = 1/2 để entropy cực đại):
**k* = (m/n) ln 2 ≈ 0.693 (m/n)**.

**(c) FPR tại k tối ưu.** Thay k* vào: p = (1/2)^{k*} = 2^{−(m/n) ln 2} ≈ 0.6185^{m/n}.

**(d) Bộ nhớ tối ưu** cho FPR mục tiêu p:
**m = − n ln p / (ln 2)²**, tương đương m/n = −log₂ p / ln 2 ≈ 1.44 log₂(1/p).
Ví dụ số: m/n = 10 → FPR ≈ 1% (Fan et al.: "for a bit array 10 times larger than the number of entries, the probability of a false positive is 1.2% for four hash functions, and 0.9% for the optimum case of five hash functions"); m/n = 8 → FPR ≈ 2%.

**(e) Xác suất tràn counter trong CBF.** Số phần tử băm vào counter thứ i là biến ngẫu nhiên nhị thức B(nk, 1/m), xấp xỉ Poisson với kỳ vọng λ = kn/m. Ở cấu hình tối ưu k = (ln 2)m/n → λ = ln 2. Xác suất một counter đạt j:
Pr(c(i) ≥ j) ≤ C(nk, j)(1/m)^j.
Với xấp xỉ Poisson, xác suất một counter bằng 16:
Pr(c(i) = 16) ≈ e^{−ln 2}(ln 2)¹⁶ / 16! ≈ **6.78×10⁻¹⁷**.
Fan et al. chặn trên toàn cục: với k ≤ (ln 2)m/n,
**Pr(max_i c(i) ≥ 16) ≤ m · 1.37×10⁻¹⁵**.

**(f) Vì sao 4 bit là đủ.** Counter 4-bit đếm 0..15; tràn xảy ra khi cần đạt 16. Vì xác suất trên nhỏ đến mức "minuscule" (với m ~ 10⁷, cận trên ~ 1.37×10⁻⁸), 4 bit đủ cho hầu hết ứng dụng. Với đề phòng, dùng saturating counter (giữ ở 15) — đánh đổi bằng khả năng false negative cực hiếm, mà Fan et al. lập luận "it is much more likely that the proxy server would be rebooted in the meantime and the entire structure reconstructed."

### 3. So sánh chi tiết theo tiêu chí

| Tiêu chí | Bloom Filter (BF) | Counting Bloom Filter (CBF) |
|---|---|---|
| Đơn vị ô | 1 bit | counter b-bit (thường 4 bit) |
| Bộ nhớ (cùng m ô, k) | m bit | b·m bit → ~4× khi b=4 |
| Insert | set k bit → O(k) | tăng k counter → O(k) |
| Query | AND k bit → O(k) | so sánh k counter > 0 → O(k) |
| Delete | **Không hỗ trợ** | **Hỗ trợ** (giảm k counter) |
| False positive | Có, p ≈ (1−e^{−kn/m})^k | Có, cùng công thức |
| False negative | **Không bao giờ** | **Có thể** (tràn/saturate hoặc xóa sai) |
| FPR ở cùng bộ nhớ vật lý | Thấp hơn (nhiều bit hơn cho cùng số byte) | Cao hơn (ít ô hơn cho cùng số byte) |
| Cache-friendly | k truy cập ngẫu nhiên → tối đa k cache miss | tương tự, ô lớn hơn nên footprint lớn hơn |
| Hợp phần cứng (FPGA/ASIC, SRAM) | Rất tốt: bit-array on-chip, k hàm băm song song, cập nhật chỉ-thêm | Cập nhật động được (tăng/giảm), nhưng tốn SRAM hơn; thường CBF giữ trong SW còn BF trong HW |
| Song song hóa cập nhật | Ghi 1 bit an toàn | Đọc–sửa–ghi counter cần đồng bộ |
| Ứng dụng điển hình | SSTable filter (Cassandra/RocksDB/HBase), Cache Digest (Squid), DPI signature tĩnh (Snort), dedup fingerprint | Router xóa flow cũ, cache thay thế nội dung, Summary Cache cập nhật động, đếm bội |

**Diễn giải quan trọng:** Khi so *cùng số ô m*, BF và CBF có FPR giống hệt nhau — CBF không "chính xác hơn". Khác biệt duy nhất là CBF thêm khả năng xóa với giá bộ nhểư ~4×. Do đó, khi *ngân sách byte cố định*, BF luôn cho FPR thấp hơn CBF vì có gấp 4 lần số ô. Chỉ chọn CBF khi *bắt buộc* phải xóa và không thể rebuild lại toàn bộ filter định kỳ.

### 4. Các biến thể liên quan (đặt trong bối cảnh)

- **Blocked Bloom Filter** (Putze, Sanders, Singler, 2007/2010): chia m bit thành các block cỡ đúng một cache line (thường 512 bit); mọi bit của một khóa nằm trong cùng một block → giảm số cache miss xuống ~1 trên truy vấn âm, đổi lại FPR tăng nhẹ do va chạm nội block. RocksDB dùng chiến lược này. Không hỗ trợ xóa (như BF).
- **Cuckoo Filter** (Fan et al., CoNEXT 2014): lưu fingerprint trong bảng băm cuckoo; **hỗ trợ xóa**, lookup nhanh hơn BF, và "requires less space than a space-optimized Bloom filter when the target false positive rate ε is less than 3%" (dùng "30% less space for the same false positive rate"). Nhược điểm: chèn có thể thất bại khi tải > ~95%, đôi khi quan sát false negative ở ngưỡng tải cao.
- **d-left Counting Bloom Filter (dlCBF)** (Bonomi, Mitzenmacher, Panigrahy, Singh, Varghese, ESA 2006): dùng d-left hashing + fingerprint, "uses less space, generally saving a factor of two or more" so với CBF, tức chỉ ~½ bit-space của CBF 4-bit — lựa chọn thay thế CBF tốt trên phần cứng.

### 5. Ứng dụng thực tế (bằng chứng)

- **DPI/IDS (Snort):** parallel Bloom filters trên FPGA (Dharmapurikar et al.), on-chip BF loại bỏ phần lớn truy cập bộ nhớ, cho phép matching ở tốc độ dây (wire speed). CBF (giữ trong phần mềm) dùng để cập nhật tập signature động và hỗ trợ xác minh.
- **CSDL LSM-tree:** RocksDB/LevelDB/Cassandra/HBase gắn BF mỗi SSTable; giảm read amplification bằng cách trả lời "definitely not present" mà không mở tệp. RocksDB dùng double hashing (Kirsch–Mitzenmacher) để sinh k vị trí từ 2 hàm băm, và từng vá lỗi tinh chỉnh delta lẻ để tránh trùng vị trí khi m là lũy thừa 2.
- **Web cache (Squid):** Cache Digests = biến thể BF, khóa = method + URI, không bao giờ false-miss (chỉ false-hit → remote miss). Summary Cache khuyến nghị dùng CBF *cục bộ* để proxy theo dõi thay đổi nội dung cache, rồi phát bản BF 0/1 cho các proxy khác.
- **Deduplication:** BF/BF-array lọc fingerprint để tránh tra cứu đĩa; hệ thống backup dùng BF cho "quick index".

## Sản phẩm 1 — Mã LaTeX hoàn chỉnh (compile bằng `pdflatex` hoặc `xelatex`)

> Ghi chú biên dịch: mặc định dùng `pdflatex` với gói `vietnam`+`inputenc(utf8)`. Nếu dùng `xelatex`, hãy **bỏ comment** khối `fontspec`/`polyglossia` và **comment** khối `inputenc`/`vietnam` (đã ghi chú rõ trong mã).

```latex
\documentclass[11pt,a4paper]{article}

% ====== HỖ TRỢ TIẾNG VIỆT ======
% --- Cách A: pdflatex (mặc định) ---
\usepackage[utf8]{inputenc}
\usepackage[T5]{fontenc}
\usepackage[vietnam]{babel}
% --- Cách B: xelatex (bỏ comment 3 dòng dưới, comment 3 dòng trên) ---
% \usepackage{fontspec}
% \usepackage{polyglossia}
% \setmainlanguage{vietnamese}\setotherlanguage{english}\setmainfont{Latin Modern Roman}

\usepackage{amsmath,amssymb}
\usepackage{graphicx}
\usepackage{booktabs}
\usepackage{array}
\usepackage{algorithm}
\usepackage{algpseudocode}   % thuộc gói algorithmicx
\usepackage{listings}
\usepackage{xcolor}
\usepackage[hidelinks]{hyperref}
\usepackage{geometry}
\geometry{margin=2.5cm}

\lstset{basicstyle=\ttfamily\small,breaklines=true,frame=single,
        columns=fullflexible,showstringspaces=false}

\title{So sánh Bloom Filter và Counting Bloom Filter\\
       trong bài toán lọc dữ liệu tốc độ cao}
\author{Báo cáo học thuật}
\date{\today}

\begin{document}
\maketitle
\begin{abstract}
Báo cáo so sánh Bloom Filter (BF, Bloom 1970) và Counting Bloom Filter
(CBF, Fan \emph{et al.} 2000) cho lọc dữ liệu tốc độ cao (network packet
filtering, deep packet inspection, database query filtering, cache filtering,
deduplication). Chúng tôi trình bày cơ sở lý thuyết, công thức xác suất
false positive, phân tích tràn counter, mã giả cho insert/query/delete,
bảng so sánh, thiết kế thực nghiệm và mã nguồn Python.
\end{abstract}

\section{Giới thiệu}
Lọc dữ liệu tốc độ cao yêu cầu kiểm tra thành viên tập hợp
(\emph{set-membership}) với bộ nhớ nhỏ và thông lượng lớn. BF là cấu trúc
dữ liệu xác suất kinh điển; CBF mở rộng BF để hỗ trợ thao tác xóa
(\emph{delete}). Bài toán tiêu biểu: DPI/IDS (Snort), bộ lọc SSTable trong
Cassandra/RocksDB/HBase, Cache Digest của Squid, và deduplication.

\section{Cơ sở lý thuyết}
\subsection{Bloom Filter}
BF gồm mảng $m$ bit khởi tạo $0$ và $k$ hàm băm độc lập
$h_1,\dots,h_k:\mathcal{U}\to\{0,\dots,m-1\}$. \emph{Insert} đặt các bit
$B[h_i(x)]=1$; \emph{Query} trả ``có thể có'' nếu mọi bit bằng $1$. BF không
có false negative nhưng không hỗ trợ xóa.

\subsection{Counting Bloom Filter}
CBF thay mỗi bit bằng counter $b$ bit (thường $b=4$). \emph{Insert} tăng,
\emph{Delete} giảm các counter tương ứng. CBF hỗ trợ xóa nhưng có thể tràn
counter và sinh false negative.

\section{Công thức toán học}
Xác suất một bit còn $0$ sau khi chèn $n$ phần tử:
\begin{equation}
\left(1-\tfrac{1}{m}\right)^{kn}\approx e^{-kn/m}.
\end{equation}
Xác suất false positive:
\begin{equation}
p \;\approx\; \left(1-e^{-kn/m}\right)^{k}.
\end{equation}
Số hàm băm tối ưu và bộ nhớ tối ưu:
\begin{equation}
k^{*}=\frac{m}{n}\ln 2,\qquad
m=-\frac{n\ln p}{(\ln 2)^{2}}.
\end{equation}
Tại $k^{*}$: $p=2^{-(m/n)\ln 2}\approx 0.6185^{\,m/n}$.

\paragraph{Tràn counter (CBF).} Số phần tử vào một counter $\sim$
Binomial$(nk,1/m)\approx$ Poisson$(\lambda)$ với $\lambda=kn/m=\ln 2$ ở
cấu hình tối ưu. Do đó
\begin{equation}
\Pr(c(i)=16)\approx \frac{e^{-\ln 2}(\ln 2)^{16}}{16!}\approx 6.78\times10^{-17},
\end{equation}
và với $k\le (\ln 2)m/n$: $\Pr(\max_i c(i)\ge 16)\le m\cdot 1.37\times10^{-15}$.
Vì vậy $b=4$ bit/counter là đủ cho hầu hết ứng dụng.

\section{Mã giả}
\begin{algorithm}[h]
\caption{Bloom Filter: Insert và Query}
\begin{algorithmic}[1]
\Procedure{BF-Insert}{$x$}
  \For{$i \gets 1$ to $k$} \State $B[h_i(x)] \gets 1$ \EndFor
\EndProcedure
\Procedure{BF-Query}{$x$}
  \For{$i \gets 1$ to $k$}
     \If{$B[h_i(x)] = 0$} \State \Return \textsc{False} \EndIf
  \EndFor
  \State \Return \textsc{True} \Comment{có thể có (maybe)}
\EndProcedure
\end{algorithmic}
\end{algorithm}

\begin{algorithm}[h]
\caption{Counting Bloom Filter: Insert, Query, Delete}
\begin{algorithmic}[1]
\Procedure{CBF-Insert}{$x$}
  \For{$i \gets 1$ to $k$}
     \If{$C[h_i(x)] < 2^{b}-1$} \State $C[h_i(x)] \gets C[h_i(x)]+1$
     \EndIf \Comment{saturating}
  \EndFor
\EndProcedure
\Procedure{CBF-Query}{$x$}
  \For{$i \gets 1$ to $k$}
     \If{$C[h_i(x)] = 0$} \State \Return \textsc{False} \EndIf
  \EndFor
  \State \Return \textsc{True}
\EndProcedure
\Procedure{CBF-Delete}{$x$}
  \If{\textsc{CBF-Query}$(x)$} \Comment{chỉ xóa nếu có mặt}
     \For{$i \gets 1$ to $k$}
        \If{$C[h_i(x)] > 0$} \State $C[h_i(x)] \gets C[h_i(x)]-1$ \EndIf
     \EndFor
  \EndIf
\EndProcedure
\end{algorithmic}
\end{algorithm}

\section{Bảng so sánh}
\begin{table}[h]\centering
\caption{So sánh Bloom Filter và Counting Bloom Filter}
\begin{tabular}{@{}lll@{}}
\toprule
Tiêu chí & Bloom Filter & Counting Bloom Filter \\
\midrule
Đơn vị ô        & 1 bit                 & counter $b$ bit ($b{=}4$) \\
Bộ nhớ          & $m$ bit               & $b\cdot m$ bit ($\sim 4\times$) \\
Xóa (delete)    & Không                 & Có \\
False positive  & Có                    & Có (cùng công thức) \\
False negative  & Không bao giờ         & Có thể (tràn/xóa sai) \\
Insert/Query    & $O(k)$                & $O(k)$ \\
Phần cứng       & Rất tốt (SRAM on-chip)& Tốn SRAM hơn \\
\bottomrule
\end{tabular}
\end{table}

\section{Thực nghiệm}
Chúng tôi đo (i) FPR thực nghiệm vs lý thuyết theo $m/n$, (ii) throughput
(ops/s) cho insert/query/delete, (iii) bộ nhớ. Chi tiết và mã nguồn Python
đi kèm báo cáo. Kỳ vọng: đường FPR thực nghiệm bám sát
$p=(1-e^{-kn/m})^k$; throughput BF cao hơn CBF một chút; bộ nhớ CBF $\approx$
$4\times$ BF.

\section{Kết luận}
BF tối ưu cho tập tĩnh; CBF cần khi phải xóa động, trả giá $\sim4\times$ bộ
nhớ và rủi ro false negative. Khi cần xóa với FPR mục tiêu $<3\%$, Cuckoo
Filter thường vượt trội CBF.

\begin{thebibliography}{9}
\bibitem{bloom1970} B.~H. Bloom, ``Space/time trade-offs in hash coding with
allowable errors,'' \emph{Communications of the ACM}, vol.~13, no.~7,
pp.~422--426, 1970.
\bibitem{fan2000} L.~Fan, P.~Cao, J.~Almeida, and A.~Z. Broder, ``Summary
Cache: A Scalable Wide-Area Web Cache Sharing Protocol,'' \emph{IEEE/ACM
Transactions on Networking}, vol.~8, no.~3, pp.~281--293, 2000.
\bibitem{broder2004} A.~Broder and M.~Mitzenmacher, ``Network Applications of
Bloom Filters: A Survey,'' \emph{Internet Mathematics}, vol.~1, no.~4,
pp.~485--509, 2004.
\bibitem{putze2010} F.~Putze, P.~Sanders, and J.~Singler, ``Cache-, hash-, and
space-efficient Bloom filters,'' \emph{ACM Journal of Experimental
Algorithmics}, vol.~14, 2010.
\bibitem{kirsch2008} A.~Kirsch and M.~Mitzenmacher, ``Less hashing, same
performance: Building a better Bloom filter,'' \emph{Random Structures \&
Algorithms}, vol.~33, no.~2, pp.~187--218, 2008.
\bibitem{bonomi2006} F.~Bonomi, M.~Mitzenmacher, R.~Panigrahy, S.~Singh, and
G.~Varghese, ``An Improved Construction for Counting Bloom Filters,'' in
\emph{ESA 2006}, LNCS 4168, pp.~684--695.
\bibitem{cuckoo2014} B.~Fan, D.~G. Andersen, M.~Kaminsky, and M.~D.
Mitzenmacher, ``Cuckoo Filter: Practically Better Than Bloom,'' in
\emph{CoNEXT 2014}, pp.~75--88.
\bibitem{dpi2003} S.~Dharmapurikar, P.~Krishnamurthy, T.~Sproull, and
J.~Lockwood, ``Deep Packet Inspection using Parallel Bloom Filters,'' in
\emph{IEEE Hot Interconnects 11}, 2003.
\end{thebibliography}
\end{document}
```

### Trích dẫn BibTeX (dùng với `biblatex`/`bibtex`)

```bibtex
@article{bloom1970,
  author  = {Bloom, Burton H.},
  title   = {Space/time trade-offs in hash coding with allowable errors},
  journal = {Communications of the ACM},
  volume  = {13}, number = {7}, pages = {422--426}, year = {1970},
  doi     = {10.1145/362686.362692}
}
@article{fan2000,
  author  = {Fan, Li and Cao, Pei and Almeida, Jussara and Broder, Andrei Z.},
  title   = {Summary Cache: A Scalable Wide-Area Web Cache Sharing Protocol},
  journal = {IEEE/ACM Transactions on Networking},
  volume  = {8}, number = {3}, pages = {281--293}, year = {2000},
  doi     = {10.1109/90.851975}
}
@article{broder2004,
  author  = {Broder, Andrei and Mitzenmacher, Michael},
  title   = {Network Applications of Bloom Filters: A Survey},
  journal = {Internet Mathematics},
  volume  = {1}, number = {4}, pages = {485--509}, year = {2004},
  doi     = {10.1080/15427951.2004.10129096}
}
@article{putze2010,
  author  = {Putze, Felix and Sanders, Peter and Singler, Johannes},
  title   = {Cache-, hash-, and space-efficient Bloom filters},
  journal = {ACM Journal of Experimental Algorithmics},
  volume  = {14}, year = {2010}, doi = {10.1145/1498698.1594230}
}
@article{kirsch2008,
  author  = {Kirsch, Adam and Mitzenmacher, Michael},
  title   = {Less hashing, same performance: Building a better Bloom filter},
  journal = {Random Structures \& Algorithms},
  volume  = {33}, number = {2}, pages = {187--218}, year = {2008},
  doi     = {10.1002/rsa.20208}
}
@inproceedings{bonomi2006,
  author    = {Bonomi, Flavio and Mitzenmacher, Michael and Panigrahy, Rina
               and Singh, Sushil and Varghese, George},
  title     = {An Improved Construction for Counting Bloom Filters},
  booktitle = {Algorithms -- ESA 2006}, series = {LNCS}, volume = {4168},
  pages     = {684--695}, year = {2006}, doi = {10.1007/11841036\_61}
}
@inproceedings{cuckoo2014,
  author    = {Fan, Bin and Andersen, David G. and Kaminsky, Michael
               and Mitzenmacher, Michael D.},
  title     = {Cuckoo Filter: Practically Better Than Bloom},
  booktitle = {Proc. 10th ACM CoNEXT}, pages = {75--88}, year = {2014},
  doi       = {10.1145/2674005.2674994}
}
@inproceedings{dpi2003,
  author    = {Dharmapurikar, Sarang and Krishnamurthy, Praveen and
               Sproull, Todd and Lockwood, John},
  title     = {Deep Packet Inspection using Parallel Bloom Filters},
  booktitle = {IEEE Hot Interconnects 11}, year = {2003},
  doi       = {10.1109/HOTI.2003.1253315}
}
```

## Sản phẩm 2 — Thiết kế thực nghiệm chi tiết

**Mục tiêu.** Định lượng đánh đổi giữa BF và CBF: (1) FPR thực nghiệm có bám công thức lý thuyết p = (1−e^{−kn/m})^k không; (2) throughput insert/query/delete; (3) bộ nhớ thực tế; (4) ảnh hưởng của tỉ lệ tải n/m.

**Giả thuyết.**
- H1: FPR thực nghiệm của cả BF và CBF (cùng m, k, n) trùng nhau và bám đường lý thuyết trong sai số thống kê.
- H2: Bộ nhớ CBF ≈ 4× BF (với counter 4-bit lưu bằng byte thì lên tới 8× nếu dùng `uint8`; dùng nibble-packing sẽ đúng 4×).
- H3: Throughput query của BF ≥ CBF (thao tác bit rẻ hơn so sánh byte); CBF thêm chi phí delete mà BF không có.
- H4: FPR giảm theo hàm mũ khi m/n tăng; k tối ưu = round((m/n) ln 2) cho FPR thấp nhất.

**Tham số.** n ∈ {10⁴, 10⁵, 10⁶}; m/n ∈ {4, 6, 8, 10, 12, 16}; k tự tính k = round((m/n) ln 2) hoặc quét k ∈ {1..12}; kích thước counter b = 4 bit (mô phỏng bằng `uint8` + saturate ở 15, hoặc packing nibble).

**Tập dữ liệu.**
- *Tổng hợp:* chuỗi ngẫu nhiên/UUID; tập "có mặt" (n phần tử được chèn) và tập "vắng mặt" (n′ phần tử chưa chèn) để đo FPR.
- *Thực:* danh sách URL (ví dụ tập tên miền công khai) hoặc IP; khóa = chuỗi.

**Phép đo.**
- *FPR thực nghiệm* = (#truy vấn dương với phần tử vắng mặt)/n′; so với p lý thuyết.
- *Throughput* (ops/s) = số thao tác / thời gian, đo riêng insert, query (hit và miss), delete (chỉ CBF).
- *Bộ nhớ* = số byte cấp phát thật (`bitarray` cho BF, `numpy uint8` cho CBF).
- *Ảnh hưởng n/m*: vẽ FPR và throughput theo m/n.

**Bảng kết quả mẫu kỳ vọng** (n = 100.000, k = round((m/n)ln2)):

| m/n | k | FPR lý thuyết | FPR thực nghiệm (kỳ vọng) | Mem BF | Mem CBF (uint8) |
|---|---|---|---|---|---|
| 6  | 4 | ~5.6%  | ~5–6%   | 1× | ~8× |
| 8  | 6 | ~2.1%  | ~2%     | 1× | ~8× |
| 10 | 7 | ~0.8%  | ~0.8–1% | 1× | ~8× |
| 12 | 8 | ~0.3%  | ~0.3%   | 1× | ~8× |

(Lưu ý: với `uint8`, CBF tốn 8 bit/ô → 8× BF; muốn đúng 4× phải packing nibble — mã dưới có phần chú thích cách làm.)

## Sản phẩm 3 — Mã Python đầy đủ (chạy được)

**Cài đặt môi trường:**
```bash
python3 -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install mmh3 bitarray numpy matplotlib
```

**File `bloom_filters.py` — cài đặt BF và CBF từ đầu:**
```python
"""Bloom Filter và Counting Bloom Filter cài đặt từ đầu.
Hàm băm: mmh3 (MurmurHash3) + double hashing (Kirsch-Mitzenmacher):
    g_i(x) = (h1(x) + i * h2(x)) mod m
"""
import math
import mmh3
import numpy as np
from bitarray import bitarray


def optimal_m(n: int, p: float) -> int:
    """Bộ nhớ tối ưu m = -n ln p / (ln 2)^2."""
    return max(1, int(math.ceil(-n * math.log(p) / (math.log(2) ** 2))))


def optimal_k(m: int, n: int) -> int:
    """Số hàm băm tối ưu k = (m/n) ln 2 (làm tròn, tối thiểu 1)."""
    return max(1, int(round((m / n) * math.log(2))))


def theoretical_fpr(m: int, n: int, k: int) -> float:
    """FPR lý thuyết p = (1 - e^{-kn/m})^k."""
    return (1.0 - math.exp(-k * n / m)) ** k


def _positions(key, k: int, m: int):
    """Sinh k vị trí bằng double hashing từ 2 seed murmur."""
    h1 = mmh3.hash(key, 0, signed=False)
    h2 = mmh3.hash(key, 1, signed=False)
    for i in range(k):
        yield (h1 + i * h2) % m


class BloomFilter:
    def __init__(self, m: int, k: int):
        self.m, self.k = m, k
        self.bits = bitarray(m)
        self.bits.setall(0)

    def insert(self, key):
        for pos in _positions(key, self.k, self.m):
            self.bits[pos] = 1

    def query(self, key) -> bool:
        return all(self.bits[pos] for pos in _positions(key, self.k, self.m))

    def memory_bytes(self) -> int:
        return self.m // 8 + (1 if self.m % 8 else 0)


class CountingBloomFilter:
    """CBF với counter 4-bit mô phỏng bằng uint8 (saturate ở 15).
    Ghi chú: uint8 tốn 8 bit/ô (=> 8x BF). Muốn đúng 4x (nibble packing),
    thay self.counts bằng mảng nửa kích thước và truy cập nibble cao/thấp;
    ở đây ưu tiên rõ ràng nên dùng uint8."""
    MAX = 15  # 2^4 - 1

    def __init__(self, m: int, k: int):
        self.m, self.k = m, k
        self.counts = np.zeros(m, dtype=np.uint8)

    def insert(self, key):
        for pos in _positions(key, self.k, self.m):
            if self.counts[pos] < self.MAX:
                self.counts[pos] += 1

    def query(self, key) -> bool:
        return all(self.counts[pos] > 0 for pos in _positions(key, self.k, self.m))

    def delete(self, key):
        # chỉ xóa nếu phần tử được xem là có mặt (tránh giảm sai)
        if self.query(key):
            for pos in _positions(key, self.k, self.m):
                if 0 < self.counts[pos] < self.MAX:  # không giảm counter đã bão hòa
                    self.counts[pos] -= 1

    def memory_bytes(self) -> int:
        return self.counts.nbytes            # uint8: m byte
        # nibble packing: return self.m // 2 + (self.m % 2)
```

**File `benchmark.py` — đo FPR, throughput, bộ nhớ và vẽ biểu đồ:**
```python
import time
import uuid
import numpy as np
import matplotlib.pyplot as plt
from bloom_filters import (BloomFilter, CountingBloomFilter,
                           optimal_k, theoretical_fpr)

def gen_keys(n):
    return [uuid.uuid4().hex for _ in range(n)]

def measure_fpr(filter_obj, absent_keys):
    fp = sum(1 for k in absent_keys if filter_obj.query(k))
    return fp / len(absent_keys)

def timed(fn, keys):
    t0 = time.perf_counter()
    for k in keys:
        fn(k)
    dt = time.perf_counter() - t0
    return len(keys) / dt, dt   # ops/s, seconds

def run(n=100_000, ratios=(4, 6, 8, 10, 12, 16)):
    present = gen_keys(n)
    absent  = gen_keys(n)
    rows, xs, fpr_th, fpr_bf, fpr_cbf = [], [], [], [], []
    thr_bf_q, thr_cbf_q, thr_cbf_del = [], [], []

    for r in ratios:
        m = r * n
        k = optimal_k(m, n)
        bf  = BloomFilter(m, k)
        cbf = CountingBloomFilter(m, k)

        ins_bf,  _ = timed(bf.insert,  present)
        ins_cbf, _ = timed(cbf.insert, present)

        q_bf,  _   = timed(bf.query,  absent)
        q_cbf, _   = timed(cbf.query, absent)
        del_cbf, _ = timed(cbf.delete, present)   # xóa toàn bộ để đo delete

        # xây lại cbf để đo FPR (vì delete ở trên đã làm rỗng)
        cbf = CountingBloomFilter(m, k)
        for x in present: cbf.insert(x)

        p_th  = theoretical_fpr(m, n, k)
        p_bf  = measure_fpr(bf,  absent)
        p_cbf = measure_fpr(cbf, absent)

        xs.append(r); fpr_th.append(p_th); fpr_bf.append(p_bf); fpr_cbf.append(p_cbf)
        thr_bf_q.append(q_bf); thr_cbf_q.append(q_cbf); thr_cbf_del.append(del_cbf)
        rows.append((r, k, p_th, p_bf, p_cbf,
                     bf.memory_bytes(), cbf.memory_bytes(),
                     ins_bf, ins_cbf, q_bf, q_cbf, del_cbf))

    # In bảng
    print(f"{'m/n':>4} {'k':>3} {'FPR_th':>8} {'FPR_BF':>8} {'FPR_CBF':>8} "
          f"{'MemBF':>9} {'MemCBF':>9} {'Qbf/s':>10} {'Qcbf/s':>10}")
    for (r,k,pth,pbf,pcbf,mbf,mcbf,ibf,icbf,qbf,qcbf,dcbf) in rows:
        print(f"{r:>4} {k:>3} {pth:>8.4f} {pbf:>8.4f} {pcbf:>8.4f} "
              f"{mbf:>9} {mcbf:>9} {qbf:>10.0f} {qcbf:>10.0f}")

    # Biểu đồ 1: FPR theo m/n
    plt.figure()
    plt.plot(xs, fpr_th,  'o-', label='FPR lý thuyết')
    plt.plot(xs, fpr_bf,  's--', label='FPR BF (thực nghiệm)')
    plt.plot(xs, fpr_cbf, '^:', label='FPR CBF (thực nghiệm)')
    plt.yscale('log'); plt.xlabel('m/n'); plt.ylabel('False Positive Rate')
    plt.title('FPR theo tỉ lệ m/n'); plt.legend(); plt.grid(True)
    plt.savefig('fpr_vs_mn.png', dpi=150, bbox_inches='tight')

    # Biểu đồ 2: throughput query
    plt.figure()
    plt.plot(xs, thr_bf_q,  'o-', label='BF query')
    plt.plot(xs, thr_cbf_q, 's--', label='CBF query')
    plt.plot(xs, thr_cbf_del, '^:', label='CBF delete')
    plt.xlabel('m/n'); plt.ylabel('ops/giây'); plt.title('Throughput')
    plt.legend(); plt.grid(True)
    plt.savefig('throughput.png', dpi=150, bbox_inches='tight')
    print("Đã lưu fpr_vs_mn.png và throughput.png")

if __name__ == '__main__':
    run(n=100_000)
```

**Hướng dẫn từng bước:**
1. Tạo môi trường và cài gói (lệnh `pip install` ở trên).
2. Lưu 2 file trên vào cùng thư mục.
3. Chạy: `python benchmark.py`. Kết quả in ra bảng và tạo 2 file PNG.
4. **Thay đổi tham số:** sửa `run(n=..., ratios=(...))`; muốn quét k thủ công, gọi `BloomFilter(m, k)` với k tùy ý thay vì `optimal_k`. Dùng dataset thực: thay `gen_keys(n)` bằng đọc file URL/IP (`present = open('urls.txt').read().split()`).
5. **Đọc kết quả:** cột `FPR_th` (lý thuyết) nên gần `FPR_BF`/`FPR_CBF` (thực nghiệm) — xác nhận H1. `MemCBF` ≈ 8× `MemBF` với `uint8` (H2, bật nibble packing để về 4×). `Qbf/s` thường ≥ `Qcbf/s` (H3). FPR giảm mũ khi m/n tăng (H4).

**Gợi ý C++ (đo hiệu năng cao hơn):** dùng `xxHash` (xxh3) cho hàm băm nhanh; `std::vector<uint8_t>` cho CBF (hoặc packing 2 nibble/byte để đúng 4×), `std::vector<bool>` hoặc mảng `uint64_t` + bit ops cho BF; đo thời gian bằng `std::chrono::high_resolution_clock`; biên dịch `-O3 -march=native`. Kỳ vọng throughput cao hơn Python 1–2 bậc độ lớn, làm rõ khác biệt query BF vs CBF vốn bị Python che khuất.

## Recommendations

Các bước triển khai theo giai đoạn, kèm ngưỡng quyết định:

1. **Xác định tập có động không.** Nếu tập gần như tĩnh hoặc có thể rebuild định kỳ (SSTable bất biến, Cache Digest phát lại mỗi chu kỳ, signature Snort ít đổi) → **dùng BF**. *Ngưỡng chuyển hướng:* nếu tần suất xóa/cập nhật khiến rebuild toàn bộ filter trở nên tốn kém hơn chi phí bộ nhớ 4× → chuyển sang cấu trúc hỗ trợ xóa.

2. **Chọn tham số BF.** Đặt FPR mục tiêu p (ví dụ 1%), tính m = −n ln p/(ln 2)² (≈ 9.6n cho p=1%), k = round((m/n) ln 2) (≈ 7). *Kiểm chứng:* chạy `benchmark.py`, xác nhận FPR thực nghiệm ≤ FPR mục tiêu; nếu vượt, tăng m/n.

3. **Nếu bắt buộc xóa động:**
   - FPR mục tiêu ≥ 3% hoặc phần cứng đơn giản/đếm bội cần thiết → **CBF 4-bit** (đúng bản gốc Fan et al.). Dùng saturating counter và lịch rebuild định kỳ để triệt tiêu tích lũy false negative.
   - FPR mục tiêu < 3% và ưu tiên bộ nhớ/tốc độ lookup → **Cuckoo Filter** ("less space than a space-optimized Bloom filter when the target false positive rate ε is less than 3%"), hoặc **dlCBF** trên phần cứng ("about half the bit-space of a 4-bit CBF"). *Ngưỡng cảnh báo:* Cuckoo Filter có thể thất bại chèn khi tải > ~95% — theo dõi load factor và mở rộng bảng trước ngưỡng này.

4. **Nếu throughput bị nghẽn bởi cache miss** (filter lớn hơn cache, tỉ lệ true-positive cao) → **Blocked Bloom Filter** (block = 1 cache line, giảm còn ~1 cache miss), chấp nhận FPR tăng nhẹ. RocksDB đã dùng cách này.

5. **Phần cứng (FPGA/ASIC, line-rate):** đặt bit-array BF trong on-chip SRAM, tính k hàm băm song song; giữ CBF trong SRAM/phần mềm cho cập nhật động (mô hình lai BF-cứng + CBF-mềm như trong DPI/NDN-NIC). Dùng double hashing (Kirsch–Mitzenmacher) để giảm số hàm băm cần tính từ k xuống 2.

## Caveats

- **Công thức FPR cổ điển là *xấp xỉ*, và thực ra là *cận dưới*.** Bose et al. (2008) và Christensen et al. (2010) chỉ ra công thức (1−e^{−kn/m})^k đánh giá thấp FPR thật khi m nhỏ; với filter lớn (m→∞) mới hội tụ. Với m nhỏ, hãy đo thực nghiệm thay vì tin tuyệt đối vào công thức.
- **Giả định hàm băm lý tưởng.** Phân tích giả định k hàm băm độc lập, phân bố đều. Thực tế dùng double hashing từ 2 hàm băm (Kirsch–Mitzenmacher) — hợp lệ tiệm cận nhưng có thể lệch với filter nhỏ/vừa; RocksDB từng phải vá lỗi double hashing (đảm bảo delta lẻ khi m là lũy thừa 2). Trong môi trường đối kháng (adversarial), tham số tính theo trung bình có thể bị khai thác gây DoS — cần dùng hàm băm có khóa/mật mã.
- **Chi phí bộ nhớ CBF trong mã Python** ở trên là 8× (do `uint8`), không phải 4× như lý thuyết. Muốn đạt đúng 4× phải nibble-packing (2 counter/byte) — đã ghi chú trong mã; điều này không đổi kết luận định tính.
- **False negative của CBF** trong thực tế thường rất hiếm (Fan et al. gọi xác suất tràn là "minuscule"), nhưng *tích lũy theo thời gian* qua nhiều chu kỳ insert/delete và qua các thao tác xóa sai (xóa phần tử được nhận nhầm do false positive). Lịch rebuild định kỳ là biện pháp phòng ngừa thực dụng.
- **Số liệu hiệu năng phần cứng** (600 Mbps trên Virtex 2000E; >10 Gbps với <50 KB on-chip + vài MB SRAM) là từ các prototype nghiên cứu (Dharmapurikar et al.) trên nền tảng và tập signature cụ thể; hiệu năng thực tế phụ thuộc công nghệ FPGA/ASIC, tần số và khối lượng luật — nên xem là minh họa xu hướng, không phải cam kết cho mọi hệ thống.