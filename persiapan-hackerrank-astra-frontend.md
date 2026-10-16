# Persiapan HackerRank: Astra Front-End Developer (React) Hiring Test 2026

Kompilasi semua soal latihan dan kunci jawaban. Tiap section berisi **Soal** lalu **Kunci Jawaban**. Kerjakan dulu soalnya sendiri, baru bandingkan. Bandingkan **prinsipnya**, bukan kemiripan kodenya.

## Informasi Tes

- Durasi: **210 menit**, batas login terakhir **17 Oktober 2026 pukul 17.45 WIB**
- Dikerjakan **mandiri tanpa alat eksternal**. Ada deteksi plagiarisme dan AI. Tidak boleh searching saat tes. Jika ragu, tanyakan ke recruiter.
- Kamera/proctoring tergantung pengaturan perusahaan. Cek halaman instruksi sebelum Start dan siapkan kamera, pencahayaan, koneksi, dan Chrome terbaru.

| No | Section | Soal |
|---|---|---|
| 1 | JavaScript (Intermediate) | 1 |
| 2 | CSS | 1 |
| 3 | React (Intermediate) | 1 |
| 4 | TypeScript (Intermediate) | 1 |
| 5 | HTML/CSS/JS | 1 |
| 6 | React (Intermediate) | 1 |
| 7 | React (Basic) | 1 |
| 8 | REST API (Advanced) | 2 |
| 9 | Problem Solving (Intermediate) | 1 |

## Strategi Waktu (210 menit)

- 0-5 menit: lihat semua soal, urutkan dari yang paling pasti
- Soal cepat (JS, CSS, React Basic): 15-20 menit masing-masing
- Soal sedang (React Intermediate x2, TypeScript, HTML/CSS/JS): 20-25 menit masing-masing
- REST API x2 dan Problem Solving: 20-30 menit masing-masing
- Sisakan 10-15 menit terakhir untuk menjalankan ulang test case
- Solusi yang lolos sebagian lebih baik daripada kosong. Jangan terjebak di satu soal.

## Daftar Isi

0. Latihan Tambahan: Task Board (React, async)
1. JavaScript (Intermediate)
2. CSS
3. React (Intermediate)
4. TypeScript (Intermediate)
5. HTML/CSS/JS (Vanilla)
8. REST API (Advanced)
9. Problem Solving (Intermediate)
10. Checklist Malam Sebelum Tes

---

# 0. Latihan Tambahan: Task Board dengan Optimistic Update dan Race Condition

Target 90-120 menit. Tantangan utama ada di *async handling*.

## Soal

### API palsu (`api.js`)

```js
let db = Array.from({ length: 50 }, (_, i) => ({
  id: i + 1,
  title: `Task ${i + 1}`,
  done: i % 3 === 0,
}));

const delay = (ms) => new Promise((r) => setTimeout(r, ms));

export async function fetchTasks({ query = "", page = 1, pageSize = 10 }) {
  await delay(100 + Math.random() * 700);
  const filtered = db.filter((t) =>
    t.title.toLowerCase().includes(query.toLowerCase())
  );
  return {
    items: filtered.slice((page - 1) * pageSize, page * pageSize),
    total: filtered.length,
  };
}

export async function toggleTask(id) {
  await delay(300 + Math.random() * 500);
  if (Math.random() < 0.3) throw new Error("Server error");
  db = db.map((t) => (t.id === id ? { ...t, done: !t.done } : t));
  return db.find((t) => t.id === id);
}
```

### Persyaratan

1. **Pencarian dengan debounce**: input `data-testid="search"`, debounce 300ms, mengetik mereset halaman ke 1.
2. **Race condition**: respons lama tidak boleh menimpa respons terbaru. Gunakan `AbortController` atau flag, jelaskan pilihan Anda.
3. **Pagination**: `data-testid="prev"` / `"next"`, teks `data-testid="page-info"` berformat `Page 2 of 5`. Tombol nonaktif di batas dan saat loading.
4. **Optimistic update dengan rollback**: klik `data-testid="toggle-{id}"` langsung mengubah UI. Jika gagal, kembalikan dan tampilkan `data-testid="toast"` berisi `Failed to update Task {id}`, hilang otomatis setelah 3 detik. Klik ganda cepat pada item yang sama harus konsisten dengan server.
5. **Statistik turunan**: `data-testid="done-count"` = jumlah task selesai pada halaman tampil, tanpa state tambahan.
6. **Custom hook** `useTasks({ query, page })` mengembalikan `{ items, total, loading, error, toggle }`.
7. **Kualitas**: tanpa warning `key`, tanpa memory leak, tanpa infinite loop. Ada state `loading`, `error`, dan tombol `retry`.

### Jebakan

Stale closure di debounce, setState setelah unmount, rollback yang menimpa update lain, `useEffect` dua kali di Strict Mode, halaman melebihi total setelah query berubah.

### Pertanyaan teori

1. Mengapa `setState(prev => ...)` lebih aman daripada `setState(items.map(...))` pada optimistic update?
2. Apa beda `AbortController` dengan flag pembatalan?
3. Kapan `useMemo` untuk `done-count` tidak diperlukan?

## Kunci Jawaban

### `useTasks.js`

```js
import { useState, useEffect, useRef, useCallback } from "react";
import { fetchTasks, toggleTask } from "./api";

export function useTasks({ query, page, pageSize = 10, onToggleError }) {
  const [items, setItems] = useState([]);
  const [total, setTotal] = useState(0);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [reloadKey, setReloadKey] = useState(0);

  const mounted = useRef(true);
  const chains = useRef(new Map());      // id -> antrean request per item
  const onErrorRef = useRef(onToggleError);

  useEffect(() => { onErrorRef.current = onToggleError; });
  useEffect(() => {
    mounted.current = true;
    return () => { mounted.current = false; };
  }, []);

  // Fetch + anti race condition
  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    setError(null);

    fetchTasks({ query, page, pageSize })
      .then((res) => {
        if (cancelled) return;
        setItems(res.items);
        setTotal(res.total);
        setLoading(false);
      })
      .catch((e) => {
        if (cancelled) return;
        setError(e);
        setLoading(false);
      });

    return () => { cancelled = true; };
  }, [query, page, pageSize, reloadKey]);

  const retry = useCallback(() => setReloadKey((k) => k + 1), []);

  const flip = (id) => (prev) =>
    prev.map((t) => (t.id === id ? { ...t, done: !t.done } : t));

  const toggle = useCallback((id) => {
    setItems(flip(id));                                // 1. optimistic

    const prevChain = chains.current.get(id) ?? Promise.resolve();
    const next = prevChain.then(async () => {          // 2. serial per id
      try {
        await toggleTask(id);
      } catch {
        if (!mounted.current) return;
        setItems(flip(id));                            // 3. rollback klik ini saja
        onErrorRef.current?.(id);
      }
    });
    chains.current.set(id, next);
  }, []);

  return { items, total, loading, error, toggle, retry };
}
```

### `App.jsx`

```jsx
import { useState, useEffect, useRef, useCallback } from "react";
import { useTasks } from "./useTasks";

const PAGE_SIZE = 10;

export default function App() {
  const [input, setInput] = useState("");
  const [query, setQuery] = useState("");
  const [page, setPage] = useState(1);
  const [toast, setToast] = useState(null);
  const toastTimer = useRef();

  useEffect(() => {
    const h = setTimeout(() => {
      setQuery(input);
      setPage(1);
    }, 300);
    return () => clearTimeout(h);
  }, [input]);

  useEffect(() => () => clearTimeout(toastTimer.current), []);

  const showToast = useCallback((msg) => {
    setToast(msg);
    clearTimeout(toastTimer.current);
    toastTimer.current = setTimeout(() => setToast(null), 3000);
  }, []);

  const { items, total, loading, error, toggle, retry } = useTasks({
    query,
    page,
    pageSize: PAGE_SIZE,
    onToggleError: (id) => showToast(`Failed to update Task ${id}`),
  });

  const totalPages = Math.max(1, Math.ceil(total / PAGE_SIZE));
  const doneCount = items.filter((t) => t.done).length; // derived

  return (
    <main>
      <label htmlFor="search">Search tasks</label>
      <input
        id="search"
        data-testid="search"
        value={input}
        onChange={(e) => setInput(e.target.value)}
      />

      {error && (
        <div role="alert">
          Something went wrong. <button onClick={retry}>Retry</button>
        </div>
      )}
      {loading && <p aria-live="polite">Loading...</p>}

      <ul>
        {items.map((t) => (
          <li key={t.id}>
            <label>
              <input
                type="checkbox"
                data-testid={`toggle-${t.id}`}
                checked={t.done}
                onChange={() => toggle(t.id)}
              />
              {t.title}
            </label>
          </li>
        ))}
      </ul>

      <p data-testid="done-count">{doneCount}</p>

      <button data-testid="prev" disabled={loading || page <= 1}
        onClick={() => setPage((p) => p - 1)}>Prev</button>
      <span data-testid="page-info">Page {page} of {totalPages}</span>
      <button data-testid="next" disabled={loading || page >= totalPages}
        onClick={() => setPage((p) => p + 1)}>Next</button>

      {toast && <div data-testid="toast" role="status">{toast}</div>}
    </main>
  );
}
```

### Pembahasan

- **Race condition.** `api.js` tidak menerima `signal`, jadi flag `cancelled` di cleanup adalah pilihan tepat. Dengan `fetch` sungguhan, pakai `AbortController` karena request benar-benar dihentikan. Strict Mode aman: effect pertama dibatalkan cleanup, effect kedua yang menang.
- **Klik ganda.** `toggleTask` itu *flip*, bukan *set*. UI membalik di setiap klik, request per `id` diantre berurutan, dan rollback hanya membalik flip milik klik yang gagal (functional update membaca state terbaru). Hasil akhir UI = nilai awal XOR jumlah request sukses.
- **Page di luar batas.** Reset ke halaman 1 saat query berubah dan `Math.max(1, ...)` pada `totalPages`.

### Jawaban teori

1. Functional update membaca state terbaru. Versi `setItems(items.map(...))` memakai `items` dari closure render lama sehingga bisa menimpa update lain, terutama pada rollback yang datang beberapa detik kemudian.
2. `AbortController` menghentikan request sungguhan (hemat jaringan). Flag hanya mengabaikan hasil, request tetap berjalan. Flag cukup jika API tidak mendukung abort atau request murah.
3. `filter` pada 10 item itu murah. `useMemo` baru berguna untuk komputasi mahal atau saat butuh identitas stabil sebagai dependency/prop.

### Kelemahan solusi (pertanyaan interviewer)

- Jika pindah halaman saat toggle masih berjalan lalu rollback terjadi, flip bisa mengenai data hasil refetch. Solusi lebih kokoh: simpan `overrides` per id (`{ serverValue, pendingCount }`).
- TanStack Query (`useMutation` dengan `onMutate`/`onError`/`onSettled`) menangani pola ini bawaan.

---

# 1. JavaScript (Intermediate)

Target total sekitar 60 menit.

## Soal

### Soal 1: Ringkasan Pesanan (15 menit)

Tulis `summarizeOrders(orders)` yang mengelompokkan pesanan per `customer`.

```js
const orders = [
  { id: 1, customer: "Ani", amount: 120, status: "paid" },
  { id: 2, customer: "Budi", amount: 80, status: "pending" },
  { id: 3, customer: "Ani", amount: 50, status: "paid" },
  { id: 4, customer: "Ani", amount: 30, status: "cancelled" },
  { id: 5, customer: "Budi", amount: 200, status: "paid" },
];
```

Aturan:
- `cancelled` diabaikan sepenuhnya.
- Hasil: array `{ customer, totalPaid, pendingCount, orderIds }`.
- `totalPaid` hanya menjumlahkan status `paid`.
- Urutkan `totalPaid` menurun; jika sama, nama menaik.
- Jangan mutate input.

Output yang diharapkan:
```js
[
  { customer: "Budi", totalPaid: 200, pendingCount: 1, orderIds: [2, 5] },
  { customer: "Ani", totalPaid: 170, pendingCount: 0, orderIds: [1, 3] },
]
```

Edge case: array kosong, semua cancelled, customer tanpa `paid`, input tidak berubah.

### Soal 2: `memoizeAsync` dengan TTL (20 menit)

```js
function memoizeAsync(fn, { ttl = 1000, keyFn = (...args) => JSON.stringify(args) } = {}) {
  // TODO
}
```

1. Hasil di-cache per kunci sampai `ttl` ms.
2. Panggilan bersamaan dengan argumen sama saat promise pertama belum selesai berbagi **satu promise**, `fn` hanya dipanggil sekali.
3. Jika `fn` reject, error **tidak di-cache**.
4. Setelah `ttl` lewat, panggilan baru memanggil `fn` lagi.
5. Sediakan `memoized.clear()`.

```js
let calls = 0;
const slow = async (x) => { calls++; await new Promise(r => setTimeout(r, 50)); return x * 2; };
const m = memoizeAsync(slow, { ttl: 200 });

await Promise.all([m(2), m(2), m(2)]);
console.log(calls);          // 1
await m(2);
console.log(calls);          // 1
await new Promise(r => setTimeout(r, 250));
await m(2);
console.log(calls);          // 2
```

Pertimbangkan: TTL dihitung saat dipanggil atau saat promise selesai?

### Soal 3: `promisePool` (25 menit)

```js
async function promisePool(tasks, limit) {
  // tasks: array fungsi yang mengembalikan Promise
}
```

- Maksimum `limit` task berjalan bersamaan.
- Hasil **sesuai urutan `tasks`**.
- Reject menjadi `{ status: "rejected", reason }`, sukses `{ status: "fulfilled", value }`, task lain tetap lanjut.
- Task berikutnya mulai segera setelah ada slot kosong.
- `limit` > jumlah task tetap berfungsi; `tasks` kosong mengembalikan `[]`.

```js
const t = (ms, v, fail) => () =>
  new Promise((res, rej) => setTimeout(() => (fail ? rej(new Error(v)) : res(v)), ms));

const out = await promisePool([t(300, "a"), t(100, "b"), t(100, "c", true), t(50, "d")], 2);
// [{fulfilled a}, {fulfilled b}, {rejected Error("c")}, {fulfilled d}]
```

### Bonus: prediksi output

```js
// A
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i), 0);

// B
const obj = { n: 1, f() { return () => this.n; }, g: () => this.n };
console.log(obj.f()(), obj.g());

// C
console.log(1);
setTimeout(() => console.log(2), 0);
Promise.resolve().then(() => console.log(3));
console.log(4);

// D
console.log([10, 9, 1].sort());
console.log(typeof null, [] + [], 0.1 + 0.2 === 0.3);
```

## Kunci Jawaban

### Soal 1

```js
function summarizeOrders(orders) {
  const map = new Map();

  for (const o of orders) {
    if (o.status === "cancelled") continue;

    if (!map.has(o.customer)) {
      map.set(o.customer, {
        customer: o.customer,
        totalPaid: 0,
        pendingCount: 0,
        orderIds: [],
      });
    }

    const entry = map.get(o.customer);
    if (o.status === "paid") entry.totalPaid += o.amount;
    else if (o.status === "pending") entry.pendingCount++;
    entry.orderIds.push(o.id);
  }

  return [...map.values()].sort(
    (a, b) => b.totalPaid - a.totalPaid || a.customer.localeCompare(b.customer)
  );
}
```

- Objek baru dibuat di dalam fungsi, input tidak ter-mutate.
- `cancelled` di-`continue` di awal.
- Comparator memakai `||`: selisih `0` (falsy) beralih ke perbandingan nama. `b - a` untuk menurun.

### Soal 2

```js
function memoizeAsync(
  fn,
  { ttl = 1000, keyFn = (...args) => JSON.stringify(args) } = {}
) {
  const cache = new Map(); // key -> { promise, expiresAt }

  function memoized(...args) {
    const key = keyFn(...args);
    const entry = cache.get(key);

    if (entry && entry.expiresAt > Date.now()) return entry.promise;

    const record = { promise: null, expiresAt: Infinity }; // Infinity = in-flight
    record.promise = Promise.resolve()
      .then(() => fn(...args))
      .then(
        (value) => {
          if (cache.get(key) === record) record.expiresAt = Date.now() + ttl;
          return value;
        },
        (err) => {
          if (cache.get(key) === record) cache.delete(key);
          throw err;
        }
      );

    cache.set(key, record);
    return record.promise;
  }

  memoized.clear = () => cache.clear();
  return memoized;
}
```

- Yang di-cache adalah **promise**, sehingga panggilan bersamaan berbagi satu request.
- TTL dihitung **saat promise selesai**: jika dihitung saat dipanggil dan `fn` lambat, cache sudah kedaluwarsa saat hasil tiba. Memilih "saat dipanggil" juga sah bila bisa dijelaskan.
- Pada reject, entri dihapus dan error dilempar ulang.
- `cache.get(key) === record` melindungi dari `clear()` saat request berjalan.
- `Promise.resolve().then(() => fn(...))` menangkap throw sinkron.

### Soal 3

```js
async function promisePool(tasks, limit) {
  const results = new Array(tasks.length);
  let next = 0;

  async function worker() {
    while (next < tasks.length) {
      const i = next++; // ambil indeks sinkron, aman dari race
      try {
        results[i] = { status: "fulfilled", value: await tasks[i]() };
      } catch (reason) {
        results[i] = { status: "rejected", reason };
      }
    }
  }

  const workerCount = Math.min(limit, tasks.length);
  await Promise.all(Array.from({ length: workerCount }, worker));
  return results;
}
```

- Ada `limit` worker; tiap worker mengambil task berikutnya begitu selesai.
- Hasil ditulis ke `results[i]`, bukan `push`, sehingga urutan terjaga.
- try/catch di dalam worker: satu kegagalan tidak menghentikan yang lain.
- Jika `limit <= 0`, tidak ada worker; paksa `limit >= 1` atau lempar error.

### Bonus

| Soal | Output | Alasan |
|---|---|---|
| A | `3 3 3` | `var` function-scoped; saat callback jalan `i = 3`. Dengan `let` hasilnya `0 1 2`. |
| B | `1 undefined` | Arrow di `f()` mewarisi `this` = `obj`. `g` arrow di level modul, `this` bukan `obj`. |
| C | `1 4 3 2` | Sinkron dulu, lalu microtask (promise), lalu macrotask (`setTimeout`). |
| D | `[1, 10, 9]`, lalu `object  false` | `sort()` default membandingkan string. `typeof null` = `"object"`. `[] + []` = `""`. `0.1 + 0.2` = `0.30000000000000004`. |

---

# 2. CSS

Target total sekitar 45 menit. Buka di browser dan ubah ukuran jendela.

## Soal

### Soal 1: Kartu Produk Responsif (15 menit)

HTML (jangan diubah, hanya tulis CSS):

```html
<section class="grid">
  <article class="card">
    <img class="thumb" src="https://picsum.photos/300/200" alt="" />
    <div class="body">
      <h3 class="title">Judul produk yang cukup panjang sampai bisa terpotong</h3>
      <p class="desc">Deskripsi singkat produk. Panjangnya bisa berbeda-beda antar kartu.</p>
      <span class="badge">Baru</span>
    </div>
    <footer class="actions">
      <button>Detail</button>
      <button class="primary">Beli</button>
    </footer>
  </article>
  <!-- ulangi 6 kartu dengan panjang teks berbeda -->
</section>
```

- >= 900px: 3 kolom; 600-899px: 2 kolom; < 600px: 1 kolom (CSS Grid).
- Jarak antar kartu `16px`, tanpa margin hack.
- Semua kartu satu baris tingginya sama; `.actions` menempel di dasar kartu walau deskripsi pendek.
- `.title` maksimal 2 baris dengan `...`.
- Gambar mengisi lebar kartu rasio 3:2, tidak gepeng/melar.
- `.badge` di pojok kanan atas (absolute).

### Soal 2: Layout Halaman Klasik (15 menit)

```html
<div class="page">
  <header>Header</header>
  <nav>Nav</nav>
  <main>Main (konten bisa panjang)</main>
  <aside>Aside</aside>
  <footer>Footer</footer>
</div>
```

- Desktop: header penuh; `nav` (200px) | `main` (sisa) | `aside` (240px); footer penuh.
- Footer selalu di dasar viewport walau konten sedikit, tanpa `position: fixed`.
- Di bawah 768px: urutan header, main, nav, aside, footer, satu kolom, tanpa mengubah HTML.
- `header` sticky di atas.
- `nav` dan `aside` tidak menyusut saat `main` berisi kata sangat panjang.

### Soal 3: Komponen Kecil (15 menit)

a) Kotak `.box` (200x100px) tepat di tengah layar, **dua cara berbeda**.

b) Tooltip murni CSS: saat hover atau fokus keyboard, teks dari `data-label` muncul di atas elemen dengan segitiga penunjuk.

```html
<button class="tip" data-label="Simpan perubahan">Simpan</button>
```

c) `.btn` dengan transisi 150ms untuk `:hover`, `:active` (sedikit mengecil), `:focus-visible` (outline jelas), `:disabled` (pudar, `not-allowed`).

### Bonus

```css
/* A: lebar total .a tanpa box-sizing diatur? Dan dengan border-box? */
.a { width: 200px; padding: 20px; border: 5px solid; }

/* B: warna teks? <p id="x"><span class="y z">Teks</span></p> */
#x .y { color: red; }
.y.z   { color: blue; }
.y     { color: green !important; }

/* C: gunanya min-width: 0? */
.parent { display: flex; }
.child  { flex: 1 1 0; min-width: 0; }

/* D: jarak vertikal .m1 dan .m2 bersebelahan? */
.m1 { margin-bottom: 30px; }
.m2 { margin-top: 20px; }
```

## Kunci Jawaban

### Soal 1

```css
*, *::before, *::after { box-sizing: border-box; }

.grid { display: grid; gap: 16px; grid-template-columns: 1fr; }
@media (min-width: 600px) { .grid { grid-template-columns: repeat(2, 1fr); } }
@media (min-width: 900px) { .grid { grid-template-columns: repeat(3, 1fr); } }

.card {
  position: relative;            /* induk untuk .badge */
  display: flex;
  flex-direction: column;
  overflow: hidden;
  border: 1px solid #ddd;
  border-radius: 8px;
}
.thumb { display: block; width: 100%; aspect-ratio: 3 / 2; object-fit: cover; }
.body  { flex: 1; padding: 12px; }   /* mendorong .actions ke dasar */
.title {
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 2;
  line-clamp: 2;
  overflow: hidden;
}
.badge   { position: absolute; top: 8px; right: 8px; }
.actions { display: flex; gap: 8px; padding: 12px; }
```

- Tinggi kartu sama karena item grid default `align-items: stretch`.
- Tombol di dasar karena `.body { flex: 1 }`.
- `object-fit: cover` mencegah gambar gepeng.

### Soal 2

```css
body { margin: 0; }
.page {
  min-height: 100dvh;
  display: grid;
  grid-template-columns: 200px minmax(0, 1fr) 240px;
  grid-template-rows: auto 1fr auto;
  grid-template-areas:
    "header header header"
    "nav    main   aside"
    "footer footer footer";
}
header { grid-area: header; position: sticky; top: 0; z-index: 10; background: #fff; }
nav    { grid-area: nav; }
main   { grid-area: main; min-width: 0; overflow-wrap: anywhere; }
aside  { grid-area: aside; }
footer { grid-area: footer; }

@media (max-width: 767px) {
  .page {
    grid-template-columns: minmax(0, 1fr);
    grid-template-rows: auto 1fr auto auto auto;
    grid-template-areas: "header" "main" "nav" "aside" "footer";
  }
}
```

- Baris tengah `1fr` + `min-height: 100dvh` membuat footer di dasar.
- `minmax(0, 1fr)` dan `min-width: 0` mencegah kata panjang melebarkan `main`.
- Urutan mobile berubah lewat `grid-template-areas`, tanpa mengubah HTML.

### Soal 3

```css
/* a) cara 1: flex */
body { margin: 0; min-height: 100vh; display: flex; justify-content: center; align-items: center; }
/* cara 2: grid */
body { margin: 0; min-height: 100vh; display: grid; place-items: center; }
.box { width: 200px; height: 100px; }

/* b) Tooltip */
.tip { position: relative; }
.tip::after, .tip::before {
  position: absolute; left: 50%; transform: translateX(-50%);
  opacity: 0; visibility: hidden; pointer-events: none;
  transition: opacity 150ms;
}
.tip::after {
  content: attr(data-label);
  bottom: calc(100% + 8px);
  padding: 4px 8px; border-radius: 4px;
  background: #222; color: #fff; white-space: nowrap;
}
.tip::before {
  content: "";
  bottom: calc(100% + 2px);
  border: 6px solid transparent; border-top-color: #222;
}
.tip:hover::after, .tip:hover::before,
.tip:focus-visible::after, .tip:focus-visible::before { opacity: 1; visibility: visible; }

/* c) Tombol */
.btn { transition: transform 150ms, background-color 150ms, opacity 150ms; }
.btn:hover:not(:disabled)  { background: #1d4ed8; }
.btn:active:not(:disabled) { transform: scale(0.97); }
.btn:focus-visible         { outline: 3px solid #2563eb; outline-offset: 2px; }
.btn:disabled              { opacity: 0.5; cursor: not-allowed; }
```

### Bonus

| | Jawaban |
|---|---|
| A | `content-box`: 200 + 40 + 10 = **250px**. `border-box`: **200px**. |
| B | **Hijau**. `!important` mengalahkan spesifisitas apa pun. |
| C | Item flex punya `min-width: auto` sehingga tidak bisa lebih kecil dari kontennya. `min-width: 0` mengizinkannya menyusut. |
| D | **30px**: margin vertikal bersebelahan *collapse* ke yang terbesar. |

---

# 3. React (Intermediate)

Target total sekitar 60 menit. Tes asli: 2 soal React Intermediate + 1 React Basic. Test otomatis biasanya memakai React Testing Library, jadi `data-testid`, teks, dan `role` harus persis.

## Soal

### Soal 1: Autocomplete dengan Navigasi Keyboard (25 menit)

Komponen `<Autocomplete options={string[]} onSelect={(value) => void} />`.

- Input `data-testid="ac-input"`. Saat mengetik, tampilkan `ul` (`data-testid="ac-list"`) berisi opsi yang **mengandung** teks (tidak peka huruf besar/kecil), maksimal **5 item**.
- Tiap `li` punya `data-testid="ac-option-{index}"`, bagian cocok ditebalkan dengan `<strong>`.
- Keyboard: `ArrowDown`/`ArrowUp` memindahkan item aktif (bergulir), `Enter` memilih item aktif (isi input, panggil `onSelect`, tutup), `Escape` menutup.
- Item aktif punya `aria-selected="true"` dan class `active`. Hover mouse juga mengaktifkan item.
- Klik item memilihnya. Klik di luar komponen menutup daftar.
- Tidak ada hasil: `data-testid="ac-empty"` berisi `No results`.
- ARIA: `role="combobox"` pada input, `role="listbox"` pada `ul`, `role="option"` pada `li`.

Jebakan: reset item aktif saat teks berubah, bersihkan listener klik luar, urutan `blur` vs `click`, jangan simpan hasil filter di state.

### Soal 2: Formulir Dua Langkah dengan Draf Tersimpan (20 menit)

- Langkah 1: `name` (wajib, min 3), `email` (wajib, valid). Langkah 2: `password` (min 8, ada angka), `confirm` (sama dengan password).
- Tombol `Next`/`Back`/`Submit` (`data-testid="next"`, `"back"`, `"submit"`).
- Error di bawah field (`data-testid="error-{field}"`) hanya muncul setelah field disentuh (blur) atau setelah Next/Submit ditekan.
- `Next` tidak berpindah jika ada error. `Back` tidak menghilangkan nilai.
- Submit sukses: `data-testid="success"` berisi `Welcome, {name}!` dan hapus draf.
- `name` dan `email` disimpan di `localStorage` (key `draft`) dan dipulihkan saat reload. **Password tidak boleh disimpan.**
- Wajib: custom hook `useLocalStorage(key, initialValue)` dan fungsi murni `validate(values)`.

Jebakan: `JSON.parse` pada data rusak (try/catch), error adalah data turunan, jangan simpan password.

### Soal 3: Perbaiki Komponen yang Rusak (15 menit)

Minimal 7 bug. Daftar bug + alasan, versi perbaikan, dan satu kalimat cara menemukannya.

```jsx
import { useState, useEffect } from "react";

function Cart({ initialItems }) {
  const [items, setItems] = useState(initialItems);
  const [total, setTotal] = useState(0);
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    setTotal(items.reduce((s, i) => s + i.price * i.qty, 0));
  });

  useEffect(() => {
    setInterval(() => setSeconds(seconds + 1), 1000);
  }, []);

  const inc = (id) => {
    const item = items.find((i) => i.id === id);
    item.qty++;
    setItems(items);
  };

  const remove = (id) => setItems(items.filter((i) => (i.id = id)));

  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>
          {item.name} x {item.qty}
          <button onClick={inc(item.id)}>+</button>
          <button onClick={() => remove(item.id)}>Remove</button>
        </li>
      ))}
      <p>Total: {total}</p>
      <p>Session: {seconds}s</p>
    </ul>
  );
}
```

### Bonus konsep

1. Mengapa `useEffect(() => {...})` tanpa dependency array hampir selalu salah untuk `setState` di dalamnya?
2. Beda `useRef` dan `useState`?
3. Mengapa `key={index}` bermasalah pada list yang bisa dihapus/diurutkan?
4. Kapan `useCallback` benar-benar berguna?
5. Apa yang terjadi pada `useEffect` di Strict Mode?

## Kunci Jawaban

### Soal 1: Autocomplete

```jsx
import { useState, useRef, useEffect, useMemo } from "react";

function highlight(text, q) {
  const i = text.toLowerCase().indexOf(q.toLowerCase());
  if (!q || i === -1) return text;
  return (
    <>
      {text.slice(0, i)}
      <strong>{text.slice(i, i + q.length)}</strong>
      {text.slice(i + q.length)}
    </>
  );
}

export default function Autocomplete({ options, onSelect }) {
  const [query, setQuery] = useState("");
  const [open, setOpen] = useState(false);
  const [active, setActive] = useState(-1);
  const rootRef = useRef(null);

  const q = query.trim();
  const results = useMemo(
    () => (q ? options.filter((o) => o.toLowerCase().includes(q.toLowerCase())).slice(0, 5) : []),
    [q, options]                                  // turunan, bukan state
  );

  useEffect(() => {                               // klik luar
    const onDown = (e) => {
      if (!rootRef.current?.contains(e.target)) setOpen(false);
    };
    document.addEventListener("mousedown", onDown);
    return () => document.removeEventListener("mousedown", onDown);
  }, []);

  const select = (value) => {
    setQuery(value); setOpen(false); setActive(-1); onSelect(value);
  };

  const onKeyDown = (e) => {
    const n = results.length;
    if (e.key === "ArrowDown") {
      e.preventDefault(); setOpen(true);
      if (n) setActive((i) => (i + 1) % n);
    } else if (e.key === "ArrowUp") {
      e.preventDefault(); setOpen(true);
      if (n) setActive((i) => (i <= 0 ? n - 1 : i - 1));
    } else if (e.key === "Enter") {
      if (open && results[active]) { e.preventDefault(); select(results[active]); }
    } else if (e.key === "Escape") {
      setOpen(false);
    }
  };

  return (
    <div ref={rootRef}>
      <input
        data-testid="ac-input"
        role="combobox"
        aria-expanded={open && q !== ""}
        aria-controls="ac-list"
        aria-autocomplete="list"
        aria-activedescendant={active >= 0 ? `ac-opt-${active}` : undefined}
        value={query}
        onChange={(e) => { setQuery(e.target.value); setActive(-1); setOpen(true); }}
        onKeyDown={onKeyDown}
      />
      {open && results.length > 0 && (
        <ul id="ac-list" role="listbox" data-testid="ac-list">
          {results.map((o, i) => (
            <li
              key={`${i}-${o}`}
              id={`ac-opt-${i}`}
              role="option"
              aria-selected={i === active}
              className={i === active ? "active" : undefined}
              data-testid={`ac-option-${i}`}
              onMouseEnter={() => setActive(i)}
              onClick={() => select(o)}
            >
              {highlight(o, q)}
            </li>
          ))}
        </ul>
      )}
      {open && q && results.length === 0 && <p data-testid="ac-empty">No results</p>}
    </div>
  );
}
```

- Klik luar memakai `mousedown` di `document` + `contains()`; klik item ada di dalam root sehingga tidak dianggap klik luar (menghindari masalah `blur` lebih dulu daripada `click`).
- `results` diturunkan, `active` di-reset di handler `onChange`, listener dibersihkan di cleanup.

### Soal 2: Form Dua Langkah

```jsx
import { useState, useEffect, useCallback } from "react";

const STEP_FIELDS = [["name", "email"], ["password", "confirm"]];

export function validate(v) {
  const e = {};
  const name = v.name.trim();
  if (!name) e.name = "Name is required";
  else if (name.length < 3) e.name = "Min 3 characters";
  if (!v.email.trim()) e.email = "Email is required";
  else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v.email)) e.email = "Invalid email";
  if (v.password.length < 8) e.password = "Min 8 characters";
  else if (!/\d/.test(v.password)) e.password = "Must contain a number";
  if (v.confirm !== v.password) e.confirm = "Passwords do not match";
  return e;
}

export function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    try {
      const raw = localStorage.getItem(key);
      return raw ? JSON.parse(raw) : initial;
    } catch {
      return initial;                              // data rusak
    }
  });
  useEffect(() => {
    try { localStorage.setItem(key, JSON.stringify(value)); } catch {}
  }, [key, value]);
  const remove = useCallback(() => {
    try { localStorage.removeItem(key); } catch {}
  }, [key]);
  return [value, setValue, remove];
}

export default function SignupForm() {
  const [draft, setDraft, clearDraft] = useLocalStorage("draft", { name: "", email: "" });
  const [secret, setSecret] = useState({ password: "", confirm: "" }); // tidak disimpan
  const [step, setStep] = useState(0);
  const [touched, setTouched] = useState({});
  const [welcome, setWelcome] = useState(null);

  const values = { ...draft, ...secret };
  const errors = validate(values);                 // turunan
  const stepHasError = (s) => STEP_FIELDS[s].some((f) => errors[f]);
  const touch = (fields) =>
    setTouched((t) => ({ ...t, ...Object.fromEntries(fields.map((f) => [f, true])) }));

  const onChange = (e) => {
    const { name, value } = e.target;
    if (name in draft) setDraft((d) => ({ ...d, [name]: value }));
    else setSecret((s) => ({ ...s, [name]: value }));
  };

  const next = () => {
    touch(STEP_FIELDS[0]);
    if (!stepHasError(0)) setStep(1);
  };
  const submit = (e) => {
    e.preventDefault();
    touch(STEP_FIELDS[1]);
    if (stepHasError(0) || stepHasError(1)) return;
    setWelcome(values.name.trim());
    clearDraft();
  };

  if (welcome) return <p data-testid="success">Welcome, {welcome}!</p>;

  const field = (name, label, type = "text") => (
    <div>
      <label htmlFor={name}>{label}</label>
      <input id={name} name={name} type={type} value={values[name]}
        onChange={onChange} onBlur={() => touch([name])}
        aria-invalid={touched[name] && !!errors[name]} />
      {touched[name] && errors[name] && (
        <span role="alert" data-testid={`error-${name}`}>{errors[name]}</span>
      )}
    </div>
  );

  return (
    <form onSubmit={submit} noValidate>
      {step === 0 ? (
        <>
          {field("name", "Name")}
          {field("email", "Email", "email")}
          <button type="button" data-testid="next" onClick={next}>Next</button>
        </>
      ) : (
        <>
          {field("password", "Password", "password")}
          {field("confirm", "Confirm", "password")}
          <button type="button" data-testid="back" onClick={() => setStep(0)}>Back</button>
          <button type="submit" data-testid="submit">Submit</button>
        </>
      )}
    </form>
  );
}
```

- Error = `validate(values)` dan hanya tampil jika `touched`.
- Password dan confirm di state terpisah sehingga tidak pernah masuk `localStorage`.
- `Back` tidak menghilangkan nilai karena state ada di komponen induk.

### Soal 3: Bug pada `Cart`

| # | Bug | Alasan |
|---|---|---|
| 1 | `useEffect` tanpa dependency memanggil `setTotal` | Berjalan tiap render, render ekstra. `total` seharusnya data turunan. |
| 2 | `setInterval` tanpa cleanup | Memory leak, ganda di Strict Mode. |
| 3 | `setSeconds(seconds + 1)` | Stale closure: `seconds` selalu `0`, macet di 1. Pakai `s => s + 1`. |
| 4 | `item.qty++` | Mutasi objek state langsung. |
| 5 | `setItems(items)` | Referensi sama, React tidak re-render. |
| 6 | `i.id = id` di `filter` | Penugasan, bukan perbandingan. Seharusnya `i.id !== id`. |
| 7 | `onClick={inc(item.id)}` | Fungsi dipanggil saat render, bukan saat klik. |
| 8 | `key={index}` | State/fokus bisa tertukar saat item dihapus. Pakai `item.id`. |
| 9 | `<p>` langsung di dalam `<ul>` | HTML tidak valid. |

```jsx
import { useState, useEffect } from "react";

function Cart({ initialItems }) {
  const [items, setItems] = useState(initialItems);
  const [seconds, setSeconds] = useState(0);

  const total = items.reduce((s, i) => s + i.price * i.qty, 0);   // turunan

  useEffect(() => {
    const id = setInterval(() => setSeconds((s) => s + 1), 1000);
    return () => clearInterval(id);
  }, []);

  const inc = (id) =>
    setItems((prev) => prev.map((i) => (i.id === id ? { ...i, qty: i.qty + 1 } : i)));

  const remove = (id) => setItems((prev) => prev.filter((i) => i.id !== id));

  return (
    <div>
      <ul>
        {items.map((item) => (
          <li key={item.id}>
            {item.name} x {item.qty}
            <button onClick={() => inc(item.id)}>+</button>
            <button onClick={() => remove(item.id)}>Remove</button>
          </li>
        ))}
      </ul>
      <p>Total: {total}</p>
      <p>Session: {seconds}s</p>
    </div>
  );
}
```

Cara menemukan: baca atas ke bawah dengan tiga pertanyaan per baris: ada mutasi? ada efek samping tanpa cleanup? ada nilai yang bisa dihitung dari data lain?

### Jawaban konsep

1. Effect berjalan tiap render dan `setState` di dalamnya memicu render lagi: loop atau render sia-sia.
2. `useRef` menyimpan nilai mutable tanpa memicu render; `useState` memicu render saat berubah.
3. React mencocokkan elemen lewat `key`; saat urutan berubah, index bergeser sehingga state (isi input, fokus) menempel pada item yang salah.
4. Berguna saat fungsi diteruskan ke komponen `React.memo` atau jadi dependency effect. Selain itu hanya menambah kerumitan.
5. Strict Mode menjalankan mount, unmount, mount lagi (hanya development) untuk membongkar effect tanpa cleanup benar.

---

# 4. TypeScript (Intermediate)

Target total sekitar 45 menit. Gunakan `strict: true` (atau TypeScript Playground), **tanpa `any`**.

## Soal

### Soal 1: Utility Types (15 menit)

```ts
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  role: "admin" | "editor" | "viewer";
  profile?: { bio: string; avatarUrl: string };
  createdAt: Date;
}
```

Definisikan **hanya dengan utility types dan operator tipe**:

1. `PublicUser`: `User` tanpa `password`.
2. `CreateUserInput`: `User` tanpa `id` dan `createdAt`.
3. `UpdateUserInput`: semua field `CreateUserInput` opsional, **kecuali** `email` tetap wajib.
4. `UserRoleMap`: kunci tiap role, nilai `User[]`.
5. `StringKeys<T>`: gabungan nama properti `T` yang bertipe `string`. `StringKeys<User>` = `"name" | "email" | "password"`.
6. `DeepReadonly<T>`: semua properti bersarang menjadi `readonly`.

Uji dengan `// @ts-expect-error`. Jika `ts-expect-error` Anda ikut error, tipenya terlalu longgar.

### Soal 2: Generic Functions (15 menit)

```ts
function groupBy<T, K extends PropertyKey>(items: T[], keyFn: (item: T) => K): Record<K, T[]> { /* TODO */ }

function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> { /* TODO */ }

function mapValues<T extends object, R>(obj: T, fn: (value: T[keyof T], key: keyof T) => R): Record<keyof T, R> { /* TODO */ }

type Events = {
  login: { userId: number };
  logout: undefined;
  error: { message: string };
};

class Emitter<E extends Record<string, unknown>> {
  on<K extends keyof E>(event: K, handler: (payload: E[K]) => void): void { /* TODO */ }
  emit<K extends keyof E>(event: K, payload: E[K]): void { /* TODO */ }
}
```

Harus terpenuhi: `groupBy([1,2,3], n => n % 2 === 0 ? "even" : "odd")` bertipe `Record<"even" | "odd", number[]>`; `pick(user, ["name", "foo"])` error; `emitter.emit("login", { userId: "1" })` error; `emitter.on("error", p => p.message)` terinferensi.

### Soal 3: Union, Narrowing, dan Props React (15 menit)

a) Discriminated union + exhaustive check:

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rect"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number };

function area(s: Shape): number { /* switch + exhaustive check */ }
```

b) Type guard dan Result:

```ts
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

async function safeFetch<T>(url: string, isT: (data: unknown) => data is T): Promise<Result<T>> { /* TODO */ }
```

Buat juga `isUser(data: unknown): data is { id: number; name: string }` tanpa `any` dan tanpa `as`.

c) Props React: `Button` dengan `variant === "link"` mewajibkan `href` dan melarang `onClick`; `variant === "button"` kebalikannya. `List<T>` generik dengan tipe item terinferensi dari `items`.

### Bonus

```ts
// A: tipe x, y, Z?
const x = ["a", "b"] as const;
const y = { a: 1, b: "s" } satisfies Record<string, number | string>;
type Z = (typeof x)[number];

// B: error atau tidak?
function f(v: string | number) { return v.toUpperCase(); }

// C: beda?
let a: unknown = 5;  a.toFixed();
let b: any = 5;      b.toFixed();

// D: hasil tipe?
type R = ReturnType<typeof setTimeout>;
type P = Parameters<(a: string, b?: number) => void>;
type N = NonNullable<string | null | undefined>;

// E: mengapa error?
const user = { name: "Ani" };
user.age = 20;
```

Konsep: (1) `interface` vs `type`, (2) `unknown` vs `any` vs `never`, (3) mengapa `as` berbahaya, (4) type narrowing dan 4 caranya, (5) fungsi `keyof`, `typeof`, `T[K]`.

## Kunci Jawaban

### Soal 1

```ts
type PublicUser = Omit<User, "password">;
type CreateUserInput = Omit<User, "id" | "createdAt">;
type UpdateUserInput = Partial<Omit<CreateUserInput, "email">> & Pick<CreateUserInput, "email">;
type UserRoleMap = Record<User["role"], User[]>;

type StringKeys<T> = {
  [K in keyof T]-?: string extends T[K] ? K : never;
}[keyof T];

type DeepReadonly<T> =
  T extends Date | ((...args: never[]) => unknown) ? T
  : T extends readonly (infer U)[] ? readonly DeepReadonly<U>[]
  : T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;
```

**Koreksi soal:** `T[K] extends string` ikut memuat `"role"` karena union literal juga turunan `string`. Hasil `"name" | "email" | "password"` hanya didapat dengan `string extends T[K]`.

```ts
declare const pu: PublicUser;
// @ts-expect-error
pu.password;
const u: UpdateUserInput = { email: "a@b.c" };        // OK
// @ts-expect-error
const bad: UpdateUserInput = { name: "Ani" };          // email wajib
type T1 = StringKeys<User>;                            // "name" | "email" | "password"
```

### Soal 2

```ts
function groupBy<T, K extends PropertyKey>(items: T[], keyFn: (item: T) => K): Record<K, T[]> {
  const result = {} as Record<K, T[]>;
  for (const item of items) {
    const k = keyFn(item);
    (result[k] ??= []).push(item);
  }
  return result;
}

function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  for (const k of keys) result[k] = obj[k];
  return result;
}

function mapValues<T extends object, R>(
  obj: T,
  fn: (value: T[keyof T], key: keyof T) => R
): Record<keyof T, R> {
  const result = {} as Record<keyof T, R>;
  for (const key of Object.keys(obj) as (keyof T)[]) {
    result[key] = fn(obj[key], key);
  }
  return result;
}

class Emitter<E extends Record<string, unknown>> {
  private handlers = new Map<keyof E, Array<(payload: unknown) => void>>();

  on<K extends keyof E>(event: K, handler: (payload: E[K]) => void): void {
    const list = this.handlers.get(event) ?? [];
    list.push(handler as (payload: unknown) => void);  // satu-satunya assertion, di batas internal
    this.handlers.set(event, list);
  }

  emit<K extends keyof E>(event: K, payload: E[K]): void {
    this.handlers.get(event)?.forEach((h) => h(payload));
  }
}
```

- `K extends PropertyKey` membuat literal `"even" | "odd"` terinferensi. Catatan: `Record<K, T[]>` mengklaim semua kunci ada; `Partial<Record<K, T[]>>` lebih akurat.
- `as (keyof T)[]` pada `Object.keys` aman jika kunci memang dari `obj`, tapi tidak sepenuhnya aman karena *structural typing*. Jelaskan alasannya.
- Assertion di `on` dibatasi di satu tempat; sisi publik tetap type-safe.

### Soal 3

```ts
function assertNever(x: never): never {
  throw new Error(`Unexpected: ${JSON.stringify(x)}`);
}

function area(s: Shape): number {
  switch (s.kind) {
    case "circle":   return Math.PI * s.radius ** 2;
    case "rect":     return s.width * s.height;
    case "triangle": return (s.base * s.height) / 2;
    default:         return assertNever(s);   // error kompilasi jika ada Shape baru
  }
}

async function safeFetch<T>(
  url: string,
  isT: (data: unknown) => data is T
): Promise<Result<T>> {
  try {
    const res = await fetch(url);
    if (!res.ok) return { ok: false, error: new Error(`HTTP ${res.status}`) };
    const data: unknown = await res.json();
    if (!isT(data)) return { ok: false, error: new Error("Invalid response shape") };
    return { ok: true, value: data };
  } catch (e) {
    return { ok: false, error: e instanceof Error ? e : new Error(String(e)) };
  }
}

function isRecord(v: unknown): v is Record<string, unknown> {
  return typeof v === "object" && v !== null;
}
function isUser(d: unknown): d is { id: number; name: string } {
  return isRecord(d) && typeof d.id === "number" && typeof d.name === "string";
}
```

```tsx
type ButtonProps =
  | { variant: "link"; href: string; onClick?: never; children: React.ReactNode }
  | { variant: "button"; onClick: () => void; href?: never; children: React.ReactNode };

function Button(props: ButtonProps) {
  return props.variant === "link"
    ? <a href={props.href}>{props.children}</a>
    : <button onClick={props.onClick}>{props.children}</button>;
}

function List<T>({ items, keyOf, render }: {
  items: T[];
  keyOf: (item: T) => string | number;
  render: (item: T) => React.ReactNode;
}) {
  return <ul>{items.map((i) => <li key={keyOf(i)}>{render(i)}</li>)}</ul>;
}
```

### Bonus

| | Jawaban |
|---|---|
| A | `x`: `readonly ["a", "b"]`; `y`: `{ a: number; b: string }`; `Z`: `"a" \| "b"` |
| B | **Error**: `toUpperCase` tidak ada pada `number`; perlu narrowing. |
| C | `unknown`: `a.toFixed()` error sampai dipersempit. `any`: lolos tanpa pengecekan. |
| D | `R`: `NodeJS.Timeout` (Node) atau `number` (DOM); `P`: `[a: string, b?: number]`; `N`: `string` |
| E | Properti `age` tidak ada pada tipe `{ name: string }`. |

### Jawaban konsep

1. `interface` bisa di-merge dan cocok untuk kontrak objek/kelas; `type` bisa untuk union, tuple, tipe kondisional.
2. `unknown` = tipe apa saja yang harus dicek dulu; `any` mematikan pengecekan; `never` = tipe tanpa nilai.
3. `as` memaksa kompiler percaya tanpa bukti. Alternatif: type guard, narrowing, `satisfies`, validasi runtime.
4. Narrowing: `typeof`, `instanceof`, `in`, pengecekan diskriminan (`kind ===`), truthiness, fungsi type guard.
5. `keyof T` = gabungan kunci; `typeof x` = tipe dari nilai; `T[K]` = tipe properti `K` pada `T`.

---

# 5. HTML/CSS/JS (Vanilla)

Target total sekitar 60 menit. Tanpa library. Jangan ubah id/class/struktur HTML yang disediakan.

## Soal

### Soal 1: Todo List dengan Event Delegation (25 menit)

```html
<form id="todo-form">
  <input id="todo-input" type="text" placeholder="Tambah tugas" />
  <button type="submit">Tambah</button>
</form>
<div id="filters">
  <button data-filter="all" class="active">Semua</button>
  <button data-filter="active">Aktif</button>
  <button data-filter="done">Selesai</button>
</div>
<ul id="todo-list"></ul>
<p id="counter">0 tugas tersisa</p>
```

- Submit menambah `li` (`data-id` unik, teks, checkbox, tombol `.delete`). Input kosong/spasi ditolak; setelah berhasil input dikosongkan dan difokuskan.
- Event delegation pada `#todo-list`: checkbox mencoret teks (class `done`), `.delete` menghapus.
- Klik ganda teks membuka mode edit inline. `Enter` simpan, `Escape` batal, teks kosong menghapus item.
- Filter all/active/done, class `active` pindah ke tombol yang diklik.
- `#counter`: `N tugas tersisa` (belum selesai).
- Data di array JS dan UI dirender dari data (`render()`).
- Teks `<img src=x onerror=alert(1)>` tampil sebagai teks biasa.

### Soal 2: Tabs dan Accordion Aksesibel (20 menit)

```html
<div id="tabs">
  <div role="tablist">
    <button role="tab" id="tab-1" aria-controls="panel-1">Profil</button>
    <button role="tab" id="tab-2" aria-controls="panel-2">Pesanan</button>
    <button role="tab" id="tab-3" aria-controls="panel-3">Pengaturan</button>
  </div>
  <div role="tabpanel" id="panel-1" aria-labelledby="tab-1">Isi profil</div>
  <div role="tabpanel" id="panel-2" aria-labelledby="tab-2">Isi pesanan</div>
  <div role="tabpanel" id="panel-3" aria-labelledby="tab-3">Isi pengaturan</div>
</div>
```

Tabs: satu panel terlihat; `aria-selected` benar; panel tersembunyi pakai atribut `hidden`; keyboard `ArrowRight/Left` (bergulir), `Home/End`; roving `tabindex`; tab aktif tersimpan di hash URL dan dipulihkan saat reload/hash berubah.

Accordion (buat sendiri, min 3 item): header `<button>` dengan `aria-expanded`, mode single-open, animasi tinggi halus.

### Soal 3: Pencarian Live dengan Fetch + Modal (15 menit)

Gunakan `https://jsonplaceholder.typicode.com/users`.

- `#search` debounce 300ms, filter nama (case-insensitive), render kartu di `#results`.
- State `Loading...`, `Error` + tombol `Retry`, `No results`.
- Race condition ditangani dengan `AbortController`.
- Klik kartu membuka modal (email, telepon, kota). Tutup dengan `×`, klik latar, dan `Escape`.
- Fokus pindah ke modal, kembali ke kartu saat ditutup, scroll `body` terkunci saat terbuka.

### Bonus

```html
<div id="outer"><button id="inner">Klik</button></div>
<script>
  outer.addEventListener("click", () => console.log("outer"));
  inner.addEventListener("click", (e) => { console.log("inner"); e.stopPropagation(); });
  outer.addEventListener("click", () => console.log("outer-capture"), true);
</script>
```

1. Urutan log? Apa yang berubah tanpa `stopPropagation()`?
2. `e.target` vs `e.currentTarget`?
3. `querySelectorAll` vs `getElementsByClassName`?
4. `textContent` vs `innerText` vs `innerHTML`?
5. `DOMContentLoaded` vs `load`, efek `defer`?
6. Fungsi `closest(selector)`?

## Kunci Jawaban

### Soal 1: Todo

```js
const form = document.getElementById("todo-form");
const input = document.getElementById("todo-input");
const list = document.getElementById("todo-list");
const counter = document.getElementById("counter");
const filters = document.getElementById("filters");

let todos = [];
let filter = "all";
let editingId = null;
let nextId = 1;

function render() {
  list.replaceChildren();
  const visible = todos.filter((t) =>
    filter === "all" ? true : filter === "active" ? !t.done : t.done
  );

  for (const t of visible) {
    const li = document.createElement("li");
    li.dataset.id = t.id;
    li.classList.toggle("done", t.done);

    const cb = document.createElement("input");
    cb.type = "checkbox";
    cb.checked = t.done;
    li.append(cb);

    if (editingId === t.id) {
      const edit = document.createElement("input");
      edit.className = "edit";
      edit.value = t.text;
      li.append(edit);
      list.append(li);
      edit.focus();
      continue;
    }

    const span = document.createElement("span");
    span.className = "text";
    span.textContent = t.text;                       // aman dari XSS

    const del = document.createElement("button");
    del.className = "delete";
    del.textContent = "Hapus";
    li.append(span, del);
    list.append(li);
  }

  const left = todos.filter((t) => !t.done).length;  // dari data, bukan DOM
  counter.textContent = `${left} tugas tersisa`;
}

const idOf = (el) => Number(el.closest("li")?.dataset.id);

form.addEventListener("submit", (e) => {
  e.preventDefault();
  const text = input.value.trim();
  if (!text) return;
  todos.push({ id: nextId++, text, done: false });
  input.value = "";
  input.focus();
  render();
});

// Event delegation: satu listener per jenis event pada <ul>
list.addEventListener("click", (e) => {
  const id = idOf(e.target);
  if (!id) return;
  if (e.target.matches("input[type=checkbox]")) {
    todos = todos.map((t) => (t.id === id ? { ...t, done: e.target.checked } : t));
    render();
  } else if (e.target.closest(".delete")) {
    todos = todos.filter((t) => t.id !== id);
    render();
  }
});

list.addEventListener("dblclick", (e) => {
  if (!e.target.matches(".text")) return;
  editingId = idOf(e.target);
  render();
});

list.addEventListener("keydown", (e) => {
  if (!e.target.matches(".edit")) return;
  const id = idOf(e.target);
  if (e.key === "Enter") {
    const text = e.target.value.trim();
    todos = text
      ? todos.map((t) => (t.id === id ? { ...t, text } : t))
      : todos.filter((t) => t.id !== id);
    editingId = null;
    render();
  } else if (e.key === "Escape") {
    editingId = null;
    render();
  }
});

filters.addEventListener("click", (e) => {
  const btn = e.target.closest("[data-filter]");
  if (!btn) return;
  filter = btn.dataset.filter;
  filters.querySelectorAll("button").forEach((b) => b.classList.toggle("active", b === btn));
  render();
});

render();
```

State ada di array, UI dirender dari data. `textContent`/`createElement` mencegah XSS. `id` memakai penghitung, bukan `Date.now()`. "Satu listener" maksudnya satu per jenis event pada `ul`, tidak per `li`.

### Soal 2: Tabs + Accordion

```js
const tabs = [...document.querySelectorAll('[role="tab"]')];

function activate(tab, { focus = false, updateHash = true } = {}) {
  tabs.forEach((t) => {
    const selected = t === tab;
    t.setAttribute("aria-selected", String(selected));   // string, bukan boolean
    t.tabIndex = selected ? 0 : -1;                      // roving tabindex
    document.getElementById(t.getAttribute("aria-controls")).hidden = !selected;
  });
  if (focus) tab.focus();
  if (updateHash && location.hash !== `#${tab.id}`) {
    history.pushState(null, "", `#${tab.id}`);
  }
}

document.querySelector('[role="tablist"]').addEventListener("keydown", (e) => {
  const i = tabs.indexOf(document.activeElement);
  if (i === -1) return;
  let target;
  if (e.key === "ArrowRight") target = tabs[(i + 1) % tabs.length];
  else if (e.key === "ArrowLeft") target = tabs[(i - 1 + tabs.length) % tabs.length];
  else if (e.key === "Home") target = tabs[0];
  else if (e.key === "End") target = tabs[tabs.length - 1];
  else return;
  e.preventDefault();
  activate(target, { focus: true });
});

tabs.forEach((t) => t.addEventListener("click", () => activate(t)));

function fromHash() {
  const t = tabs.find((x) => `#${x.id}` === location.hash) ?? tabs[0];
  activate(t, { updateHash: false });
}
window.addEventListener("hashchange", fromHash);   // tombol Back/Forward
fromHash();
```

Accordion:

```html
<div id="accordion">
  <div class="acc-item">
    <h3><button class="acc-header" id="h1" aria-expanded="false" aria-controls="p1">Judul 1</button></h3>
    <div class="acc-panel" id="p1" role="region" aria-labelledby="h1" inert>
      <div class="acc-inner">Isi 1</div>
    </div>
  </div>
  <!-- ulangi untuk item 2 dan 3 -->
</div>
```
```css
.acc-panel { display: grid; grid-template-rows: 0fr; transition: grid-template-rows 250ms; }
.acc-panel.open { grid-template-rows: 1fr; }
.acc-inner { overflow: hidden; }
```
```js
const acc = document.getElementById("accordion");
acc.addEventListener("click", (e) => {
  const btn = e.target.closest(".acc-header");
  if (!btn) return;
  const wasOpen = btn.getAttribute("aria-expanded") === "true";
  acc.querySelectorAll(".acc-header").forEach((h) => {
    const open = h === btn && !wasOpen;                 // single-open
    h.setAttribute("aria-expanded", String(open));
    const panel = document.getElementById(h.getAttribute("aria-controls"));
    panel.classList.toggle("open", open);
    panel.inert = !open;
  });
});
```

### Soal 3: Pencarian Live + Modal

```js
const $ = (s) => document.querySelector(s);
const input = $("#search");
const results = $("#results");
const URL_USERS = "https://jsonplaceholder.typicode.com/users";

let controller = null;
let timer;

input.addEventListener("input", () => {
  clearTimeout(timer);
  timer = setTimeout(() => load(input.value), 300);     // debounce
});

async function load(q) {
  controller?.abort();                                   // batalkan request lama
  controller = new AbortController();
  const { signal } = controller;
  results.textContent = "Loading...";

  try {
    const res = await fetch(URL_USERS, { signal });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);  // fetch tidak melempar untuk 404/500
    const data = await res.json();
    if (signal.aborted) return;
    const term = q.trim().toLowerCase();
    renderResults(data.filter((u) => u.name.toLowerCase().includes(term)));
  } catch (err) {
    if (err.name === "AbortError") return;               // pembatalan disengaja
    results.replaceChildren();
    const msg = document.createElement("p");
    msg.textContent = "Error";
    const retry = document.createElement("button");
    retry.textContent = "Retry";
    retry.addEventListener("click", () => load(input.value));
    results.append(msg, retry);
  }
}

function renderResults(users) {
  results.replaceChildren();
  if (!users.length) { results.textContent = "No results"; return; }
  for (const u of users) {
    const card = document.createElement("button");
    card.className = "card";
    card.textContent = u.name;
    card.addEventListener("click", () => openModal(u, card));
    results.append(card);
  }
}

let overlay = null;
let lastFocus = null;

function openModal(user, trigger) {
  lastFocus = trigger;
  overlay = document.createElement("div");
  overlay.className = "modal-overlay";
  overlay.innerHTML = `
    <div class="modal" role="dialog" aria-modal="true" aria-labelledby="m-title">
      <button class="close" aria-label="Tutup">×</button>
      <h2 id="m-title"></h2>
      <p class="email"></p><p class="phone"></p><p class="city"></p>
    </div>`;
  overlay.querySelector("#m-title").textContent = user.name;
  overlay.querySelector(".email").textContent = user.email;
  overlay.querySelector(".phone").textContent = user.phone;
  overlay.querySelector(".city").textContent = user.address.city;

  overlay.addEventListener("click", (e) => {
    if (e.target === overlay || e.target.closest(".close")) closeModal();
  });
  document.addEventListener("keydown", onKey);
  document.body.append(overlay);
  document.body.style.overflow = "hidden";
  overlay.querySelector(".close").focus();
}

function onKey(e) { if (e.key === "Escape") closeModal(); }

function closeModal() {
  document.removeEventListener("keydown", onKey);
  overlay?.remove();
  overlay = null;
  document.body.style.overflow = "";
  lastFocus?.focus();
}
```

Nilai tambah yang belum ada: *focus trap* agar Tab tidak keluar dari modal.

### Bonus

1. Urutan: `outer-capture`, `inner`. `outer` tidak muncul karena `stopPropagation()` menghentikan bubbling. Tanpa itu: `outer-capture`, `inner`, `outer`.
2. `e.target` = elemen asal event; `e.currentTarget` = elemen tempat listener dipasang. Delegation memakai `e.target` (dengan `closest`).
3. `querySelectorAll` = `NodeList` statis; `getElementsByClassName` = koleksi live.
4. `textContent` = teks mentah semua node; `innerText` = teks yang terlihat (memperhitungkan CSS); `innerHTML` = parsing HTML (rawan XSS).
5. `DOMContentLoaded` = DOM siap tanpa menunggu gambar/stylesheet; `load` = semuanya selesai. `defer` = skrip diunduh paralel, dijalankan setelah HTML selesai di-parse.
6. `closest(selector)` naik ke leluhur terdekat yang cocok (termasuk dirinya). Berguna saat klik mengenai anak (ikon/span) di dalam tombol.

---

# 8. REST API (Advanced)

Bobot terbesar (2 soal). Target total sekitar 75 menit, Node.js 18+.

## Soal

### API palsu (`mockApi.js`)

```js
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

const products = Array.from({ length: 47 }, (_, i) => ({
  id: i + 1,
  name: `Product ${i + 1}`,
  category: ["food", "tech", "toys"][i % 3],
  price: 10 + ((i * 7) % 90),
  stock: (i * 5) % 20,
}));

const users = Array.from({ length: 12 }, (_, i) => ({ id: i + 1, name: `User ${i + 1}` }));

// userId 13 sengaja tidak ada di /users
const orders = Array.from({ length: 95 }, (_, i) => ({
  id: i + 1,
  userId: ((i * 7) % 13) + 1,
  amount: 5 + ((i * 11) % 50),
}));

const paginate = (list, page, perPage) => ({
  page,
  per_page: perPage,
  total: list.length,
  total_pages: Math.ceil(list.length / perPage),
  data: list.slice((page - 1) * perPage, page * perPage),
});

let flakyCalls = 0;

// Mengembalikan { status, headers, body }
export async function mockGet(path, { signal } = {}) {
  const url = new URL(path, "http://mock");
  const page = Number(url.searchParams.get("page") ?? 1);
  await sleep(20 + Math.random() * 80);
  if (signal?.aborted) throw new DOMException("Aborted", "AbortError");

  switch (url.pathname) {
    case "/products": {
      const cat = url.searchParams.get("category");
      const list = cat ? products.filter((p) => p.category === cat) : products;
      return { status: 200, headers: {}, body: paginate(list, page, 10) };
    }
    case "/users":
      return { status: 200, headers: {}, body: paginate(users, page, 5) };
    case "/orders":
      return { status: 200, headers: {}, body: paginate(orders, page, 20) };
    case "/flaky": {
      flakyCalls++;
      if (flakyCalls % 4 === 1) return { status: 500, headers: {}, body: { error: "boom" } };
      if (flakyCalls % 4 === 2) return { status: 429, headers: { "retry-after": "1" }, body: {} };
      return { status: 200, headers: {}, body: { ok: true, calls: flakyCalls } };
    }
    case "/slow":
      await sleep(3000);
      return { status: 200, headers: {}, body: { ok: true } };
    default:
      return { status: 404, headers: {}, body: { error: "Not found" } };
  }
}
```

### Soal 1: Nilai Inventaris per Kategori (20 menit)

```js
async function getInventoryValue(category) { /* TODO */ }
```

- Ambil **semua halaman** `/products?category={category}&page={n}`, baca `total_pages`.
- Nilai inventaris = jumlah `price * stock`.
- Kembalikan `{ category, totalValue, productCount, mostValuable }`; `mostValuable` = **nama** produk dengan `price * stock` terbesar (sama: `id` terkecil).
- Kategori kosong: `{ category, totalValue: 0, productCount: 0, mostValuable: null }`.
- Status bukan `200`: lempar `Error` `Request failed: {status}`.

Jebakan: off-by-one (halaman mulai 1), berhenti saat `data` kosong vs `total_pages`, `encodeURIComponent`, mengabaikan `status`.

### Soal 2: `fetchWithRetry` (25 menit)

```js
async function fetchWithRetry(path, { retries = 3, baseDelay = 100, timeout = 1000 } = {}) { /* TODO */ }
```

- Kembalikan `body` jika `2xx`.
- Coba ulang (maks `retries` kali setelah percobaan pertama) hanya untuk `5xx`, `429`, dan error jaringan/timeout. `4xx` lain (mis. `404`): langsung lempar `Error` berisi status.
- Jeda: `baseDelay * 2^attempt` + jitter acak 0-50ms. Untuk `429`, hormati `retry-after` (detik), pakai yang lebih besar antara itu dan backoff.
- Tiap percobaan dibatasi `timeout` ms dengan `AbortController`; timeout boleh dicoba ulang.
- Gagal semua: `Failed after {n} attempts: {alasan}`.

```js
console.log(await fetchWithRetry("/flaky", { baseDelay: 50 }));
await fetchWithRetry("/missing").catch((e) => console.log(e.message));
await fetchWithRetry("/slow", { timeout: 200, retries: 1 }).catch((e) => console.log(e.message));
```

Jebakan: `clearTimeout` setelah request, jangan retry 404, 3 retries = 4 percobaan, header huruf kecil di mock (fetch asli: `res.headers.get("retry-after")`).

### Soal 3: Pelanggan Teratas dari Dua Endpoint (30 menit)

```js
async function topCustomers(n, { concurrency = 3 } = {}) { /* TODO */ }
```

- Ambil **semua** `/users` dan `/orders` (berpaginasi).
- Total `amount` per `userId`; kembalikan `n` teratas `[{ userId, name, total, orderCount }]`, urut `total` menurun lalu `userId` menaik.
- `userId` yang tidak ada di `/users` tetap dihitung dengan `name: "Unknown"`.
- Setelah halaman 1 diketahui, ambil sisanya **paralel dengan batas `concurrency`**. `/users` dan `/orders` boleh bersamaan.
- Hasil deterministik walau respons tiba tidak berurutan.
- `n` > jumlah pelanggan mengembalikan semua; `n <= 0` mengembalikan `[]`.

### Bonus konsep

1. Beda `401`, `403`, `404`, `409`, `422`, `429`, `502`, `503`; mana yang layak dicoba ulang?
2. Apa itu idempotent; mengapa `GET`/`PUT`/`DELETE` aman diulang tetapi `POST` biasanya tidak?
3. Pagination offset vs cursor.
4. Mengapa `fetch` tidak melempar error untuk 404/500?
5. Exponential backoff with jitter.
6. Masalah N+1 request.

## Kunci Jawaban

### Soal 1

```js
import { mockGet } from "./mockApi.js";

async function getInventoryValue(category) {
  const products = [];
  let page = 1;
  let totalPages = 1;

  do {
    const res = await mockGet(`/products?category=${encodeURIComponent(category)}&page=${page}`);
    if (res.status !== 200) throw new Error(`Request failed: ${res.status}`);
    products.push(...res.body.data);
    totalPages = res.body.total_pages;
    page++;
  } while (page <= totalPages);

  let totalValue = 0;
  let best = null;
  let bestValue = -1;
  for (const p of products) {
    const v = p.price * p.stock;
    totalValue += v;
    if (v > bestValue || (v === bestValue && p.id < best.id)) {
      best = p;
      bestValue = v;
    }
  }

  return {
    category,
    totalValue,
    productCount: products.length,
    mostValuable: best ? best.name : null,
  };
}
```

Loop berbasis `total_pages` lebih aman daripada berhenti saat `data` kosong (butuh satu request sia-sia). Kategori kosong: `total_pages = 0`, `best` tetap `null`. `bestValue = -1` membuat produk bernilai 0 tetap bisa terpilih.

### Soal 2

```js
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function fetchWithRetry(path, { retries = 3, baseDelay = 100, timeout = 1000 } = {}) {
  const maxAttempts = retries + 1;
  let lastReason = "unknown";

  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), timeout);
    let waitMs = baseDelay * 2 ** attempt + Math.random() * 50;   // backoff + jitter

    try {
      // Mock tidak berhenti di tengah sleep, jadi balapan dengan sinyal abort
      const aborted = new Promise((_, reject) =>
        controller.signal.addEventListener(
          "abort",
          () => reject(new DOMException("Aborted", "AbortError")),
          { once: true }
        )
      );
      const res = await Promise.race([mockGet(path, { signal: controller.signal }), aborted]);

      if (res.status >= 200 && res.status < 300) return res.body;

      if (res.status === 429) {
        const ra = Number(res.headers["retry-after"]);       // fetch asli: res.headers.get(...)
        if (Number.isFinite(ra)) waitMs = Math.max(waitMs, ra * 1000);
      } else if (res.status < 500) {
        const err = new Error(`Request failed: ${res.status}`);
        err.retryable = false;                               // 4xx lain: jangan ulang
        throw err;
      }
      lastReason = `HTTP ${res.status}`;
    } catch (e) {
      if (e.retryable === false) throw e;
      lastReason = e.name === "AbortError" ? "timeout" : e.message;
    } finally {
      clearTimeout(timer);
    }

    if (attempt < maxAttempts - 1) await sleep(waitMs);      // tidak tidur setelah percobaan terakhir
  }

  throw new Error(`Failed after ${maxAttempts} attempts: ${lastReason}`);
}
```

- `retries = 3` berarti 4 percobaan.
- Mock hanya memeriksa `signal` sekali di awal, jadi butuh `Promise.race` agar timeout memutus `/slow`. Dengan `fetch` asli, `AbortController` saja cukup.
- Alur `/flaky`: 500, lalu 429 (menunggu minimal 1 detik), lalu sukses.

### Soal 3

```js
async function mapPool(items, limit, fn) {
  const results = new Array(items.length);
  let next = 0;
  async function worker() {
    while (next < items.length) {
      const i = next++;
      results[i] = await fn(items[i], i);      // ditulis ke indeks, bukan push
    }
  }
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return results;
}

async function fetchPage(path, page) {
  const sep = path.includes("?") ? "&" : "?";
  const res = await mockGet(`${path}${sep}page=${page}`);
  if (res.status !== 200) throw new Error(`Request failed: ${res.status}`);
  return res.body;
}

async function fetchAllPages(path, concurrency) {
  const first = await fetchPage(path, 1);
  const rest = Array.from({ length: Math.max(0, first.total_pages - 1) }, (_, i) => i + 2);
  const others = await mapPool(rest, concurrency, (p) => fetchPage(path, p));
  return [first, ...others].flatMap((b) => b.data);        // urutan sesuai nomor halaman
}

async function topCustomers(n, { concurrency = 3 } = {}) {
  if (n <= 0) return [];

  const [users, orders] = await Promise.all([
    fetchAllPages("/users", concurrency),
    fetchAllPages("/orders", concurrency),
  ]);

  const names = new Map(users.map((u) => [u.id, u.name]));
  const stats = new Map();
  for (const o of orders) {
    const s = stats.get(o.userId) ?? { userId: o.userId, total: 0, orderCount: 0 };
    s.total += o.amount;
    s.orderCount++;
    stats.set(o.userId, s);
  }

  return [...stats.values()]
    .map((s) => ({
      userId: s.userId,
      name: names.get(s.userId) ?? "Unknown",
      total: s.total,
      orderCount: s.orderCount,
    }))
    .sort((a, b) => b.total - a.total || a.userId - b.userId)
    .slice(0, n);
}
```

`/users` dan `/orders` diambil bersamaan, halaman sisa paralel dengan batas `concurrency`, hasil digabung menurut nomor halaman sehingga deterministik. Asumsi: "pelanggan" = yang punya minimal satu pesanan. Di tes asli, ikuti contoh output soal.

### Jawaban konsep

1. `401` belum terautentikasi; `403` terautentikasi tapi dilarang; `404` tidak ada; `409` konflik state; `422` data tidak valid secara semantik; `429` terlalu banyak request; `502` gateway menerima respons buruk; `503` layanan sementara tidak tersedia. **Layak diulang:** `429`, `502`, `503` (umumnya `5xx`).
2. Idempotent = dipanggil berkali-kali, hasil akhir sama. `POST` biasanya membuat resource baru tiap kali, sehingga diulang bisa menggandakan data (kecuali memakai idempotency key).
3. Offset bisa melewatkan/menduplikasi item jika data berubah di tengah penelusuran; cursor menunjuk posisi tertentu sehingga stabil.
4. `fetch` hanya reject untuk kegagalan jaringan. Status HTTP error tetap resolve, jadi selalu cek `res.ok`.
5. Backoff eksponensial menambah jeda tiap percobaan; jitter (acak) mencegah banyak klien mengulang serentak (*thundering herd*).
6. N+1: satu request daftar lalu N request detail. Hindari dengan endpoint batch, parameter `include`/`expand`, atau ambil sekaligus lalu gabungkan di klien.

### Tips REST API di HackerRank

- Endpoint sering `jsonmock.hackerrank.com/api/...` dengan format `{ page, per_page, total, total_pages, data }`. Baca nama field persis dan perhatikan tipe data (angka kadang string).
- Buat versi benar dulu (loop `await` berurutan), optimalkan kalau ada waktu.
- Selalu cek `res.ok`/`res.status` sebelum `res.json()`.
- Ikuti library yang disediakan di kode awal (`axios`, `https`, dst).
- `console.log` respons halaman pertama untuk memahami struktur.
- Perhatikan format output: urutan, pembulatan, angka vs string.

---

# 9. Problem Solving (Intermediate)

Target total sekitar 70 menit. Tulis dulu kompleksitas target sebelum coding.

## Soal

### Soal 1: Subarray dengan Jumlah K (15 menit)

```js
function countSubarrays(nums, k) { /* TODO */ }
```

Hitung subarray berurutan yang jumlahnya tepat `k`. Array bisa berisi negatif dan nol. `n` sampai 200.000, target O(n).

```js
countSubarrays([1, 1, 1], 2);   // 2
countSubarrays([1, 2, 3], 3);   // 2
countSubarrays([1, -1, 0], 0);  // 3
```

Jebakan: sliding window biasa tidak bekerja dengan angka negatif; lupa `map.set(0, 1)`; urutan update map.

### Soal 2: Substring Terpanjang dengan Maksimal K Karakter Berbeda (15 menit)

```js
function longestKDistinct(s, k) { /* TODO */ }
```

Panjang substring terpanjang dengan paling banyak `k` karakter unik. Panjang `s` sampai 100.000, target O(n).

```js
longestKDistinct("eceba", 2); // 3
longestKDistinct("aa", 1);    // 2
longestKDistinct("abc", 0);   // 0
longestKDistinct("", 3);      // 0
```

### Soal 3: Interval (20 menit)

a) `mergeIntervals(intervals)`: gabungkan interval bertumpuk; yang bersentuhan (`[1,4]` dan `[4,5]`) ikut digabung. Input tidak terurut dan tidak boleh di-mutate.

```js
mergeIntervals([[1,3],[2,6],[8,10],[15,18]]); // [[1,6],[8,10],[15,18]]
mergeIntervals([[1,4],[4,5]]);                // [[1,5]]
mergeIntervals([[8,10],[1,3],[2,6]]);         // [[1,6],[8,10]]
mergeIntervals([]);                           // []
```

b) `minRooms(meetings)`: ruang minimum agar tidak bentrok; rapat yang berakhir tepat saat rapat lain dimulai boleh berbagi ruang.

```js
minRooms([[0,30],[5,10],[15,20]]); // 2
minRooms([[7,10],[2,4]]);          // 1
minRooms([[1,5],[5,8]]);           // 1
```

Sampai 100.000 interval, target O(n log n).

### Soal 4: Jalur Terpendek di Grid (20 menit)

```js
function shortestPath(grid) { /* TODO */ }
```

`S` = awal, `E` = tujuan, `.` = jalan, `#` = tembok. Gerak atas/bawah/kiri/kanan. Kembalikan langkah minimum atau `-1`. Grid sampai 1000x1000, target O(baris x kolom).

```js
shortestPath(["S..#", ".#..", "...E"]); // 5
shortestPath(["S#E"]);                  // -1
```

Bonus: boleh menghancurkan maksimal 1 tembok.

### Bonus konsep

1. Kompleksitas `arr.includes(x)` di dalam loop? Cara menurunkannya?
2. Kapan two pointers vs hash map?
3. Mengapa `arr.sort()` tanpa comparator salah untuk angka?
4. Beda BFS dan DFS; mana yang menjamin jalur terpendek pada graf tak berbobot?
5. Mengapa `[...a, ...b]` atau `arr.shift()` di dalam loop bisa membuat O(n) menjadi O(n²)?

## Kunci Jawaban

### Soal 1 (O(n))

```js
function countSubarrays(nums, k) {
  const seen = new Map([[0, 1]]);        // prefix sum 0 muncul sekali
  let sum = 0, count = 0;
  for (const x of nums) {
    sum += x;
    count += seen.get(sum - k) ?? 0;     // hitung SEBELUM memperbarui map
    seen.set(sum, (seen.get(sum) ?? 0) + 1);
  }
  return count;
}
```

Ide: subarray `(i, j]` berjumlah `k` jika `prefix[j] - prefix[i] = k`.

### Soal 2 (O(n))

```js
function longestKDistinct(s, k) {
  if (k === 0) return 0;
  const cnt = new Map();
  let left = 0, best = 0;
  for (let right = 0; right < s.length; right++) {
    const c = s[right];
    cnt.set(c, (cnt.get(c) ?? 0) + 1);
    while (cnt.size > k) {                       // while, bukan if
      const l = s[left];
      const v = cnt.get(l) - 1;
      if (v === 0) cnt.delete(l); else cnt.set(l, v);   // hapus kunci saat 0
      left++;
    }
    best = Math.max(best, right - left + 1);
  }
  return best;
}
```

### Soal 3 (O(n log n))

```js
function mergeIntervals(intervals) {
  const sorted = intervals.map((i) => [...i]).sort((a, b) => a[0] - b[0]); // salin: input tidak berubah
  const out = [];
  for (const [s, e] of sorted) {
    const last = out[out.length - 1];
    if (last && s <= last[1]) last[1] = Math.max(last[1], e);  // <= : bersentuhan ikut digabung
    else out.push([s, e]);
  }
  return out;
}

function minRooms(meetings) {
  const starts = meetings.map((m) => m[0]).sort((a, b) => a - b);
  const ends = meetings.map((m) => m[1]).sort((a, b) => a - b);
  let rooms = 0, j = 0;
  for (const s of starts) {
    if (s >= ends[j]) j++;     // ada rapat selesai (termasuk tepat saat mulai): pakai ruang sama
    else rooms++;              // tidak ada yang selesai: butuh ruang baru
  }
  return rooms;
}
```

### Soal 4 (O(R x C))

```js
function shortestPath(grid) {
  const R = grid.length;
  if (!R) return -1;
  const C = grid[0].length;

  let start = -1, end = -1;
  for (let r = 0; r < R; r++) {
    for (let c = 0; c < C; c++) {
      if (grid[r][c] === "S") start = r * C + c;
      else if (grid[r][c] === "E") end = r * C + c;
    }
  }
  if (start === -1 || end === -1) return -1;

  const dist = new Int32Array(R * C).fill(-1);   // sekaligus visited
  const queue = new Int32Array(R * C);
  let head = 0, tail = 0;
  dist[start] = 0;
  queue[tail++] = start;

  const dirs = [[1, 0], [-1, 0], [0, 1], [0, -1]];
  while (head < tail) {                           // indeks penunjuk, bukan shift()
    const cur = queue[head++];
    if (cur === end) return dist[cur];
    const r = (cur / C) | 0, c = cur % C;
    for (const [dr, dc] of dirs) {
      const nr = r + dr, nc = c + dc;
      if (nr < 0 || nc < 0 || nr >= R || nc >= C) continue;
      if (grid[nr][nc] === "#") continue;
      const id = nr * C + nc;
      if (dist[id] !== -1) continue;
      dist[id] = dist[cur] + 1;                   // tandai saat dimasukkan
      queue[tail++] = id;
    }
  }
  return -1;
}
```

Bonus (hancurkan 1 tembok): state `(r, c, used)` dengan `used ∈ {0,1}`, `dist` berukuran `R * C * 2`. Saat bertemu `#` dan `used === 0`, boleh masuk dengan `used = 1`. Kompleksitas tetap O(R x C).

### Jawaban konsep

1. `includes` O(n) waktu; di dalam loop jadi O(n²). Turunkan dengan `Set`/`Map` (O(n) waktu, O(n) memori).
2. Two pointers untuk data **terurut** atau pencarian pasangan/jendela; hash map untuk data **tak terurut** dan lookup cepat (two-sum, frekuensi).
3. `sort()` default membandingkan sebagai string: `[10, 9, 1]` menjadi `[1, 10, 9]`. Pakai `(a, b) => a - b`.
4. BFS per lapisan, DFS menyelam dulu. Pada graf tak berbobot, **BFS** menjamin jalur terpendek.
5. `shift()` O(n) dan `[...a, ...b]` menyalin seluruh array tiap iterasi; di dalam loop totalnya O(n²).

### Tips Problem Solving di HackerRank

- Baca format input/output; banyak soal memakai `stdin` (Node: `readline` atau `fs.readFileSync(0)`) dan output harus persis.
- Mulai dari brute force yang benar, lalu optimalkan. Lolos sebagian lebih baik daripada kosong.
- Uji edge case: kosong, satu elemen, semua sama, negatif, nilai maksimum.
- `n = 10^5` berarti O(n²) hampir pasti time-out.
- `Number` aman sampai sekitar 9 x 10^15; gunakan `BigInt` bila lebih besar.
- Gunakan `Map`/`Set`, bukan `includes`/`indexOf` di dalam loop.

---

# 10. Checklist Malam Sebelum Tes

## Prioritas belajar

1. Ulangi **REST API (Section 8)**: pola `fetchAllPages` dan `fetchWithRetry` (2 dari 10 soal).
2. Ulangi **Problem Solving soal 1 dan 2** (prefix sum, sliding window).
3. Latih **vanilla JS DOM** (todo, tabs, modal) tanpa React.
4. Ulangi pola React: `useEffect` + cleanup, fetch + loading/error, controlled input, form validasi.
5. TypeScript: utility types, generics, discriminated union.

## Persiapan teknis

- [ ] Chrome versi terbaru (v101+) atau Edge v140+
- [ ] Mulai paling lambat sekitar 13.00-14.00 WIB (batas login 17.45 WIB, durasi 210 menit)
- [ ] Kamera, mikrofon, dan izin browser berfungsi (jaga-jaga jika proctoring aktif)
- [ ] Koneksi stabil, laptop tercharge, notifikasi dimatikan
- [ ] Tab dan aplikasi lain ditutup; ruangan tenang
- [ ] Baca halaman instruksi sebelum menekan Start; tanyakan recruiter jika ada yang tidak jelas
- [ ] Tidur cukup

## Saat tes

- Lihat semua soal dulu, kerjakan yang paling pasti lebih dulu.
- Jalankan test case sesering mungkin; ikuti `data-testid`, class, dan teks persis dari soal.
- Jangan lebih dari batas waktu per soal; kembali lagi nanti.
- Solusi sebagian lebih baik daripada kosong.
- Sisakan 10-15 menit terakhir untuk pengecekan.

Semoga sukses!
