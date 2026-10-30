<!--
CARA MEMASANG KE README.md
1. Bagian A  -> tempel SETELAH bagian "4.7 Konsep Pendukung", sebagai "4.8".
2. Bagian B  -> tambahkan satu baris ke tabel di bagian 4.2.
3. Bagian C  -> tambahkan poin 9-15 ke "7. Catatan Jujur" dan ganti poin 8 dengan versi di bawah.
4. Bagian D  -> tempel di akhir bagian "6. Pertanyaan dan Jawaban" sebagai "I".
5. Bagian E  -> ganti kotak "Cakupan" di bagian awal README.
Semua nomor baris sudah dicocokkan dengan kode di CLONE_PBO.zip.
-->

# BAGIAN A.

### 4.8 Pemetaan OOP per Fitur Website

Bagian ini menunjukkan **di fitur mana** tiap konsep OOP muncul, lengkap dengan file dan nomor barisnya. Setiap fitur memakai format yang sama: *Di website*, *Fakta penting*, tabel *Konsep → Bukti*, *Kalimat presentasi*, dan *Jangan klaim*.

**Aturan dasar:**
- Semua class OOP ada di backend Python (`models.py`, `views_admin.py`, `admin.py`). JavaScript proyek ini tidak memakai `class`.
- Kebanyakan view adalah **fungsi biasa**. OOP-nya ada pada **object yang dipakai** fungsi itu.
- Hanya klaim yang ada di kolom "Bukti". Bagian "Jangan klaim" adalah hal yang sering dikira OOP padahal bukan.

**Class yang dipakai di seluruh fitur:**

| Class | Letak | Peran |
|---|---|---|
| `ItemKatalog` (abstract) → `BukuDigital`, `Jurnal` | `models.py:33, 49, 69` | Abstraksi dan pewarisan |
| `CustomUser(AbstractUser)` | `models.py:178` | Pewarisan, property, method perilaku |
| `Peminjaman`, `Bookmark`, `Notifikasi` | `models.py:84, 111, 157` | Asosiasi lewat `ForeignKey` |
| `AchievementGambar` | `models.py:331` | Property `ikon_tampil` |
| `AchievementForm(forms.ModelForm)` | `views_admin.py:471` | Pewarisan, overriding `__init__`, class `Meta` bersarang |
| `AchievementGambarAdmin(admin.ModelAdmin)` | `admin.py:19` | Pewarisan (panel `/admin` Django) |

---

## USER

### 1. Login

**Di website:** form "NIM" dan "Password", ada "Ingat Saya" dan "Lupa Password". Kata "NIM" di layar adalah field `username` di kode.

**Fakta penting:** `login_view` (`views.py:262`) adalah fungsi biasa, bukan class. OOP-nya ada pada object yang dipakai fungsi itu:

```python
user = authenticate(request, username=username_input, password=password_input)
...
kandidat = CustomUser.objects.filter(username=username_input or "").first()
if kandidat and not kandidat.is_active and kandidat.check_password(password_input or ""):
```

| Konsep | Bukti |
|---|---|
| Object | `authenticate(...)` mengembalikan object `CustomUser`, karena `AUTH_USER_MODEL = 'perpustakaan.CustomUser'` (`config/settings.py:101`) |
| Pewarisan | `user.is_staff`, `kandidat.is_active`, dan `kandidat.check_password(...)` (`views.py:297`) diwarisi dari `AbstractUser`. Tidak ditulis di `models.py` |
| Atribut tambahan | `kandidat.menunggu_persetujuan` milik `CustomUser` |
| Atribut tambahan | `otp_code` dan `otp_created_at` di `CustomUser`, disiapkan untuk reset password |

**Kalimat presentasi:** "Login memakai *inheritance*. `CustomUser` mewarisi `AbstractUser`, jadi autentikasi, `check_password`, dan `is_active` kami dapat dari kelas induk, lalu kami tambah atribut seperti `menunggu_persetujuan`."

**Jangan klaim:** enkapsulasi, polimorfisme, atau abstraksi di Login. Konsep itu tidak muncul di sana.

### 2. Register

**Di website:** form NIM, nama tampilan, email, password, dan konfirmasi password.

**Yang benar-benar OOP di `register_view` (`views.py:307`):**

| Konsep | Bukti |
|---|---|
| Class dan object | `CustomUser.objects.filter(username=nim).first()` dan `filter(email__iexact=email).exists()` (`views.py:332, 341`) untuk cek duplikat |
| Pembuatan object | `CustomUser.objects.create_user(...)` (`views.py:353`) membuat satu object user |
| Pewarisan | `create_user` (hash password otomatis) berasal dari `AbstractUser`. `nama_tampilan` adalah atribut tambahan milik `CustomUser` |

**Kalimat presentasi:** "Register membuat object `CustomUser`. Penyimpanan password aman kami dapat dari `AbstractUser`, lalu kami tambah atribut `nama_tampilan`."

**Jangan klaim:** `validasi_password` (`views.py:55`) sebagai OOP. Itu fungsi biasa.

### 3. Dashboard

**Di website:** sapaan nama, peringkat, "Bacaanmu" (bacaan terakhir dan sisa hari), jumlah sedang dipinjam, total dibaca, bookmarks, dan "Rekomendasi Hari Ini".

**Yang benar-benar OOP di `dashboard` (`views.py:375`):**

| Konsep | Bukti |
|---|---|
| Enkapsulasi | `user.reset_streak_if_expired()` (`views.py:377`, definisi `models.py:246`). `buku.kategori_label` (`views.py:421`). `{{ user.nama_asli }}` di template |
| Asosiasi | `Peminjaman.objects.filter(user=user, status="Dibaca")` (`views.py:379-380`). Bacaan terakhir lewat `p.buku_digital` (`views.py:408`). `Bookmark.objects.filter(user=user).count()` (`views.py:439`) |
| Class dan object | `BukuDigital.objects.order_by("id_item")` (`views.py:449`) untuk rekomendasi |
| Method warisan | `p.save(update_fields=["tanggal_kembali"])` (`views.py:412`) |

**Kalimat presentasi:** "Dashboard tidak menghitung streak sendiri. Ia memanggil `user.reset_streak_if_expired()`, karena aturannya dienkapsulasi di class `CustomUser`."

**Jangan klaim:** `bacaan_terakhir` sebagai object. Itu dict yang membungkus object. Peringkat (`views.py:383`) dihitung di view, bukan method class.

### 4. Katalog Buku

**Di website:** daftar buku, filter kategori, pencarian, dan tombol bookmark. Sebagian buku dari database lokal, sebagian dari Open Library lewat JavaScript.

**Yang benar-benar OOP di `katalog_view` (`views.py:462`):**

```python
buku_queryset = BukuDigital.objects.all().distinct()
...
'kategori': buku.kategori_label,
'subjek': [buku.kategori_label] if buku.kategori else [],
```

| Konsep | Bukti |
|---|---|
| Class dan object | `BukuDigital.objects.all()` (`views.py:466`). List `buku_lokal` dibentuk dari object itu |
| Pewarisan | `buku.id_item`, `judul`, `penulis`, `tahun_terbit`, `cover` diwarisi dari `ItemKatalog`. `deskripsi` dan `kategori` milik `BukuDigital` |
| Enkapsulasi | `buku.kategori_label` (`views.py:515-516`) memanggil property (`models.py:57`) |
| Asosiasi | `Bookmark.objects.filter(user=request.user, buku_digital__isnull=False)` (`views.py:498`) |
| Pembuatan object | Buku Open Library menjadi `BukuDigital.objects.create(...)` di `_pastikan_buku` (`views.py:547`) |

**Kalimat presentasi:** "Setiap buku di katalog adalah object `BukuDigital` yang mewarisi `ItemKatalog`. Label kategorinya lewat property `kategori_label`, jadi aturan konversinya tidak ditulis ulang di view."

**Jangan klaim:** tampilan hasil Open Library (dict dan JavaScript tanpa class).

### 5. Jurnal (E-Journal)

**Di website:** daftar jurnal (cover, judul, penulis, tahun), pencarian, dan tombol baca.

**Yang benar-benar OOP di `ejournal_view` (`views.py:684`):**

| Konsep | Bukti |
|---|---|
| Class dan object | `Jurnal.objects.order_by("judul", "id_item")` (`views.py:687`). `get_object_or_404(Jurnal, pk=jurnal_id)` di `baca_jurnal` (`views.py:760`) |
| Pewarisan | `judul`, `penulis`, `cover`, `tahun_terbit` diwarisi dari `ItemKatalog`. `issn`, `volume`, `bidang_ilmu`, `link_akses` milik `Jurnal` (`models.py:69`) |
| Polimorfisme | `Jurnal.__str__` mengembalikan `[Jurnal] ...` dan `BukuDigital.__str__` `[Digital] ...` (`models.py:62, 77`). **Tidak terlihat di halaman ini**, demokan lewat `python manage.py shell` |

**Kalimat presentasi:** "Jurnal adalah subclass `ItemKatalog` seperti buku. Atribut dasarnya sama, tetapi `Jurnal` menambah ISSN, volume, dan bidang ilmu."

**Jangan klaim:** daftar yang tampil berbentuk dict (`baca_url`, `views.py:695`). Entri `JSON-{index}` (`views.py:715`) juga bukan object model.

### 6. Bookmarks

**Di website:** koleksi buku dan jurnal tersimpan, hapus bookmark, dan paginasi.

**Yang benar-benar OOP di `bookmarks_view` (`views.py:877`):**

| Konsep | Bukti |
|---|---|
| Asosiasi | `Bookmark` punya `ForeignKey` ke user, `buku_digital`, dan `jurnal` (`models.py:111-121`). `Bookmark.objects.filter(user=request.user)` (`views.py:879`). Template: `mark.buku_digital.judul`, `mark.jurnal.judul` |
| Enkapsulasi | `mark.buku_digital.kategori_label` (`bookmarks.html:83-84`) |
| Method warisan | `bookmarks.delete()` (`views.py:674`), `bookmark.delete()` (`views.py:909`), `get_or_create` (`views.py:679`) |
| Pewarisan | Buku dan jurnal tampil dengan atribut warisan yang sama (`judul`, `penulis`, `cover`) |

**Kalimat presentasi:** "Bookmark adalah object yang menghubungkan user dengan buku atau jurnal lewat `ForeignKey`. Datanya tidak disalin, melainkan diakses lewat relasi."

**Jangan klaim:** percabangan `{% if mark.buku_digital %}` di template sebagai polimorfisme. Itu hanya kondisi.

### 7. Top Reader

**Di website:** podium 3 teratas dan tabel Rank, Pengguna, Streak, Total EXP.

**Yang benar-benar OOP di `leaderboard_view` (`views.py:930`):**

| Konsep | Bukti |
|---|---|
| Class dan object | `CustomUser.objects.filter(...)` (`views.py:931`). `top_3[0].exp` (`views.py:936`) |
| Enkapsulasi | `podium.rank1.nama_asli` di template |
| Pewarisan | `exp`, `streak_days`, dan `foto_profil` adalah atribut tambahan `CustomUser` di atas `AbstractUser` |

**Kalimat presentasi:** "Papan peringkat memakai object `CustomUser`. Nama yang tampil lewat property `nama_asli`, jadi aturan pemilihan nama tidak ditulis ulang di template."

**Jangan klaim:** perhitungan persen bar (`views.py:940`) dan peringkat sebagai method class. Itu logika di view.

### 8. Profil

**Di website:** informasi pribadi, foto profil, background cover, ganti password, level, streak, achievement, dan profil user lain (`profil/<username>/`).

**Yang benar-benar OOP di `profil_view` (`views.py:957`):**

| Konsep | Bukti |
|---|---|
| Pewarisan | `user.check_password(...)` dan `user.set_password(...)` (`views.py:1022-1023`) diwarisi dari `AbstractUser`. `user.save()` (`views.py:1018`) dari `Model` |
| Enkapsulasi | `user.nama_asli`. `user.update_streak()` (`views.py:1311`, definisi `models.py:226`) |
| Object dari field | `user.foto_profil.delete(save=False)` (`views.py:988-1013`) dan `.url`. `ImageField` menghasilkan object file dengan method sendiri |
| Class dan object | `get_object_or_404(CustomUser, username=username)` (`views.py:1191`) untuk profil user lain |

**Kalimat presentasi:** "Ganti password memakai method `set_password()` warisan, dan streak dihitung lewat method `update_streak()` di class `CustomUser`."

**Jangan klaim:** level (`user.exp // LEVEL_EXP + 1`, `views.py:1035`) sebagai property. Itu dihitung di view. `_achievements` (`views.py:1075`) adalah fungsi.

---

## ADMIN

*Semua halaman admin memakai `{% extends 'base_admin.html' %}`. Itu fitur template Django, jadi sebut sebagai analogi pewarisan saja. Rute admin ada di `views_admin.py` (`urls.py:36-62`).*

### 9. Dashboard Admin

**Di website:** total anggota, total buku, sedang dipinjam, dikembalikan minggu ini, grafik 7 hari, aktivitas terbaru, dan buku terpopuler.

**Yang benar-benar OOP di `dashboard_admin` (`views_admin.py:157`):**

| Konsep | Bukti |
|---|---|
| Class dan object | `BukuDigital.objects.count()` dan `Jurnal.objects.count()` (`views_admin.py:187-188`) |
| Asosiasi | `BukuDigital.objects.annotate(jumlah=Count("peminjaman"))` (`views_admin.py:170`) memakai relasi balik ke `Peminjaman` |
| Pewarisan | `_judul_item` (`views_admin.py:58-61`) memakai `item.judul`. Komentar di baris 56 menyebut atribut itu dari `ItemKatalog` |
| Enkapsulasi | `_nama_pengguna` mengembalikan `user.nama_asli` (`views_admin.py:64`) |

**Kalimat presentasi:** "Dashboard admin mengambil statistik dari object model dan memanfaatkan relasi `Peminjaman` ke `BukuDigital` untuk menghitung buku terpopuler."

**Jangan klaim:** `_aktivitas_terbaru` (`views_admin.py:67`) dan `_waktu_lalu` (`views_admin.py:43`) sebagai OOP. Itu fungsi pembantu.

### 10. Kelola Buku

**Di website:** form tambah buku (judul, penulis, kategori, tahun, ISBN, cover, deskripsi, PDF), tabel buku, edit, hapus, dan hapus massal.

**Yang benar-benar OOP di `kelola_buku` (`views_admin.py:208`):**

| Konsep | Bukti |
|---|---|
| Pembuatan object | `BukuDigital.objects.create(...)` (`views_admin.py:232`), dengan atribut warisan dan atribut khusus (`isbn`, `link_unduh`, `ukuran_file`, `format_file`, `deskripsi`) |
| Pewarisan | Bandingkan dengan `Jurnal.objects.create` (`views_admin.py:353`). Atribut dasar sama, tetapi jurnal punya `issn`, `volume`, `bidang_ilmu`, `link_akses`. Bisa didemokan dari dua form di UI |
| Enkapsulasi | `buku.kategori_label` di teks notifikasi (`views_admin.py:248`) dan tabel (`kelola_buku.html:67`) |
| Method warisan | `edit_buku` (`views_admin.py:278`) memanggil `buku.save()` (baris 306). `hapus_buku` memanggil `buku.delete()` (baris 274) |
| Asosiasi | Satu `Notifikasi` per `CustomUser` anggota (`views_admin.py:247`). `Peminjaman.objects.filter(buku_digital=b).count()` (`views_admin.py:261`) |

**Kalimat presentasi:** "Admin membuat object `BukuDigital`, subclass `ItemKatalog`. Field umum diwarisi, ISBN khusus buku. Edit dan hapus memakai `save()` dan `delete()` warisan dari `Model`."

**Jangan klaim:** `b.total_pembaca` (`views_admin.py:261`) sebagai enkapsulasi. Itu atribut sementara yang ditempel saat runtime. `@user_passes_test` adalah decorator fungsi.

### 11. Kelola Jurnal

**Di website:** form tambah jurnal (judul, penulis, bidang ilmu, volume, tahun, cover, link akses, ISSN, PDF), tabel, edit, hapus, hapus massal.

**Yang benar-benar OOP di `kelola_jurnal` (`views_admin.py:326`):**

| Konsep | Bukti |
|---|---|
| Pembuatan object | `Jurnal.objects.create(...)` (`views_admin.py:353`) |
| Pewarisan | Field dasar sama dengan buku. Field khusus: `issn`, `volume`, `bidang_ilmu`, `link_akses` |
| Method warisan | `jurnal.save()` (`views_admin.py:448`), `jurnal.delete()` (`views_admin.py:414`), `qs.delete()` untuk hapus massal (`views_admin.py:458`) |
| Asosiasi | `Notifikasi.objects.create(...)` per anggota (`views_admin.py:364-365`). `Peminjaman.objects.filter(jurnal=j).count()` (`views_admin.py:379`) |

**Kalimat presentasi:** "Jurnal dan buku berbagi induk `ItemKatalog`, jadi form admin keduanya punya field dasar yang sama, dan hanya field khususnya yang berbeda."

**Jangan klaim:** `j.total_pembaca` (`views_admin.py:379`) sebagai atribut class.

### 12. Kelola Achievement

**Di website:** form tambah achievement (nama, kategori Umum/Langka/Epik, deskripsi, jenis syarat, target, gambar terkunci dan terbuka), daftar, edit, hapus, hapus massal.

**Ini fitur dengan OOP paling kaya di sisi admin.** Class `AchievementForm` ada di `views_admin.py:471`:

```python
class AchievementForm(forms.ModelForm):
    class Meta:
        model = AchievementGambar
        fields = ["nama", "kategori", "deskripsi", "tipe_syarat", "target",
                  "gambar_terkunci", "gambar_terbuka"]

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.fields["nama"].error_messages["required"] = "wajib diisi."
        ...

    def clean_nama(self):
        nama = self.cleaned_data["nama"].strip()
        ada = AchievementGambar.objects.filter(nama__iexact=nama)
        ...

    def _cek_ukuran(self, field):
        berkas = self.cleaned_data.get(field)
        ...

    def clean_gambar_terkunci(self):
        return self._cek_ukuran("gambar_terkunci")
```

| Konsep | Bukti |
|---|---|
| Pewarisan | `AchievementForm(forms.ModelForm)` (`views_admin.py:471`) |
| Class bersarang | `class Meta` menentukan `model = AchievementGambar` dan daftar `fields` |
| Overriding | `__init__` ditimpa dan memanggil `super().__init__(...)` untuk mengubah pesan error |
| Enkapsulasi | Validasi ada di dalam class: `clean_nama`, `clean_target`, `clean_gambar_terkunci`, `clean_gambar_terbuka`. Helper `_cek_ukuran` berawalan `_` (protected menurut konvensi Python) |
| Method warisan | `form.is_valid()` (`views_admin.py:543`), `form.save()` (`views_admin.py:544, 583`), `ach.delete()` (`views_admin.py:600`) |
| Property | `a.ikon_tampil` (`models.py:360`) di `kelola_achievement.html` |
| Keterkaitan class | `AchievementForm(request.POST, request.FILES, instance=ach)` (`views_admin.py:581`) mengikat form ke object `AchievementGambar` |

**Kalimat presentasi:** "Form achievement mewarisi `ModelForm`, meng-override `__init__` dengan `super()`, dan menyimpan aturan validasi di dalam class-nya sendiri."

**Jangan klaim:** `clean_*` sebagai overriding. Itu *hook* yang dipanggil Django berdasarkan nama. Overriding yang benar hanya `__init__`. `_galat_form` dan `_hapus_berkas` adalah fungsi biasa.

### 13. Kelola Anggota

**Di website:** daftar anggota (aktif/nonaktif), persetujuan akun baru, tambah anggota, edit, hapus, aktifkan.

**Yang benar-benar OOP di `kelola_anggota` (`views_admin.py:673`):**

| Konsep | Bukti |
|---|---|
| Instansiasi | `CustomUser(...)` dibuat langsung (`views_admin.py:689`) |
| Pewarisan | `user.set_password(password)` (`views_admin.py:696`). `is_active` diwarisi. `menunggu_persetujuan` milik `CustomUser` |
| Method warisan | `user.save(update_fields=["is_active", "menunggu_persetujuan"])` (`views_admin.py:723`), `user.delete()` (`views_admin.py:744`) |
| Enkapsulasi | `user.nama_asli` (`views_admin.py:733, 743`) |
| Asosiasi | `Notifikasi.objects.create(...)` ke user saat disetujui (`views_admin.py:724`) |

**Kalimat presentasi:** "Persetujuan anggota hanya mengubah state object `CustomUser` lewat `save()` dan membuat object `Notifikasi` yang terhubung ke user itu."

**Jangan klaim:** `a.jumlah_pinjaman` (`views_admin.py:705`) sebagai atribut class. Itu ditempel saat runtime.

### 14. Peminjaman & Pengembalian (admin)

**Di website:** tabel pinjaman (anggota, buku, tanggal dipinjam, jatuh tempo, status, sudah dikembalikan) dengan filter.

**Yang benar-benar OOP di `peminjaman_admin` (`views_admin.py:895`):**

| Konsep | Bukti |
|---|---|
| Asosiasi | `Peminjaman.objects.select_related(_USER_FIELD, "buku_digital", "jurnal")` (`views_admin.py:899`). Template: `p.pengguna.nama_asli`, `p.item.judul` |
| Pewarisan | `p.item = p.buku_digital or p.jurnal` (`views_admin.py:905`), lalu `p.item.judul` dan `p.item.penulis` memakai atribut warisan `ItemKatalog` |
| Enkapsulasi | `nama_asli` pada anggota |

**Kalimat presentasi:** "Satu object `Peminjaman` terhubung ke user dan ke buku atau jurnal, lalu kami membaca judulnya lewat atribut warisan `ItemKatalog`."

**Jangan klaim:** `p.is_kembali` dan `p.is_telat` (`views_admin.py:906-912`) sebagai atribut class. `_hitung_jatuh_tempo` (`views_admin.py:845`) adalah fungsi. `buku_digital or jurnal` bukan polimorfisme.

### 15. Laporan

**Di website:** kartu statistik (anggota aktif, buku dan jurnal tersedia, peminjaman bulan ini), "Buku per kategori", dan unduh CSV.

**OOP di halaman ini sedikit, jangan dipaksakan:**

| Konsep | Bukti |
|---|---|
| Class dan object | `CustomUser.objects.filter(...)` (`views_admin.py:929`), `Peminjaman.objects.filter(...)` (`views_admin.py:936, 945`), `Jurnal.objects.count()` dan `BukuDigital.objects.count()` (`views_admin.py:973, 976`) |
| Object library | `HttpResponse(content_type="text/csv")` dan `csv.writer(response)` (`views_admin.py:988-992`) |

**Kalimat presentasi:** "Laporan mengagregasi object model dan menghasilkan CSV memakai object `HttpResponse` dan `csv.writer`."

**Jangan klaim:** pewarisan, enkapsulasi, atau polimorfisme di Laporan.

---

## FITUR PENDUKUNG

| Fitur | OOP yang sah |
|---|---|
| Pinjam buku | `Peminjaman.objects.create(user=user, buku_digital=buku)` (`views.py:661`). Aturan batas pakai atribut `user.max_loans` (`views.py:652`) |
| Baca buku dan progres | `get_or_create` (`views.py:602`) dan `pinjam.save(update_fields=["progress_baca"])` (`views.py:636`) |
| Waktu baca dan streak | Enkapsulasi: `user.reset_streak_if_expired()` (`views.py:1287`) dan `user.update_streak()` (`views.py:1311`) |
| Pengembalian dan perpanjang | Asosiasi `Peminjaman`. Object diambil, diubah, lalu `save()` (`views.py:821-828, 858`) |
| Riwayat | `Peminjaman.objects.filter(user=...)` (`views.py:917`) |
| Notifikasi | Asosiasi `Notifikasi` → `CustomUser`. Class `Meta` bersarang (`models.py:170`). `item.save(update_fields=["dibaca"])` (`views.py:258`) |
| Achievement (user) | Object `AchievementGambar` dibaca atributnya (`views.py:1161`). **OOP minim**: logika syarat ada di fungsi `_achievements` |
| Logout | Tidak ada OOP |

Contoh pola "object diambil, diubah, lalu disimpan" di pengembalian (`views.py:819-828`):

```python
pinjaman = get_object_or_404(Peminjaman, id=pinjaman_id, user=request.user)
if pinjaman.status == "Dibaca":
    pinjaman.status = "Selesai"
    pinjaman.progress_baca = 100
    pinjaman.tanggal_kembali = timezone.now()
    pinjaman.save()
```

### Ringkasan per halaman

| Halaman | Konsep OOP yang sah |
|---|---|
| Login | Pewarisan (`CustomUser` ← `AbstractUser`), object |
| Register | Pembuatan object, pewarisan (`create_user`) |
| Dashboard | Enkapsulasi (`reset_streak_if_expired`, `kategori_label`), asosiasi |
| Katalog Buku | Pewarisan, enkapsulasi, asosiasi, pembuatan object |
| Jurnal | Pewarisan, object |
| Bookmarks | Asosiasi, method warisan, enkapsulasi |
| Top Reader | Object `CustomUser`, enkapsulasi (`nama_asli`) |
| Profil | Pewarisan (`set_password`), enkapsulasi (`update_streak`) |
| Dashboard Admin | Asosiasi (relasi balik), pewarisan atribut |
| Kelola Buku / Jurnal | Pewarisan, pembuatan object, method warisan, asosiasi |
| Kelola Achievement | Pewarisan, overriding `__init__`, class bersarang, enkapsulasi validasi |
| Kelola Anggota | Instansiasi, method warisan, asosiasi |
| Peminjaman Admin | Asosiasi, pewarisan atribut |
| Laporan | Object (OOP minim) |

### Pilar paling kuat untuk presentasi

1. **Pewarisan:** `ItemKatalog` → `BukuDigital`/`Jurnal`, `CustomUser` ← `AbstractUser`, `AchievementForm` ← `ModelForm`.
2. **Abstraksi:** `ItemKatalog` dengan `Meta.abstract = True`.
3. **Enkapsulasi:** `nama_asli`, `kategori_label`, `ikon_tampil`, `update_streak()`, dan validasi di `AchievementForm`.
4. **Polimorfisme:** `__str__` yang di-override, `Pengaturan.save()`, dan `AchievementForm.__init__`.
5. **Asosiasi:** `Peminjaman`, `Bookmark`, dan `Notifikasi` yang terhubung ke `CustomUser`.

---

# BAGIAN B. Tambahkan satu baris ke tabel 4.2 (Inheritance)

| Subclass | Induk | Yang diwarisi | Yang ditambah |
|---|---|---|---|
| `AchievementForm` (`views_admin.py:471`) | `forms.ModelForm` (Django) | `is_valid()`, `save()`, `cleaned_data`, mekanisme validasi | class `Meta`, `__init__` yang ditimpa, `clean_nama`, `clean_target`, `clean_gambar_*`, helper `_cek_ukuran` |

---

# BAGIAN C. Tambahan untuk "7. Catatan Jujur"

**Ganti poin 8** dengan:

8. **Sudah diperiksa:** `views_admin.py` (seluruh view admin) dan `views.py` (alur login, register, dashboard, katalog, jurnal, bookmark, leaderboard, profil, peminjaman, notifikasi, achievement). **Belum dijelaskan baris per baris:** isi template `katalog.html` dan `peminjaman.html`, `baca_buku`, `simpan_progress`, dan nilai konstanta `ITEM_ID_RE`, `MASA_PINJAM_HARI`, `MAKS_PERPANJANG`, `RING_DETIK_PENUH`, `RING_EXP`. Nilai `data-total="120"` di `baca.html` juga perlu dicek artinya di `baca-ring.js`.

**Tambahkan poin 9 dan seterusnya:**

9. **`add_exp()` dan `Pengaturan.muat()` tidak dipanggil di view mana pun.** Sudah dicari di seluruh file `.py` selain migrasi. EXP benar-benar ditambah langsung di `waktu_baca` (`views.py:1299`, `user.exp += exp_gain`). Jika penguji bertanya, jawab bahwa kedua method itu disiapkan sebagai antarmuka resmi tetapi belum dipakai. Komentar di `views.py:919` menyebut "classmethod Pengaturan", padahal kodenya tidak memanggilnya.
10. **`Achievement` dan `UserAchievement` hanya terdaftar di `admin.py`.** Tidak ada view yang memakainya. Yang aktif di fitur achievement adalah `AchievementGambar` dan fungsi `_achievements` (`views.py:1075`). Contoh komposisi `UserAchievement` di bagian 4.5 sah di tingkat model, tetapi jangan dikatakan "dipakai di fitur".
11. **View ganda yang tidak dirutekan.** `dashboard_admin` dan `kelola_buku_admin` di `views.py:1363-1380` tidak dipakai karena `urls.py` mengarahkan ke `views_admin.py`. Aman dihapus sebelum pengumpulan.
12. **`kembalikan_buku` tidak memakai `@require_POST`** (`views.py:819-820`), berbeda dengan `perpanjang_buku`. Ini mengonfirmasi jawaban nomor 52. Method itu juga tidak menambah EXP, sehingga cocok dengan poin 1.
13. **Atribut sementara bukan bagian class.** `total_pembaca`, `jumlah_pinjaman`, `is_kembali`, `is_telat`, `p.item`, dan `jatuh_tempo` ditempel ke object saat runtime. Jangan dihitung sebagai enkapsulasi.
14. **Atribut private hanya konvensi.** Tidak ada atribut `_x` atau `__x` di `models.py`. Method berawalan `_` yang benar-benar ada di class adalah `_cek_ukuran` (`AchievementForm`) dan `_preview` serta `_ikon` (`admin.py`).
15. **`clean_*` di `AchievementForm` bukan overriding.** Itu hook yang dipanggil Django berdasarkan nama. Overriding yang sah hanya `__init__`.

---

# BAGIAN D. Tambahan untuk "6. Pertanyaan dan Jawaban"

### I. Pertanyaan per fitur website

**65. OOP apa yang ada di halaman Login?**
Pewarisan. `authenticate()` mengembalikan object `CustomUser`, dan `is_active`, `is_staff`, serta `check_password()` diwarisi dari `AbstractUser`. Atribut `menunggu_persetujuan` ditambahkan oleh `CustomUser`. `login_view` sendiri adalah fungsi biasa.

**66. OOP apa yang ada di Katalog Buku?**
Setiap buku lokal adalah object `BukuDigital` yang mewarisi `ItemKatalog` (`judul`, `penulis`, `cover`). Label kategori diambil lewat property `kategori_label` (enkapsulasi). Status bookmark memakai asosiasi `Bookmark`. Buku Open Library menjadi object lewat `BukuDigital.objects.create` di `_pastikan_buku`.

**67. OOP apa yang ada di Kelola Buku admin?**
Pembuatan object `BukuDigital` dengan atribut warisan dan khusus (`isbn`), `save()` dan `delete()` warisan dari `Model` untuk edit dan hapus, `kategori_label` untuk label, dan asosiasi saat membuat `Notifikasi` untuk setiap anggota.

**68. Fitur mana yang paling kaya OOP di sisi admin?**
Kelola Achievement. `AchievementForm` mewarisi `forms.ModelForm`, memakai class `Meta` bersarang, meng-override `__init__` dengan `super()`, dan menyimpan aturan validasi di dalam class.

**69. Apakah `clean_nama()` di `AchievementForm` termasuk overriding?**
Tidak. `ModelForm` tidak punya method itu. Django memanggil method berawalan `clean_` berdasarkan nama field. Itu hook. Overriding yang sah adalah `__init__`.

**70. Di fitur mana enkapsulasi terlihat paling jelas?**
`nama_asli` (dipakai di dashboard, top reader, profil, dan admin), `kategori_label` (katalog, bookmark, kelola buku), serta `update_streak()` dan `reset_streak_if_expired()` yang dipanggil dari dashboard dan `waktu_baca`.

**71. Di fitur mana asosiasi terlihat jelas?**
Bookmarks (`mark.buku_digital.judul`), Peminjaman (`p.item.judul`), dan Notifikasi (satu `Notifikasi` per `CustomUser`).

**72. Halaman mana yang OOP-nya paling sedikit, dan bagaimana menjelaskannya?**
Laporan, Login, dan Register. Jawab jujur: halaman itu terutama memakai object model lewat ORM, dan OOP-nya ada di class yang dipakainya (`CustomUser`, `Peminjaman`), bukan di view-nya.

---

# BAGIAN E. Ganti kotak "Cakupan" di awal README

> **Cakupan.** README ini menjelaskan `config/urls.py`, `perpustakaan/urls.py`, `models.py`, `admin.py`, `tests.py`, `views.py`, `views_admin.py`, serta template `dashboard.html`, `baca.html`, dan `kelola_buku.html`. Pemetaan OOP per fitur website ada di bagian 4.8. Bagian yang belum dijelaskan baris per baris ditandai di bagian 7.
