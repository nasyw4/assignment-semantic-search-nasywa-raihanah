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
| **Generate** | Hasil retrieval disisipkan sebagai *context* ke dalam prompt terstruktur (Persona, Task, Context, Format), dikirim ke Gemini (`gemini-2.5-flash`) untuk menghasilkan jawaban natural. Prompt mengarahkan LLM untuk menjawab hanya dari context dan mengakui ketidaktahuan bila informasi tidak tersedia. |

Seluruh pipeline digabung dalam fungsi `generate_answer(query)` yang menjalankan: query → retrieval (semantic search) → prompt building → LLM call → jawaban.

## 3. Hasil Pengujian & Analisis Relevansi

Diuji dengan 6 query: 2 direct match, 3 paraphrase/sinonim, 1 di luar knowledge base.

| # | Query | Tipe | FAQ Teratas yang Ditemukan | Relevan? |
| :-- | :---- | :---- | :---- | :---- |
| 1 | Berapa lama estimasi waktu pengiriman barang? | Direct | *(isi setelah run notebook)* | |
| 2 | Apakah member baru mendapat diskon khusus? | Direct | | |
| 3 | Barang saya belum sampai juga padahal udah lewat estimasi, itu kenapa ya? | Paraphrase | | |
| 4 | Bisa nggak sih bayar belanjaan pakai cicilan kartu kredit? | Paraphrase | | |
| 5 | Duit di rekening saya udah kepotong tapi status pesanan masih belum dibayar, kenapa tuh? | Paraphrase | | |
| 6 | Apakah platform ini menyediakan opsi investasi reksadana untuk pembeli? | Tidak ada di KB | | |

> **Catatan**: Isi kolom "FAQ Teratas" dan "Relevan?" setelah menjalankan `notebook/starter_notebook_semantic_search.ipynb` dari awal hingga akhir, lalu tuliskan analisis di bawah ini berdasarkan output aktual (lihat juga cell "Catatan Analisis Relevansi" di notebook).

**Analisis**:
- *(isi: query mana yang retrieval-nya akurat — biasanya direct match dan paraphrase berhasil karena embedding menangkap makna semantik, bukan hanya kata kunci)*
- *(isi: query mana yang kurang relevan, jika ada, dan kemungkinan sebabnya)*
- *(isi: bagaimana kualitas jawaban LLM pada query yang tidak ada di knowledge base — apakah LLM mengakui ketidaktahuan dengan baik atau justru berhalusinasi)*

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
