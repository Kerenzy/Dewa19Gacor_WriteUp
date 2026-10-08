# Dewa19 Gacor — 500 PTS

**Difficulty:** Hard
**Author:** Dxrie

> *The house always wins. Or does it?*

Awalnya gua kira challenge ini bakal jadi challenge slot biasa: manipulasi balance, negative bet, atau mungkin RNG yang gampang ditebak.

Ternyata bukan.

Setelah baca source code-nya, ternyata masalah utamanya ada di **`Math.random()` yang dipakai server**.

---

## 1. Nemunya dari `/api/stats`

Pertama gua cari endpoint yang berhubungan sama random.

Terus nemu:

```js
app.get('/api/stats', (req, res) => {
    const a = Math.random();
    const b = Math.random();
    const c = Math.random();

    res.json({
        liveA: a * 8,
        liveB: b * 1024,
        liveC: c * 8
    });
});
```

Ini langsung sus banget.

Server sebenarnya ngasih kita hasil `Math.random()`, cuma hasilnya dikali angka:

```text
liveA = random × 8
liveB = random × 1024
liveC = random × 8
```

Jadi tinggal dibalikin:

```text
a = liveA / 8
b = liveB / 1024
c = liveC / 8
```

dan kita dapet lagi nilai `Math.random()` yang dipakai server.

Lebih parahnya lagi, `Math.random()` ini **global**, jadi bukan RNG yang terpisah untuk setiap user atau endpoint.

Artinya kalau kita tahu output-output sebelumnya, kita bisa coba nebak output berikutnya.

---

## 2. Cari tahu apa yang dipakai buat jackpot

Terus gua cek endpoint `/api/spin`.

Satu kali spin ternyata makan **6 random number**:

```js
const rGame = Math.random();

const r1 = Math.random();
const r2 = Math.random();
const r3 = Math.random();

const rMsg = Math.random();
const rTx = Math.random();
```

Dan reel bakal jadi `SEVEN` kalau:

```text
random >= 0.9375
```

Jadi jackpot membutuhkan:

```text
r1 >= 0.9375
r2 >= 0.9375
r3 >= 0.9375
```

secara berurutan.

Kalau random-nya benar-benar random, kemungkinan ini kecil banget.

Tapi karena kita bisa **leak random output-nya**, kita tinggal coba predict.

---

# 3. Masalahnya: V8 `Math.random()` bukan sekadar random biasa

Di sini mulai agak ribet.

V8 menggunakan **xorshift128+** untuk `Math.random()`.

Jadi secara konsep ada internal state:

```text
state
  ↓
xorshift128+
  ↓
random
  ↓
state berikutnya
```

Kalau kita punya cukup output, state tersebut bisa direcover.

Solver yang gua pakai basically melakukan itu menggunakan **Z3**.

Tapi ada satu jebakan lagi.

---

# 4. V8 punya random cache

V8 nggak selalu menjalankan PRNG dari awal setiap kali JavaScript memanggil:

```js
Math.random()
```

Random number dibatch ke dalam cache.

Challenge ini menggunakan cache sebanyak **64 output**.

Dan yang agak ngeselin:

> V8 menghasilkan batch tersebut lalu mengembalikannya ke JavaScript dalam urutan terbalik.

Jadi urutan random yang kita lihat dari endpoint bukan sekadar:

```text
state 1
state 2
state 3
state 4
...
```

melainkan arahnya terbalik terhadap urutan state PRNG sebenarnya.

Makanya solver perlu memperhitungkan direction ini.

---

# 5. Recover state dengan Z3

Bagian `calibrate()` di solver adalah bagian yang melakukan recovery state.

Kita ambil beberapa output terakhir:

```python
window = history[-CAL_WINDOW:]
state = calibrate(window)
```

dengan:

```python
CAL_WINDOW = 6
```

Jadi kita kasih Z3 enam output random yang berurutan.

Di dalam `calibrate()`, sequence-nya dibalik:

```python
rev = seq[::-1]
```

lalu dimodelkan sebagai operasi xorshift128+.

Intinya Z3 diminta mencari:

```text
state0
state1
```

yang kalau dijalankan melalui algoritma xorshift128+ menghasilkan output random yang sama dengan leak dari `/api/stats`.

Kalau ketemu:

```python
solver.check() == z3.sat
```

berarti kita berhasil recover state.

Kalau:

```python
unsat
```

berarti kemungkinan window kita melewati batas cache refill 64-entry.

Makanya calibration nggak selalu berhasil.

---

# 6. Setelah state ketemu, nggak perlu Z3 lagi

Setelah punya state yang valid, kita nggak perlu menjalankan Z3 terus-terusan.

Solver tinggal menggunakan implementasi Python dari xorshift128+:

```python
def predict_ahead(state, n):
    cur0, cur1 = state
    out = []

    for _ in range(n):
        out.append(to_double(cur0))
        cur0, cur1 = inverse_step(cur0, cur1)

    return out
```

Di sini kita generate prediction ke depan.

Gua pakai:

```python
LOOKAHEAD = 60
```

Kenapa cuma 60?

Karena cache V8 ukurannya 64.

Kalau prediction melewati refill boundary, state yang kita prediksi bisa langsung salah total.

Jadi lebih aman tetap berada di dalam batch yang sama.

---

# 7. Cari sequence jackpot

Sekarang kita punya misalnya 60 random yang akan keluar.

Kita tinggal scan enam-enam karena satu `/api/spin` menggunakan enam random:

```text
rGame
r1
r2
r3
rMsg
rTx
```

Solver-nya:

```python
for offset in range(len(predicted) - 6):
    w = predicted[offset:offset + 6]

    if (
        w[1] >= 0.9375 and
        w[2] >= 0.9375 and
        w[3] >= 0.9375
    ):
        return offset
```

Yang dicari cuma:

```text
ANY
SEVEN
SEVEN
SEVEN
ANY
ANY
```

Begitu ketemu, kita tahu:

> "Oke, kalau sekarang kita berhasil maju `offset` random calls, spin berikutnya bakal jackpot."

---

# 8. Tapi gimana cara majuin RNG tanpa spin?

Ini bagian yang menurut gua paling satisfying.

Kita nggak mau melakukan `/api/spin` berkali-kali karena balance bakal habis.

Untungnya ada endpoint yang juga menggunakan `Math.random()` tapi **nggak memengaruhi balance**.

Contohnya:

```text
/api/stats
/api/recent
```

`/api/stats`:

```text
3 random calls
```

sedangkan:

```http
/api/recent?limit=1
```

menggunakan:

```text
2 random calls
```

Jadi kita bisa "burn" random number secara gratis.

Misalnya jackpot diprediksi berada:

```text
offset = 8
```

Kita tinggal konsumsi 8 random sebelum melakukan spin.

Solver melakukan ini lewat:

```python
def burn_draws(offset):
    ...
```

Dia basically mencari kombinasi:

```text
3 + 3 + 2
```

untuk mendapatkan tepat 8 draw.

Jadi:

```text
/api/stats
/api/stats
/api/recent?limit=1
```

Setelah itu posisi RNG sudah tepat di sequence jackpot.

---

# 9. Fire the spin

Setelah RNG sudah di-align:

```python
spin = SESSION.post(
    f"{BASE}/api/spin",
    timeout=5
).json()
```

Kalau prediksi kita benar:

```text
r1 >= 0.9375
r2 >= 0.9375
r3 >= 0.9375
```

maka:

```text
jackpot = true
```

Balance naik sampai memenuhi requirement challenge.

---

# 10. Ada kemungkinan false positive

Nah ini juga penting.

Kadang solver bisa menemukan jackpot di prediction, tapi pas `/api/spin` ternyata bukan jackpot.

Kenapa?

Karena kita bisa saja sudah melewati **cache refill boundary**.

Prediction kita cuma reliable selama masih berada di batch 64 random yang sama.

Makanya solver melakukan:

```python
if fire_and_claim():
    return

history = []
```

Kalau gagal, history dibuang dan kita mulai calibration lagi dari awal.

Jadi bukan berarti exploit-nya salah.

Kita cuma kena boundary V8.

---

# 11. Ambil flag

Kalau jackpot berhasil:

```python
if spin.get("jackpot"):
    flag = SESSION.get(
        f"{BASE}/api/flag",
        timeout=5
    ).json()
```

dan akhirnya:

```text
C2SI{geb_gacor123}
```

---

# TL;DR

Kalau disingkat banget, solve-nya sebenarnya cuma:

```text
/api/stats
     ↓
leak Math.random()
     ↓
recover xorshift128+ state
     ↓
predict next random numbers
     ↓
find SEVEN SEVEN SEVEN
     ↓
burn random calls gratis
     ↓
/api/spin
     ↓
JACKPOT
     ↓
/api/flag
```

Yang bikin challenge ini tricky bukan cuma `Math.random()`-nya.

Tapi harus ngerti juga kalau:

* `Math.random()` V8 menggunakan PRNG yang predictable
* `/api/stats` membocorkan output RNG
* semua endpoint menggunakan RNG global yang sama
* V8 melakukan caching 64 random values
* cached values diberikan dalam reverse order
* prediction cuma aman selama belum melewati refill boundary
* random state bisa direcover dengan Z3
* endpoint gratis bisa dipakai buat **burn RNG state** tanpa kehilangan balance

**Intinya: house-nya bukan kalah karena gambling. House-nya kalah karena RNG-nya bocor.** 💀
