# Proyek Prototipe EKG dengan STM32F411

---

## 1. Overview Proyek

Proyek ini bertujuan membuat prototipe EKG untuk keperluan pembelajaran dengan pendekatan yang mendekati standar medis, namun tetap mempertimbangkan kesulitan implementasi, ketersediaan komponen, dan biaya. Sistem dirancang agar teknisi dapat memahami prinsip kerja mulai dari elektroda hingga pengolahan data digital.

---

## 2. Arsitektur Sistem

- **Elektroda**: 10 lead utama (RA, LA, LL, V1-V6), dengan tambahan WCT (Wilson Central Terminal) dan RLD (Driven Right Leg).
- **Filter HPF dan LPF**: Simple RC filter dengan cutoff 0.5 Hz (HPF) dan 100 Hz (LPF).
- **Amplifier**: AD620 sebagai instrumentation amplifier untuk setiap channel.
- **MCU**: STM32F411CEU6 dengan ADC 12-bit, sampling 500 SPS/channel.
- **Komunikasi**: Data dikirim via UART dengan baudrate 250k (opsi 500k jika diperlukan) dalam format `[time, s1, s2, ..., s9]`.
- **Pengolahan Data**: Dilakukan di mesin eksternal (PC atau embedded lain), STM32 hanya bertugas baca ADC dan kirim data.

---

## 3. Filter RC Perhitungan

### 3.1 High Pass Filter (HPF) 0.5 Hz

Formula cutoff frequency:

\[
f_c = \frac{1}{2 \pi R C}
\]

Diketahui:

- \( f_c = 0.5 \, \text{Hz} \)
- \( C = 100 \, \mu F = 100 \times 10^{-6} F \)

Hitung resistor \( R \):

\[
R = \frac{1}{2 \pi f_c C} = \frac{1}{2 \pi \times 0.5 \times 100 \times 10^{-6}} \approx 3183 \, \Omega \approx 3.3 k\Omega
\]

---

### 3.2 Low Pass Filter (LPF) 100 Hz

Dengan \( C = 100 \mu F \), hitung \( R \):

\[
R = \frac{1}{2 \pi f_c C} = \frac{1}{2 \pi \times 100 \times 100 \times 10^{-6}} \approx 15.9 \, \Omega
\]

**Catatan:** Resistor ini sangat kecil, biasanya kapasitor 100 µF terlalu besar untuk LPF cutoff 100 Hz. Disarankan menggunakan nilai kapasitor lebih kecil (misalnya 0.01 µF) agar resistor LPF berada di kisaran kΩ (praktis untuk RC low pass).

Contoh jika \( C = 0.01 \mu F \):

\[
R = \frac{1}{2 \pi \times 100 \times 0.01 \times 10^{-6}} \approx 159 k\Omega
\]

---

## 4. Tabel Koneksi Elektroda ke AD620

| Channel | Input (+) Elektroda | Input (−) Elektroda (WCT) |
| ------- | ------------------- | ------------------------- |
| 1       | RA                  | WCT                       |
| 2       | LA                  | WCT                       |
| 3       | LL                  | WCT                       |
| 4       | V1                  | WCT                       |
| 5       | V2                  | WCT                       |
| 6       | V3                  | WCT                       |
| 7       | V4                  | WCT                       |
| 8       | V5                  | WCT                       |
| 9       | V6                  | WCT                       |

---

## 5. Wilson Central Terminal (WCT) dan Buffer AD620

- **WCT** dibentuk dari rata-rata tegangan elektroda RA, LA, dan LL sebelum filter analog, secara matematis:

\[
WCT = \frac{RA + LA + LL}{3}
\]

- Rangkaian averaging menggunakan resistor identik dari RA, LA, LL yang kemudian masuk ke input AD620 yang dikonfigurasi sebagai buffer gain=1 (tanpa resistor gain eksternal).

- Buffer ini memberikan impedansi output rendah agar sinyal WCT stabil dan kuat untuk referensi input amplifier channel lain dan sebagai input DRL.

- **Rangkaian sederhana WCT buffer menggunakan AD620**:
  - Input: titik rata-rata RA, LA, LL (masuk ke pin + AD620).
  - Output: sinyal buffer WCT.
  - Gain diatur ke 1 (tanpa resistor gain).

---

## 6. Driven Right Leg (RLD) dan Amplifiernya

- **RLD** adalah sinyal invers dari WCT yang diinjeksikan kembali ke elektroda RL untuk mengurangi noise common-mode.

- Rangkaian RLD terdiri dari:

  - Op-amp konfigurasi inverting amplifier.
  - Gain disesuaikan (biasanya 10–100×).
  - Output melalui resistor pembatas arus (100 kΩ – 1 MΩ) ke elektroda RL demi keamanan.

- **Rekomendasi op-amp untuk RLD:**

  - Rail-to-rail low noise, seperti OPA2333, MCP6002, atau TL072 (dengan supply dual rail).

- RLD bersifat opsional dan bisa ditambahkan jika noise 50/60 Hz sangat mengganggu.

---

## 7. Sampling dan Komunikasi STM32

- Sampling ADC: 12-bit, 500 samples per second (SPS) per channel.
- Data 9 channel dikirim secara serial via UART dengan baudrate standar 250000 bps (opsional 500000 bps).
- Format data: `[time, s1, s2, ..., s9]` dalam format biner atau ASCII (disesuaikan).

---

## 8. Aspek Kritis dan Catatan Penting

- **Proteksi pasien:**

  - Gunakan resistor pembatas arus pada jalur elektroda, khususnya di jalur RLD dan input amplifier.
  - Jaga isolasi rangkaian dan grounding yang benar untuk keselamatan.

- **Noise dan Grounding:**

  - Pastikan jalur elektroda dan referensi WCT di-ground dengan baik.
  - Tempatkan RLD jika noise lingkungan tinggi.

- **Filter RC LPF:**

  - Perhatikan nilai kapasitor dan resistor agar nilai praktis dan efektif.
  - Kapasitor elco 100 µF ideal untuk HPF, tapi kurang cocok untuk LPF 100 Hz → gunakan nilai kapasitor lebih kecil (film, keramikk).

- **Pembuatan WCT dan RLD:**
  - WCT harus di-buffer agar stabil.
  - RLD harus membalik sinyal WCT dan membatasi arus injeksi.

---

## 9. Referensi Singkat Teori EKG

- Lead standar: I, II, III dibentuk dari elektroda limb RA, LA, LL.
- Augmented leads (aVR, aVL, aVF) dihitung secara digital dari lead standar.
- Precordial leads V1–V6 diambil diferensial terhadap WCT.

---

## 10. Diagram Blok Sistem (Sederhana)

Electroda RA, LA, LL ---+
|---[Resistor Identik]---+
Electroda RA, LA, LL ---+ |
+----> WCT (Wilson Central Terminal)
| (rata-rata tegangan elektroda)
|
v
[Buffer AD620 (gain=1)] --------+----> Input (−) AD620 untuk semua channel
|
Elektroda V1, V2, ..., V6 ---[HPF 0.5 Hz]---+ +----> Input (+) AD620 channel terkait
| |
Elektroda RA, LA, LL ---[HPF 0.5 Hz]-------+ |
|----> LPF 100 Hz --> ADC STM32
(12-bit, 500 SPS per channel)

WCT buffer (output) --> RLD Amplifier (inverting amp, gain ~10-100) --> resistor pembatas --> Elektroda RL (untuk reduksi noise)

Data ADC --> UART TX --> Mesin eksternal untuk pemrosesan data digital dan perhitungan lead

## Catatan Penting tentang Penghitungan Lead dengan Referensi WCT

Pada sistem EKG ini, setiap kanal AD620 menggunakan **WCT (Wilson Central Terminal)** sebagai referensi negatif (input −), sehingga sinyal yang terbaca adalah tegangan diferensial antara elektroda target dan WCT:

\[
S*{X} = V*{X} - V\_{WCT}
\]

di mana \(X\) adalah elektroda RA, LA, atau LL, dan:

\[
V*{WCT} = \frac{V*{RA} + V*{LA} + V*{LL}}{3}
\]

---

### Bagaimana menghitung Lead I (LA − RA)?

Lead I didefinisikan sebagai:

\[
Lead\ I = V*{LA} - V*{RA}
\]

Karena kita hanya memiliki \(S*{LA}\) dan \(S*{RA}\), maka:

\[
Lead\ I = (V*{LA} - V*{WCT}) - (V*{RA} - V*{WCT}) = S*{LA} - S*{RA}
\]

Perhatikan bahwa \(V\_{WCT}\) tereliminasi saat pengurangan, sehingga kamu cukup mengurangkan sinyal hasil ADC dari kanal LA dan RA.

---

### Penghitungan Lead II dan Lead III juga sama:

\[
Lead\ II = S*{LL} - S*{RA}
\]
\[
Lead\ III = S*{LL} - S*{LA}
\]

---

### Kesimpulan

Meskipun setiap sinyal kanal diambil relatif terhadap WCT, kamu dapat menghitung lead standar EKG secara akurat hanya dengan mengurangkan sinyal diferensial hasil ADC.

Ini mempermudah pengolahan sinyal digital dan menghilangkan noise common-mode yang tidak diinginkan.
