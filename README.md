# Semantic Search & LLM Application — E-commerce FAQ Assistant

Prototype asisten AI customer service e-commerce yang menjawab pertanyaan pelanggan secara otomatis berdasarkan knowledge base FAQ, menggunakan semantic search (Gemini Embeddings + FAISS) dan LLM (Gemini) untuk menghasilkan jawaban natural.

## 1. Problem Statement

**Objective**: Tim customer service menerima ratusan pertanyaan repetitif setiap hari seputar pemesanan, pengiriman, pembayaran, dan pengembalian barang. Tujuan prototype ini adalah membangun sistem yang dapat menemukan entri FAQ paling relevan terhadap pertanyaan pelanggan — meskipun pelanggan tidak menggunakan kata-kata yang sama persis dengan FAQ — lalu menyusun jawaban yang natural menggunakan LLM berdasarkan FAQ yang ditemukan.

**Sumber dan struktur knowledge base**: `data/ecommerce_faq_knowledge_base.csv`, berisi 59 entri FAQ dengan kolom:
- `faq_id` — ID unik entri
- `category` — kategori pertanyaan (Pemesanan, Pengiriman, Pembayaran, Pengembalian, Akun, Promo, Produk, Customer Service)
- `question` — pertanyaan FAQ
- `answer` — jawaban FAQ

**Batasan/asumsi**:
- Sistem hanya menjawab berdasarkan FAQ yang tersedia di knowledge base — tidak menggunakan pengetahuan umum LLM.
- Jika pertanyaan pelanggan tidak memiliki jawaban relevan di knowledge base, sistem akan memberitahukan bahwa informasi tidak tersedia dan menyarankan menghubungi customer service, bukan mengarang jawaban (menghindari halusinasi).
- Knowledge base tidak diubah secara langsung — file asli di `data/` tetap utuh.
- Tidak ada teknik chunking dokumen panjang karena setiap entri FAQ sudah berukuran pendek (question + answer digabung jadi satu unit retrievable).

## 2. Alur Sistem (Ingest → Embed → Store → Retrieve → Generate)

| Tahap | Implementasi |
| :---- | :---- |
| **Ingest** | Load `ecommerce_faq_knowledge_base.csv` dengan Pandas, gabungkan `question` + `answer` menjadi satu teks (`content`) per entri sebagai unit retrievable. |
| **Embed** | Setiap unit `content` diubah menjadi vector embedding menggunakan Gemini Embeddings API (`gemini-embedding-001`). |
| **Store** | Seluruh embedding disimpan ke index FAISS (`IndexFlatL2`), jumlah vektor diverifikasi sama dengan jumlah entri knowledge base. |
| **Retrieve** | Query pengguna diembed dengan model yang sama, dicari top-k entri FAQ terdekat di index FAISS berdasarkan L2 distance. |
| **Generate** | Hasil retrieval disisipkan sebagai *context* ke dalam prompt terstruktur (Persona, Task, Context, Format), dikirim ke Gemini (`gemini-flash-lite-latest`) untuk menghasilkan jawaban natural. Prompt mengarahkan LLM untuk menjawab hanya dari context dan mengakui ketidaktahuan bila informasi tidak tersedia. |

Seluruh pipeline digabung dalam fungsi `generate_answer(query)` yang menjalankan: query → retrieval (semantic search) → prompt building → LLM call → jawaban.

## 3. Hasil Pengujian & Analisis Relevansi

Diuji dengan 6 query: 2 direct match, 3 paraphrase/sinonim, 1 di luar knowledge base.

| # | Query | Tipe | FAQ Teratas yang Ditemukan | Distance | Relevan? |
| :-- | :---- | :---- | :---- | :---- | :---- |
| 1 | Berapa lama estimasi waktu pengiriman barang? | Direct | Berapa lama estimasi waktu pengiriman barang? | 0.195 | Ya |
| 2 | Apakah member baru mendapat diskon khusus? | Direct | Apakah member baru mendapat diskon khusus? | 0.188 | Ya |
| 3 | Barang saya belum sampai juga padahal udah lewat estimasi, itu kenapa ya? | Paraphrase | Bagaimana jika paket saya belum sampai melebihi estimasi? | 0.257 | Ya |
| 4 | Bisa nggak sih bayar belanjaan pakai cicilan kartu kredit? | Paraphrase | Apakah saya bisa membayar dengan cicilan? | 0.217 | Ya |
| 5 | Duit di rekening saya udah kepotong tapi status pesanan masih belum dibayar, kenapa tuh? | Paraphrase | Kenapa pembayaran saya berhasil dipotong tapi status pesanan masih belum terbayar? | 0.243 | Ya |
| 6 | Apakah platform ini menyediakan opsi investasi reksadana untuk pembeli? | Tidak ada di KB | Bagaimana cara melakukan pemesanan di platform ini? | 0.681 | Tidak (sesuai ekspektasi) |

**Analisis**:
- **Direct match (#1-2)**: Keduanya berhasil retrieve FAQ yang identik persis dengan pertanyaan asli, distance sangat kecil (~0.19). Hasil ini jadi baseline: ketika query sangat mirip kata-katanya dengan FAQ, distance selalu rendah artinya mirip/relevan.
- **Paraphrase/sinonim (#3-5)**: Ketiganya tetap berhasil menemukan FAQ yang benar secara makna meskipun kata-katanya berbeda atau jauh berbeda dari FAQ asli — bahkan query #5 memakai bahasa sangat informal ("duit", "kepotong", "tuh") dan tetap match ke FAQ "Kenapa pembayaran saya berhasil dipotong tapi status pesanan masih belum terbayar?" dengan distance 0.243. Ini membuktikan semantic search berhasil menangkap makna, bukan sekadar mencocokkan kata kunci, dan distance-nya hanya sedikit lebih tinggi dari query yang direct match (0.21-0.26 vs 0.19) masih dibawah 0.5, bukan melonjak jauh.
- **Di luar knowledge base (#6)**: Pertanyaan soal investasi reksadana tidak ada hubungannya dengan FAQ e-commerce manapun. Distance FAQ teratas melonjak ke 0.681 (lebih dari 0.5), jauh lebih besar dibanding 5 query lain (semua di bawah 0.5 bahkan di bawah 0.3). Ini jadi sinyal yang jelas: ketika distance jauh lebih tinggi dari biasanya, kemungkinan besar tidak ada jawaban relevan di knowledge base. Tapi di sini LLM merespons dengan baik: mengakui tidak memiliki informasi tersebut dan menyarankan menghubungi customer service, tanpa mengarang jawaban (tidak halusinasi) dimana sesuai instruksi prompt di bagian TASK (masih sesuai ekspektasi).
- **Kesimpulan**: Sistem semantic search bekerja baik untuk direct match maupun paraphrase — tidak hanya cocok kata kunci, tapi benar-benar menangkap makna. Ada pola yang berguna: distance di bawah ~0.3 cenderung berarti ada jawaban relevan di Knowledge Base, sedangkan distance di atas ~0.6 mengindikasikan pertanyaan di luar cakupan Knowledge Base. Namun ini pola empiris dari 6 sampel, bukan threshold yang sudah divalidasi secara statistik karena perlu lebih banyak data uji untuk menentukan ambang batas yang andal. Prompt design (instruksi eksplisit untuk menjawab hanya dari context dan mengaku tidak tahu bila informasi tidak tersedia) berhasil mencegah LLM berhalusinasi pada query di luar Knowledge Base.

## 4. Keterbatasan Sistem

- Vector search menggunakan `IndexFlatL2` (brute-force) — cukup untuk 59 entri, namun tidak scalable untuk knowledge base berukuran besar tanpa index yang lebih efisien (misal IVF/HNSW).
- Tidak ada mekanisme chunking untuk dokumen panjang (di luar scope assignment ini).
- Similarity/distance score tidak memiliki threshold otomatis untuk menolak hasil yang kurang relevan — sistem selalu mengembalikan top-k meskipun secara semantik jauh, sehingga keputusan "relevan atau tidak" sepenuhnya diserahkan ke instruksi prompt pada LLM.
- Tidak ada memory/riwayat percakapan — setiap query diproses independen.
- Kualitas jawaban bergantung pada kelengkapan knowledge base; pertanyaan di luar topik FAQ hanya bisa dijawab dengan "informasi tidak tersedia", bukan jawaban faktual lain.

## 5. Cara Menjalankan

1. Clone repository ini dan masuk ke folder `semantic-search-assignment/`.
2. Buat virtual environment (opsional) lalu install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Salin `.env.example` menjadi `.env`, lalu isi dengan API key Gemini milik Anda:
   ```
   GEMINI_API_KEY=your_gemini_api_key_here
   ```
4. Buka dan jalankan `notebook/starter_notebook_semantic_search.ipynb` dari awal hingga akhir (Run All).
5. Hasil embedding, FAISS index, semantic search, dan jawaban LLM untuk setiap query uji akan tercetak langsung di output notebook.
