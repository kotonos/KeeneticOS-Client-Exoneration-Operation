# KeeneticOS Yeniden Başlatma Sonrası Statik NAT / Port Yönlendirme Hedefinin Eski (Hayalet) IP'ye Kilitlenmesi Hatası
## Zaman Damgalı CLI/Log Kanıtlarıyla Canlı Yeniden Üretim Raporu (Gizlilik İçin Maskelenmiş)

> **Not:** Bu belgede, güvenli şekilde paylaşılabilmesi için kimliği ifşa edebilecek bilgiler (genel WAN IP adresi, KeenDNS/kayıtlı alan adları, router hostname'i ve hedef cihazın MAC adresi) yer tutucu değerlerle değiştirilmiştir. Tüm özel LAN IP'leri (`192.168.1.x`), üçüncü taraf genel IP'ler (Cloudflare kenar sunucuları, tarama yapan hostlar) ve teknik detaylar değiştirilmemiştir ve orijinal kanıtları tam olarak temsil etmektedir.

**Router:** Keenetic Extra DSL (KN-2111), hw_id `KN-2111`
**Firmware:** release `5.01.C.1.0-0`, title `5.1.1`, ndw4 `5.1.C.1.0`, bölge TR
**Raporu Hazırlayan:** Ağ sahibi / doğrudan CLI (Telnet, NDMS) üzerinden bağımsız doğrulama
**İlgili Kamuya Açık Konu:** [Keenetic Forum #26592](https://forum.keenetic.com/topic/26592-a-user-experience-keenetic-kernel-level-bug-post-soft-reload-conntracknat-chain-locks-port-forwarding-target-to-a-staleghost-ip-proven-with-arp-nat-table/)

---

## 1. Özet

MAC adresine dayalı statik port yönlendirme kuralı (`ip static tcp`), DHCP üzerinden IP rezervasyonu yapılmış bir cihazı hedeflerken, router yeniden başlatıldıktan ("soft reload") sonra trafiği **önceki, artık geçerli olmayan/süresi dolmuş bir IP adresine** yönlendirmeye devam etmektedir. Bu durum şu koşullar gerçekleşmiş olsa dahi ortaya çıkmaktadır:

- cihaz DHCP el sıkışmasını başarıyla tamamlamış ve **güncel, doğru IP adresine** atanmış/onaylanmış olsa da,
- router'ın kendi `Network::InterfaceFlusher` mekanizması boot sırasında IPv4 conntrack/route cache'ini açıkça temizlemiş olsa da.

Kural, yalnızca **manuel olarak devre dışı bırakılıp tekrar etkinleştirildiğinde** (`no ip static tcp ...` / `ip static tcp ...`) doğru hedef IP'yi yeniden çözümlemektedir; bu işlem `Network::StaticNat` modülünün kuralı yeniden kaydetmesini zorlamaktadır. Basit bir yeniden başlatma bu yeniden kaydı **tetiklememektedir**.

Bu hata, etkilenen router üzerinde talep üzerine canlı olarak yeniden üretilmiş; hata penceresi sırasında bağımsız, coğrafi olarak farklı konumlardaki gerçek dış TCP trafiğinin (Cloudflare kenar ağı + çeşitli genel tarama hostları) hayalet hedefe ulaştığı gözlemlenmiş ve **önce/sırasında/sonra** davranışsal testiyle (bağlantı zaman aşımı ile başarılı HTTP yanıtı karşılaştırması) hem arızanın hem de düzeltmenin doğruluğu teyit edilmiştir.

---

## 2. Etkilenen Yapılandırma

```
known host IoT-Device <MAC_ADRESI>
ip dhcp host <MAC_ADRESI> 192.168.1.114
ip static tcp PPPoE0 80 <MAC_ADRESI>
ip http proxy ev
    upstream http <MAC_ADRESI> 80
    domain ndns
    ssl redirect
    security-level public
    timeout 86400
ppe software
ppe hardware
```

Notlar:
- Yönlendirme hedefi statik bir IP ile değil **MAC adresi ile** tanımlanmıştır; bu da KeeneticOS'un kural uygulanırken MAC'i cihazın *o anki* DHCP kirasına çözümlemesine bağımlıdır.
- Cihazın `192.168.1.114` için bir DHCP **statik rezervasyonu** bulunmaktadır, ancak geçmişte (ve bu testte tekrar) DHCP sunucusu düzeltmeden önce kısa süreliğine eski kirası olan `.101`'i talep etmeye çalışmaktadır.
- `ppe hardware` (donanımsal NAT hızlandırma / hwnat) etkindir — ilgili gözlem için bkz. Bölüm 6.
- MAC'i bağımsız olarak hedefleyen ikinci bir mekanizma da mevcuttur: `ip http proxy ev` (KeenDNS/nginx ters proxy), bu da `ip static tcp` kuralından ayrı olarak aynı MAC'i hedeflemektedir.

---

## 3. Test Metodolojisi

1. "Önce" anlık görüntüsü alındı: `show ip neighbour`, `show ip nat tcp`.
2. Router sahibinin açık onayıyla NDMS CLI üzerinden `system reboot` (soft reload) manuel olarak tetiklendi.
3. Router yeniden çevrimiçi olduğu anda (~135–195 sn çalışma süresi) yeniden bağlanılıp anında şu bilgiler kaydedildi:
   - `show system` (çalışma süresi teyidi)
   - `show ip neighbour`
   - `show ip nat tcp`
   - `show log`
4. Gerçek davranışı gözlemlemek için router'ın genel (public) IP'sinin 80 portuna, bağımsız bir hosttan (LAN kaynaklı, WAN IP'sine hairpin/NAT-loopback yolu üzerinden) canlı bir TCP bağlantısı denendi.
5. Bilinen topluluk çözümü uygulandı: `no ip static tcp PPPoE0 80 <MAC_ADRESI>` ardından `ip static tcp PPPoE0 80 <MAC_ADRESI>`.
6. Düzeltmenin doğrulanması için aynı bağlantı testi tekrarlandı.
7. Ayrıca, özetlenmiş NAT tablosundan bağımsız olarak paket seviyesinde analiz yapılabilmesi için, router'ın **yerleşik native paket yakalama özelliği** (`monitor > capture > interface <arayüz>`, `output-directory`, `filter "tcp port 80"`, `direction in-out`) eşzamanlı olarak `PPPoE0` (WAN) ve `Home`/`Bridge0` (LAN) arayüzlerinde yapılandırılıp, ayrılmış bir ext4 USB bölümüne yazılacak şekilde ayarlandı.

---

## 4. Yeniden Başlatma Penceresinin Zaman Çizelgesi (sistem logu, yerel saat UTC+3)

```
I [Jul 18 05:37:09] ndm: Core::System::Clock: system time has been changed.
I [Jul 18 05:37:09] ndhcps: DHCPREQUEST received (STATE_SELECTING) for
                    192.168.1.101 from <MAC_ADRESI> hostname "IoT-Device".
...
I [Jul 18 05:37:11] ndm: Network::InterfaceFlusher: flushed IPv4 conntrack and
                    route cache.
...
I [Jul 18 05:37:19] ndhcps: DHCPREQUEST received (STATE_INIT) for 192.168.1.101
                    from <MAC_ADRESI> hostname "IoT-Device".
I [Jul 18 05:37:19] ndhcps: sending NAK to <MAC_ADRESI>.
I [Jul 18 05:37:19] ndhcps: DHCPDISCOVER received from <MAC_ADRESI>
                    hostname "IoT-Device".
I [Jul 18 05:37:19] ndhcps: making OFFER of 192.168.1.114 to <MAC_ADRESI>.
I [Jul 18 05:37:19] ndhcps: DHCPREQUEST received (STATE_SELECTING) for
                    192.168.1.114 from <MAC_ADRESI> hostname "IoT-Device".
I [Jul 18 05:37:20] ndhcps: sending ACK of 192.168.1.114 to <MAC_ADRESI>.
```

**Kritik gözlem:** Cihaz, *eski* kirasını (`.101`) yenilemeyi iki kez denemiş, doğru şekilde `NAK` almış ve `05:37:20`'de doğru şekilde `.114`'e yeniden atanmış/onaylanmıştır. DHCP mekanizmasının kendisi doğru çalışmıştır.

Kritik olarak, **bu boot döngüsünün log kayıtlarının hiçbir yerinde `Network::StaticNat: static NAT rule has been added` olayı görünmemektedir.** Bunu, ilgisiz bir bileşen kurulumunun tetiklediği *önceki* yeniden başlatmayla karşılaştıralım; o seferde tam olarak aynı kural, boot sırasında şu log satırını üretmişti:

```
I [Jul 18 03:57:52] ndm: Network::Nat: a NAT rule added.
I [Jul 18 03:57:52] ndm: Network::Nat: a NAT rule added.
I [Jul 18 03:57:52] ndm: Network::StaticNat: static NAT rule has been added.
```

Bu durum, `Network::StaticNat` modülünün MAC→IP eşleşmesini **her boot'ta mutlaka** yeniden kaydetmediğini/yeniden çözümlemediğini kuvvetle göstermektedir — gözlemlenen iki yeniden başlatmadan en az birinde, kural DHCP sunucusunun o anki kira tablosuna taze şekilde bağlanmak yerine, eski/serileştirilmiş bir iç durumdan devralınmıştır. Bu da, `no ip static tcp` / `ip static tcp` manuel geçişinin (ki bu kesin olarak `Network::StaticNat`'ın taze şekilde yeniden kaydolmasını zorlar) sorunu neden çözdüğünü, ancak yeniden başlatmanın bunu güvenilir şekilde neden yapmadığını açıklamaktadır.

---

## 5. Canlı NAT Tablosu Kanıtı — Hata Aktifken (yukarıdaki ACK'den saniyeler sonra kaydedilmiştir)

```
show ip neighbour   (<MAC_ADRESI>)
    address: 192.168.1.101   expired: yes   <- hayalet/geçersiz
    address: 192.168.1.114   expired: no    <- güncel, DHCP tarafından onaylanmış, leasetime 193 sn

show ip nat tcp   (ilgili satırlarla filtrelenmiştir; <WAN_IP> = router'ın genel IP'si)
TCP  172.69.63.192   9633   <WAN_IP>  80   4
     192.168.1.101   80     172.69.63.192 9633  4      <- gerçek Cloudflare kenar sunucusu, HAYALET IP'YE yönlendirildi
TCP  172.69.63.193   13722  <WAN_IP>  80   4
     192.168.1.101   80     172.69.63.193 13722 4      <- aynı durum
TCP  160.200.30.10   40733  <WAN_IP>  80   1
     192.168.1.101   80     160.200.30.10 40733 1      <- dış tarama hostu, HAYALET IP'YE yönlendirildi
TCP  160.200.32.10   40819  <WAN_IP>  80   1
     192.168.1.101   80     160.200.32.10 40819 1
TCP  160.200.38.10   40820  <WAN_IP>  80   1
     192.168.1.101   80     160.200.38.10 40820 1
```

Tam olarak aynı anda, router'ın kendi neighbour (komşu) tablosu, bu MAC için `192.168.1.114`'ün canlı ve süresi dolmamış host olduğunu (doğru şekilde) göstermektedir. **Beş bağımsız gerçek dış kaynak, doğru IP zaten onaylanmış ve aktif olduğu halde ölü bir IP'ye yönlendirilmiştir.**

### Davranışsal Doğrulama (canlı bağlantı testi)

| Durum | Test | Sonuç |
|---|---|---|
| Hata aktif | Router'ın genel IP'sine port 80'den `GET / HTTP/1.1` | **`TimeoutError`** — bağlantı hiç kurulamadı, ne RST ne veri geldi (`.101`'de sessiz bir kara delik) |
| `no ip static tcp` / `ip static tcp` geçişi uygulandıktan sonra | Aynı test tekrarlandı | **`HTTP/1.1 500 Internal Server Error`** anında döndü — paketin artık `.114`'teki canlı cihaza ulaştığını kanıtlıyor |

---

## 6. Ek Gözlem: Donanımsal NAT Hızlandırma Yolu

`Home`/`Bridge0` (LAN) üzerinde `filter "tcp port 80"` ile yakalama yapılandırılmışken, LAN'dan başlatılan ve router'ın kendi genel IP'sine giden bir bağlantı (NAT hairpin/loopback), bağlantı seviyesinde başarılı olmasına ve kısa süre sonra yazılım tarafındaki `show ip nat tcp` tablosunda da bir girdi göstermemesine rağmen, **LAN tarafı yakalamasında hiç görünmemiştir.** Gerçek dış kaynaklı trafik (yukarıdaki Cloudflare/tarama hostları) ise yazılım NAT tablosunda ve WAN tarafı yakalama dosyası büyümesinde *görünmüştür*.

Bu durum, orijinal forum konusunda da öne sürülen bir hipotezle örtüşmektedir (ancak kanıtlamamaktadır): KeeneticOS'un donanımsal NAT hızlandırması (`ppe hardware`, "hwnat") özellikle yeniden bağlanma/yeniden başlatma olayları civarında yazılım `nf_conntrack`/`StaticNat` durumundan farklılaşabilen kendine ait bir akış (flow) tablosu tutmaktadır. Native paket yakalama özelliğiyle (bkz. Bölüm 7) ek bir tam yeniden başlatma döngüsü üzerinde daha fazla araştırma yapılması bu hipotezi kesin olarak doğrulamaya veya çürütmeye yardımcı olacaktır.

---

## 7. Araç Notları (bu testi yeniden üretmek için)

- KeeneticOS'un Telnet/SSH CLI'ı (NDMS), `opkg` / `opkg-kmod-netfilter` bileşenleri kurulduktan sonra bile **ham bir Linux shell'i sunmamaktadır.** `opkg chroot`, `opkg disk` gibi komutlar, komut çalıştırma değil yapılandırma anahtarlarıdır.
- Entware/opkg gerektirmeyen **yerleşik, native bir paket yakalama özelliği** mevcuttur:
  ```
  monitor
  capture
  interface <arayüz_adı>
      output-directory <disk-etiketi>:/
      filter "<bpf-ifadesi>"
      direction in-out
      enable
  ```
  Bu, herhangi bir bağlı depolama birimine gerçek `.pcap` dosyaları (+ `.stats`) yazar ve birden fazla arayüzde eşzamanlı olarak çalıştırılabilir (örn. `PPPoE0` ve `Home`) — üçüncü taraf paket kurulumu gerektirmeden "eşzamanlı WAN/LAN tcpdump" gereksinimini doğrudan karşılamaktadır.
- Bileşen kurulumu (`components` modu, `install <isim>`, `commit`) çekirdek modülleri içeriyorsa (örn. `opkg-kmod-netfilter`) **otomatik bir router yeniden başlatmasını tetikleyebilir** — bu, bu araştırma sırasında gözlemlenen iki yeniden başlatmaya sebep olmuştur ve bu durum bilinçli olarak öngörülmeli/planlanmalıdır.

---

## 8. Sonuç

Bu araştırma, router'a özgü kanıtlarla (sistem logu + canlı NAT tablosu + gerçek dış trafik + kontrollü önce/sonra davranışsal testi) Keenetic Forum #26592 konusunda bildirilen hatayı bağımsız olarak yeniden üretmektedir: **MAC adresine dayalı statik TCP port yönlendirme kuralı, soft reboot sonrasında eski/süresi dolmuş bir DHCP kirasına bağlı kalabilmekte, kural manuel olarak kapatılıp tekrar açılana kadar dış ve iç bağlantıları sessizce kara deliğe göndermektedir.** DHCP sunucusunun kendisi ve `show ip neighbour` tablosu süreç boyunca doğru çalışmaktadır — tutarsızlık, `Network::StaticNat` kural çözümleme/önbellekleme mantığının her boot döngüsünde yenilenmemesinde izole olmaktadır.

### Keenetic Mühendisliği İçin Önerilen Düzeltme Yönleri
1. `Network::StaticNat`, her boot'ta MAC tabanlı her statik NAT kuralının hedef IP'sini, sadece kural açıkça (yeniden) yapılandırıldığında değil, o anki DHCP kira tablosuna karşı yeniden çözümlemeli (veya en azından yeniden doğrulamalı).
2. Alternatif olarak/ek olarak, `Network::StaticNat`'ın DHCP kira değişikliği/ACK olaylarına abone olması sağlanmalı; böylece sadece yapılandırma değişikliği değil, kira yenilenmesi de yeniden çözümlemeyi tetiklemeli.
3. `ppe hardware` (hwnat) akış (flow) girdilerinin, boot ve arayüz temizleme (flush) olayları sırasında yazılım `StaticNat`/`nf_conntrack` durumuyla eşzamanlı olarak geçersiz kılınıp kılınmadığı denetlenmelidir.

### Pratik Geçici Çözüm (etkinliği doğrulanmıştır)
```
no ip static tcp <arayüz> <port> <mac>
ip static tcp <arayüz> <port> <mac>
```
Keenetic bir düzeltme yayınlayana kadar bu komutları her yeniden başlatmadan sonra çalıştırın (veya bunu bir başlangıç betiği/zamanlayıcı ile otomatikleştirin).

---

## 9. Takip Talebi

Bu rapor hakkında Keenetic mühendislik ekibinden bir geri bildirim/durum güncellemesi bekliyoruz. Talep edilmesi hâlinde ek tanılama dosyaları, ham paket kayıtları (`.pcap`) sağlamaya veya ek testler yapmaya hazırız. Bu hatanın, yaygın olarak kullanılan bir yapılandırmada (MAC tabanlı statik port yönlendirme) sessiz ve tespiti zor bağlantı kesintilerine yol açtığını düşünüyoruz; bu nedenle önümüzdeki bir firmware sürümünde incelenip düzeltilmesini rica ederiz.

---
*Bu rapor, router sahibinin bilgisi ve onayıyla, 18 Temmuz 2026 tarihinde etkilenen router üzerindeki canlı CLI/Telnet oturumlarından derlenmiştir. Kamuya açık paylaşım için kimliği ifşa edebilecek detaylar maskelenmiştir — bkz. belgenin başındaki not.*
