# caddy-coraza-build

Caddy + OWASP Coraza WAF, dibangun di GitHub Actions. Tujuannya satu: menghasilkan
binary `caddy` yang **versinya sama persis** dengan yang jalan di joyboy (`v2.11.7`)
supaya Caddyfile tidak perlu diubah dan tidak ada modul standar yang hilang.

Build tidak pernah dijalankan di joyboy (host produksi, RAM 3.8 GB) maupun di laptop.

## Versi yang dipin

| Komponen | Versi | Alasan |
|---|---|---|
| Caddy | `v2.11.7` | sama dengan binary apt yang jalan di joyboy |
| coraza-caddy | `v2.6.2` | rilis stabil terbaru, go.mod-nya menargetkan Caddy 2.11.x |
| xcaddy | `v0.4.7` | tool build resmi Caddy |
| Go | `1.26.11` | Caddy 2.11.7 dan coraza 2.6.2 sama-sama `go 1.26.0` |

Catatan penting: `coraza-caddy v2.6.2` meng-`require` Caddy `v2.11.4`, sedangkan
build ini meminta `v2.11.7`. Go modules (MVS) mengambil versi tertinggi, jadi yang
dikompilasi adalah Caddy 2.11.7. Kalau suatu saat build gagal karena perubahan API
di patch release, jatuhkan `CADDY_VERSION` ke `v2.11.4` lewat input workflow.

## OWASP CRS tertanam di binary

Sejak coraza-caddy v2.6.x, Core Rule Set **dikompilasi masuk ke binary**. Tidak ada
folder rule yang perlu diunduh atau ditempel di server. Konsekuensinya:

- pakai `load_owasp_crs` di dalam blok `coraza_waf`
- rujuk rule lewat path berawalan `@`, contoh `Include @owasp_crs/*.conf`
- jangan menaruh path filesystem di depan `@`, misal `/etc/caddy/rules/@owasp_crs/*.conf` (gagal)

## Cara build

Manual, dari tab Actions → `build-caddy-coraza` → Run workflow. Atau lewat tag:

```bash
git tag v2.11.7-coraza2.6.2
git push origin v2.11.7-coraza2.6.2
```

Push tag akan sekaligus membuat release berisi `caddy`, `caddy.sha256`,
`modules.txt`, dan `build-info.txt`.

Workflow memverifikasi dirinya sendiri sebelum upload:

1. `caddy build-info` harus memuat `corazawaf/coraza-caddy/v2 v2.6.2`
2. `caddy list-modules` harus memuat modul coraza
3. modul standar yang dipakai produksi (`reverse_proxy`, `file_server`, `encode`,
   `matchers.host`, `encoders.gzip`) harus masih ada
4. `caddy validate` pada Caddyfile uji yang memuat `load_owasp_crs` +
   `Include @owasp_crs/*.conf` harus lolos. Langkah ini benar-benar memuat CRS dari
   ruleset tertanam, jadi kegagalan ruleset akan terlihat di sini, bukan di produksi.

## Deploy ke joyboy

Belum dieksekusi. Urutannya begini kalau nanti dijalankan:

```bash
# 1. ambil artifact (dari laptop, bukan dari joyboy: joyboy tidak perlu token GitHub)
gh run download <run-id> --repo adi805/caddy-coraza-build \
  --name caddy-coraza-v2.11.7-v2.6.2 --dir ./dist
sha256sum -c ./dist/caddy.sha256

# 2. bandingkan inventaris modul: tidak boleh ada modul joyboy yang hilang
ssh joyboy 'caddy list-modules --packages' > joyboy-modules.txt
diff <(sort joyboy-modules.txt) <(sort ./dist/modules.txt)

# 3. kirim binary
scp ./dist/caddy joyboy:/tmp/caddy-new

# 4. pasang tanpa melawan apt (dpkg-divert: binary apt dipindah, bukan ditimpa)
ssh joyboy 'sudo dpkg-divert --divert /usr/bin/caddy.distrib --rename /usr/bin/caddy'
ssh joyboy 'sudo install -m 0755 /tmp/caddy-new /usr/bin/caddy'
ssh joyboy 'caddy version'

# 5. validasi Caddyfile SEKARANG, sebelum reload. Kalau validate gagal, reload tidak dijalankan.
ssh joyboy 'sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile'

# 6. baru reload
ssh joyboy 'sudo systemctl reload caddy && systemctl is-active caddy'
```

Rollback, satu perintah:

```bash
ssh joyboy 'sudo dpkg-divert --remove --rename /usr/bin/caddy && sudo systemctl reload caddy'
```

### Kenapa `dpkg-divert`, bukan `cp` biasa

`/usr/bin/caddy` dimiliki paket apt. Menimpanya langsung berarti upgrade apt
berikutnya diam-diam mengembalikan binary tanpa coraza. `dpkg-divert` memindahkan
binary apt ke `/usr/bin/caddy.distrib`, jadi apt menulis ke situ dan binary kita
tetap di `/usr/bin/caddy`. Systemd tidak perlu diubah karena path-nya sama.

`caddy upgrade` **jangan dipakai** di joyboy: perintah itu mengunduh build resmi
upstream yang tidak punya modul coraza.

## Mode awal wajib DetectionOnly

WAF di depan produksi yang belum pernah diuji bisa memblokir pengguna asli. Urutan
yang benar:

1. `SecRuleEngine DetectionOnly` + paranoia level 1 (CRS default). Tidak ada yang diblokir,
   hanya tercatat.
2. Jalankan minimal seminggu, baca log rule yang kena pada trafik sah.
3. Tulis exclusion untuk false positive yang terbukti.
4. Baru naik ke `SecRuleEngine On`.

## Yang belum ada di repo ini

- Langkah deploy belum dieksekusi; repo ini hanya menghasilkan binary.
- Konfigurasi `coraza_waf` di Caddyfile joyboy belum ditulis.
- Belum ada job yang menyentuh joyboy. Build dan deploy sengaja dipisah.
