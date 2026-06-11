# Context — routecauseanalisysmikrotik

## Tujuan Project
Root cause analysis traffic drop pada MikroTik MGW2-CYB1 yang terjadi pada 2026-06-10 jam 20:25 WIB dan recovery 21:17 WIB (durasi ±50 menit).

## Devices yang Diinvestigasi

| Device | Hostname | IP | Model | Akses |
|---|---|---|---|---|
| MikroTik | MGW2-CYB1 | 150.242.176.185 | CCR2116-12G-4S+ | SSH langsung |
| Juniper Router | IGW1-CYB1 | 150.242.176.161 | MX240 | SSH langsung |
| Juniper Switch | CSW1-CYB1 | 172.30.0.9 | QFX5120-48Y-8C | SSH via jump IGW1 |
| NMS | Observium | nms.nsc.id | - | Web (user: plm) |

## Topologi (Lengkap)

```
Internet (IIX, International Transit, IPTV)
    │ ← koneksi FISIK langsung ke CSW
    ▼
┌───────────────────────────────────┐
│  CSW1-CYB1 (QFX5120, 172.30.0.9)     │
│  Core Switch — semua colok ke sini    │
└───────────────────────────────────┘
    │           │           │
    │xe-0/0/33  │xe-0/0/30  │xe-0/0/31...
    │⚠️FLAP     │           │
    ▼           ▼           ▼
IGW1-CYB1   MGW2-CYB1   MGW-CYB-1 ...
(Juniper    (MikroTik)  (MikroTik)
MX240)          │
 BGP router     ▼
            Starlink
                │
                ▼
            Customer
```

> **Semua device fisik terhubung ke CSW1-CYB1.**
> IGW bukan "di atas" CSW, tapi colok ke CSW sebagai router BGP.
> Ketika link CSW ↔ IGW (xe-0/0/33) flap → routing dari IGW terganggu → traffic tidak bisa keluar ke internet.

## BGP Peers IGW1-CYB1

| Peer | AS | Tipe |
|---|---|---|
| 10.9.32.1 | 150930 | Direct Peering IPTV |
| 113.59.234.208 | 45296 | International Transit |
| 123.108.8.111 | 7597 | Domestic (IIX) |
| 123.108.9.111 | 7597 | Domestic (IIX) |
| 150.242.176.182 | 152069 | Internal |

## Root Cause — TERKONFIRMASI

**Link flap antara CSW1-CYB1 (xe-0/0/33) dan IGW1-CYB1 (xe-2/0/2)**

### Evidence:
| Evidence | Data |
|---|---|
| Output drops xe-0/0/33 (switch→Juniper) | **24,040,979** packet |
| Carrier transitions xe-0/0/33 | **11x** link naik-turun |
| MTU mismatch | Switch 9000 vs Juniper 9216 |
| MTU errors di Juniper xe-2/0/2 | **5,677,146** |
| Semua interface drop serentak 20:25 WIB | Terkonfirmasi Observium |
| CPU & Memory normal | Bukan overload |
| Tidak ada config change 10 Juni | Bukan human error |

### Alur:
```
Link CSW1 xe-0/0/33 ↔ IGW1 xe-2/0/2 flap
    ↓
Semua sub-interface Juniper terdampak (IIX, Intl, IPTV, Uplink)
    ↓
Downstream MikroTik ikut drop (logis: upstream semua terdampak)
    ↓
Traffic pelanggan turun ±50 menit (20:25-21:17 WIB)
```

## Status Investigasi
- [x] Investigasi MikroTik selesai
- [x] Investigasi Juniper IGW1-CYB1 selesai
- [x] Investigasi Switch CSW1-CYB1 selesai
- [x] Konfirmasi dari Observium NMS (traffic drop visual)
- [x] Root cause terkonfirmasi
- [x] Dokumentasi & push GitHub selesai
- [ ] Fix MTU mismatch (switch 9000 vs Juniper 9216)
- [ ] Nonaktifkan radius debug logging di MikroTik
- [ ] Setup syslog eksternal
- [ ] Cek fisik SFP/kabel xe-0/0/33 di CSW1-CYB1

## Log Permintaan
- [2026-06-10] Inisialisasi investigasi, SSH ke MikroTik berhasil
- [2026-06-10] Investigasi MikroTik: interface, log, VRRP, routing, queue
- [2026-06-10] VRRP normal by design (bukan anomali)
- [2026-06-11] SSH ke Juniper IGW1-CYB1 berhasil
- [2026-06-11] Temuan BGP IIX 525 flaps (hipotesis awal, kemudian direvisi)
- [2026-06-11] Akses Observium NMS berhasil, traffic drop visual terkonfirmasi
- [2026-06-11] Semua interface drop serentak → bukan BGP partial, masalah physical
- [2026-06-11] CPU/Memory normal → overload gugur
- [2026-06-11] SSH ke CSW1-CYB1 (via jump IGW1) berhasil
- [2026-06-11] TERKONFIRMASI: xe-0/0/33 drops 24jt, carrier transitions 11x, MTU mismatch
- [2026-06-11] Starlink tidak ada gangguan (konfirmasi user)
- [2026-06-11] Push semua evidence ke GitHub: dhomanhuri/RCA-Incident-100626
