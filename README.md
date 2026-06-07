# OTA Toolchain Analizi — new-firmware.z1

**Analiz Edilen Firmware:** `new-firmware.z1`  
**Platform:** Contiki-NG / Z1 Mote (MSP430F2617)  
**Araç Zinciri:** msp430-gcc 4.7.4, msp430-readelf, msp430-objdump, msp430-nm, msp430-size

---

## 1. Binary Kimlik Analizi

### Komut
```bash
msp430-readelf -h new-firmware.z1
```

### Çıktı
```
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 ff ...
  Class:                             ELF32
  Data:                              2's complement, little endian
  Type:                              EXEC (Executable file)
  Machine:                           Texas Instruments msp430 microcontroller
  Entry point address:               0x3100
```

### Yorum

| Alan | Değer | Anlam |
|---|---|---|
| ELF Sınıfı | ELF32 | 32-bit ELF formatı |
| Mimari | TI MSP430 | Texas Instruments MSP430 mikrodenetleyici |
| Endianness | Little-endian | Düşük anlamlı byte önce gelir |
| Tür | EXEC | Doğrudan çalıştırılabilir dosya |
| Giriş Noktası | 0x3100 | Reset sonrası CPU'nun ilk çalıştıracağı adres |
| ABI | Standalone App | İşletim sistemi olmaksızın çalışır |

**Neden "ham binary" değil "ELF executable"?**  
Ham binary (.bin) yalnızca makine kodlarını içerir; giriş noktası, bölüm bilgisi, sembol tablosu veya debug bilgisi taşımaz. ELF formatı bu meta-verilerin tamamını yapılandırılmış biçimde saklar; bu sayede linker, debugger ve flash araçları firmware'i doğru biçimde işleyebilir.

**Endianness nedir?**  
Endianness, çok-byte'lı bir sayının bellekte hangi byte'ın önce yazıldığını belirtir. MSP430 little-endian mimarisinde 0x1234 sayısı bellekte `34 12` olarak saklanır. Bu özellik, farklı platformlar arasında veri alışverişinde (OTA gibi) byte sırasına dikkat edilmesini zorunlu kılar.

**ABI nedir?**  
ABI (Application Binary Interface), derlenmiş kodun işletim sistemi veya donanımla nasıl etkileşime gireceğini tanımlar. `Standalone App` değeri, bu firmware'in bir işletim sistemi çekirdeği olmadan çalışabileceğini gösterir.

**Compiler izi ve Toolchain versiyonu:**
```bash
msp430-strings new-firmware.z1 | grep -i "gcc\|contiki"
# Çıktı: Starting Contiki-NG-release/v4.8-625-g8518cbaff-dirty
```
Firmware Contiki-NG v4.8 ile derlenmiş, `msp430-gcc 4.7.4` araç zinciri kullanılmıştır. `-dirty` son eki, resmi sürümden farklı yerel değişiklikler bulunduğuna işaret eder.

**Optimization level:** Makefile'daki `-Os` bayrağı boyut optimizasyonu uygulandığını gösterir.

---

## 2. Bellek Kullanım Analizi

### Komut
```bash
msp430-size new-firmware.z1
msp430-readelf -S new-firmware.z1
```

### Çıktı
```
   text    data     bss     dec     hex filename
  71715     336    5706   77757   12fbd new-firmware.z1
```

### Bölüm Tablosu

| Bölüm | Adres | Boyut | Tür | Anlam |
|---|---|---|---|---|
| `.text` | 0x3100 | 38766 B | Kod | Çalıştırılabilir program kodu |
| `.far.text` | 0x10000 | 19064 B | Kod | MSP430X uzak bellek bölgesindeki kod |
| `.rodata` | 0xC870 | 13821 B | Salt okunur | String sabitler, lookup tabloları |
| `.data` | 0x1100 | 336 B | RAM | Başlangıç değerli global değişkenler |
| `.bss` | 0x1250 | 5704 B | RAM | Sıfır başlangıçlı global değişkenler |
| `.noinit` | 0x2898 | 2 B | RAM | Reset'te sıfırlanmayan değişkenler |
| `.vectors` | 0xFFC0 | 64 B | Flash | Kesme vektör tablosu (32 adet × 2 B) |

### Flash, RAM, Stack, Heap Kavramları

- **Flash (ROM):** Firmware kodu ve sabit verilerin kalıcı olarak saklandığı alan. `.text`, `.rodata`, `.vectors` burada yer alır. MSP430F2617'de 64 KB Flash bulunur.
- **RAM (SRAM):** Çalışma zamanında değişkenlerin tutulduğu alan. `.data` ve `.bss` burada yer alır. MSP430F2617'de 8 KB RAM bulunur.
- **Stack:** Fonksiyon çağrılarında yerel değişkenler ve dönüş adreslerinin saklandığı alan. RAM'in üst kısmından aşağı doğru büyür.
- **Heap:** Dinamik bellek tahsisi (`malloc`) için kullanılır. Bu firmware heap kullanmamaktadır (gömülü sistemlerde yaygın).

### Bellek Yerleşim Diyagramı

```
FLASH (0x0000 - 0xFFFF) — 64 KB
┌─────────────────────────┐ 0x3100
│        .text            │ 38766 B  (çalıştırılabilir kod)
├─────────────────────────┤ 0xC870
│        .rodata          │ 13821 B  (salt okunur sabitler)
├─────────────────────────┤ 0xFFC0
│        .vectors         │    64 B  (kesme vektör tablosu)
└─────────────────────────┘ 0xFFFF

FAR FLASH (0x10000 - ...) — MSP430X uzantısı
┌─────────────────────────┐ 0x10000
│        .far.text        │ 19064 B  (uzak bellek kodu)
└─────────────────────────┘ 0x14A78

RAM (0x1100 - 0x30FF) — 8 KB
┌─────────────────────────┐ 0x1100
│         .data           │   336 B  (başlangıç değerli değişkenler)
├─────────────────────────┤ 0x1250
│         .bss            │  5704 B  (sıfır başlangıçlı değişkenler)
├─────────────────────────┤ 0x2898
│        .noinit          │     2 B
├─────────────────────────┤
│         Stack ↓         │  (kalan RAM)
└─────────────────────────┘ 0x30FF
```

**Toplam Flash kullanımı:** 71715 B / 65536 B → ⚠️ Z1 Mote'un 64KB Flash'ına **sığmıyor** (+5266 B taşıyor).  
**Toplam RAM kullanımı:** 6042 B / 8192 B → ✅ RAM'e sığıyor (%74).

> Not: Bu firmware MSP430X mimarisine yönelik `.far.text` bölümü içerdiğinden, daha büyük flash kapasiteli bir cihaz hedeflenmiştir.

---

## 3. Symbol / Function Analizi

### Komut
```bash
msp430-nm -n new-firmware.z1 | head -50
```

### Tespit Edilen Önemli Semboller

| Sembol | Adres | Tür | Anlam |
|---|---|---|---|
| `accmeter_process` | 0x1100 | D | İvmeölçer Contiki process'i |
| `cc2420_process` | 0x110C | D | CC2420 radyo sürücüsü process'i |
| `ctimer_process` | 0x1130 | D | Callback timer process'i |
| `etimer_process` | 0x113C | D | Event timer process'i |
| `hello_world_process` | 0x114A | D | Ana uygulama process'i |
| `rpl_mrhof` | 0x11A8 | D | RPL MRHOF metriği |
| `rpl_neighbors` | 0x11CA | D | RPL komşu tablosu |
| `sensors_process` | 0x11D6 | D | Sensör yönetim process'i |
| `stack_check_process` | 0x11E2 | D | Stack taşma kontrol process'i |
| `tcpip_process` | 0x11EE | D | TCP/IP yığını process'i |

**Contiki Process Entry'leri:**  
Sembol tablosundaki `_process` son ekli değişkenler, Contiki-NG'nin olay-tabanlı işlemci yapısını oluşturur. Her process bir `PROCESS_THREAD` makrosu ile tanımlanmış kooperatif görev birimidir.

**Radio Driver Fonksiyonları:**  
`cc2420_process` sembolü CC2420 2.4GHz radyo yongasının sürücüsünün firmware içinde yer aldığını göstermektedir.

**Timer Callback'leri:**  
`ctimer_list_list` ve `timerlist` sembolleri, Contiki'nin timer altyapısının bağlı liste yapısını kullandığını ortaya koyar.

---

## 4. String ve Metadata Analizi

### Komut
```bash
msp430-strings new-firmware.z1 | grep -i "contiki\|rpl\|udp\|merhaba"
```

### Tespit Edilen Önemli String'ler

```
Starting Contiki-NG-release/v4.8-625-g8518cbaff-dirty
created a new RPL DAG
failed to create a new RPL DAG
TTL value from rpl-lite: %u
RPL Lite
compression: inlined UDP ports on send side: %x, %x
IPHC: cannot compress UDP headers
```

### Yorum

- **Contiki-NG v4.8** sürümü kullanılmış, yerel değişiklikler mevcut (`-dirty`)
- **RPL Lite** yönlendirme protokolü aktif — mesh ağ yönlendirmesi için
- **UDP** ve **6LoWPAN** haberleşme katmanları firmware içinde yer alıyor
- Debug `printf` mesajları firmware içinde bırakılmış (release build değil)
- **Process isimleri:** `hello_world_process`, `sensors_process` → bu firmware bir "hello world" / sensör uygulamasıdır

---

## 7. ELF Yapısı Analizi

### Komut
```bash
msp430-readelf -l new-firmware.z1
msp430-readelf -S new-firmware.z1
```

### Program Header (Segment) Tablosu

```
Program Headers:
  Type    Offset   VirtAddr   PhysAddr   FileSiz  MemSiz   Flg
  LOAD    0x000000 0x0000300c 0x0000300c 0x09862  0x09862  R E  → .text
  LOAD    0x009864 0x0000c870 0x0000c870 0x035fd  0x035fd  R    → .rodata
  LOAD    0x00ce62 0x00001100 0x0000fe6e 0x00150  0x01798  RW   → .data + .bss
  LOAD    0x00cfb2 0x00002898 0x0000ffbe 0x00000  0x00002  RW   → .noinit
  LOAD    0x00cfb2 0x0000ffc0 0x0000ffc0 0x00040  0x00040  R E  → .vectors
  LOAD    0x00cff2 0x00010000 0x00010000 0x04a78  0x04a78  R E  → .far.text
```

**DWARF Debug Bölümleri:**  
`.debug_info`, `.debug_line`, `.debug_frame`, `.debug_str`, `.debug_loc` bölümleri mevcut. Bu, firmware'in debug sembolleriyle derlendiğini gösterir. Bu bölümler flash'a yüklenmez, yalnızca ELF dosyasında yer alır.

**Sembol Tablosu:**  
`.symtab` bölümü 4608 byte (0x1200) büyüklüğünde ve 385 sembol içermektedir. `.strtab` bölümü 16048 byte string verisi taşır.

---

## 8. Interrupt ve Donanım Analizi

### Komut
```bash
msp430-readelf -S new-firmware.z1 | grep vectors
msp430-objdump -d new-firmware.z1 | grep -i "isr\|interrupt" | head -10
```

### Kesme Vektör Tablosu

`.vectors` bölümü `0xFFC0` adresinde, 64 byte büyüklüğündedir. Bu, 32 adet 2-byte'lık kesme vektörüne karşılık gelir.

MSP430F2617 kesme vektör tablosu `0xFFC0`–`0xFFFF` adres aralığında bulunur. Her giriş, ilgili kesme tetiklendiğinde CPU'nun atlayacağı ISR (Interrupt Service Routine) adresini içerir.

**Başlangıç Adresi ile İlişki:**  
Reset vektörü `0xFFFE`–`0xFFFF` adresindedir ve giriş noktası olan `0x3100`'ı işaret eder. CPU, reset sonrasında bu adresi okuyarak `0x3100`'dan çalışmaya başlar.

**Tespit Edilen Donanım Etkileşimleri (sembol tablosundan):**
- `cc2420_arch_spi` → SPI üzerinden CC2420 radyo iletişimi
- `i2cmaster` → I2C sensör haberleşmesi
- `button_hal_button_count` → Buton girişi
- `gpio_hal_arch_*` → GPIO pin kontrolü
- `uart0` → UART seri haberleşme

---

## 20. Contiki-NG Özel Analizler

### PROCESS_THREAD Tespiti

```bash
msp430-nm -n new-firmware.z1 | grep process
```

```
00001100 D accmeter_process
0000110c D cc2420_process
00001130 D ctimer_process
0000113c D etimer_process
0000114a D hello_world_process
000011d6 D sensors_process
000011e2 D stack_check_process
000011ee D tcpip_process
```

**Protothread Yapısı:**  
Contiki-NG, PROCESS_BEGIN/PROCESS_END makroları ile gerçeklenen protothread mekanizmasını kullanır. Her process sembolü RAM'de bir `struct process` yapısı olarak tutulur (D = data bölgesi). Bu yapı process'in durumunu (çalışıyor/bekliyor) ve thread fonksiyon adresini saklar.

**Event-Driven Scheduler:**  
`etimer_process` ve `ctimer_process` sembolleri Contiki'nin olay-tabanlı zamanlayıcı altyapısını oluşturur. `tcpip_process` ağ yığınını, `sensors_process` ise sensör veri akışını yönetir.

**NETSTACK Etkileşimi:**  
String analizinde tespit edilen `RPL Lite`, `6LoWPAN`, `CSMA` mesajları firmware'in tam bir ağ yığını içerdiğini gösterir:

```
Uygulama (hello_world_process)
    ↓
UDP / IPv6
    ↓
6LoWPAN (sıkıştırma/ayrıştırma)
    ↓
CSMA MAC katmanı
    ↓
CC2420 Radyo Sürücüsü
```

---

## Genel Değerlendirme

`new-firmware.z1`, Contiki-NG v4.8 ile derlenmiş, MSP430 mimarisine yönelik bir ELF32 çalıştırılabilir dosyasıdır. Firmware; RPL mesh yönlendirmesi, 6LoWPAN sıkıştırması, UDP haberleşmesi ve CC2420 radyo sürücüsünü içeren tam bir kablosuz sensör ağı yığını barındırmaktadır.

ELF formatının ham binary'ye kıyasla tercih edilmesinin temel nedeni; giriş noktası, bölüm yerleşimi, sembol tablosu ve debug bilgisi gibi meta-verilerin yapılandırılmış biçimde taşınmasıdır. Bu sayede araç zinciri (linker, debugger, flash programlayıcı) firmware'i doğru ve güvenilir şekilde işleyebilmektedir.