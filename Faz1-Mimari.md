# FAZ 1: Active Directory Temel Mimarisi ve Mantıksal Yapı

Bu bölümde Active Directory'nin (AD) hiyerarşik yapısını, sınırlarını ve bileşenlerini inceliyoruz. Hedefimiz sadece sistemi kurmak değil, "saldırgan nereyi hedefler, savunmacı nereyi korur" vizyonuyla büyük resmi ve mantıksal sınırları anlamaktır.

---

## 1. Hiyerarşik Yapı: Domain, Tree ve Forest

### Domain (Alan Adı)
**AD Üzerindeki Yeri:** Active Directory mimarisinin en temel yönetim birimidir. Kullanıcı hesapları, bilgisayarlar, yazıcılar ve güvenlik politikaları (GPO) bu sınırın içinde tanımlanır ve yönetilir. Her domain kendi mantıksal sınırlarına sahiptir.
**Örnek:** Yeni bir şirket kurduğumuzu düşünelim: `sirket.local`. Bu isim bizim ana kök (root) domain'imizdir. Tüm çalışanlarımız, istemci (client) bilgisayarlarımız ve sunucularımız bunun içindedir.

> *[Görsel Gelecek: İzole bir domain mimarisi veya lab kurulumundaki sirket.local ekran görüntüsü]*


### Tree (Ağaç)
**AD Üzerindeki Yeri:** Aynı isim alanını (contiguous namespace) paylaşan ve birbirine hiyerarşik olarak çift yönlü güven (trust) ilişkisiyle bağlı olan domain'ler topluluğudur. 
**Örnek:** `sirket.local` ana şirketimiz büyüdü ve farklı departmanlar (veya farklı coğrafi lokasyonlar) için yeni alt domain'lere (Child Domain) ihtiyaç duyduk: `IT.sirket.local` ve `kalite.sirket.local`. Aynı kökten türedikleri için bu kök ve dalların oluşturduğu bütün yapıya "Tree" diyoruz.

> *[Görsel Gelecek: Root Domain ve Child Domain'leri birbirine bağlayan ağaç şeması]*


### Forest (Orman)
**AD Üzerindeki Yeri:** Active Directory'nin en üst düzey kapsayıcısı ve **mutlak güvenlik sınırıdır.** Bir veya birden fazla Tree'yi (Ağaç) içinde barındırır.
**Örnek:** Holdingimiz devasa bir büyüme gerçekleştirdi ve `farklibirfirma.local` adında bambaşka bir yapıyı satın aldı. İsimleri tamamen farklı olmasına rağmen, merkez sistemlerimizin birbiriyle konuşabilmesi için bu iki ayrı ağacı içine alan en büyük dış kapsayıcıya "Forest" diyoruz. 
**Güvenlik Vizyonu:** Bir saldırgan herhangi bir alt domain'de (Child Domain) hak elde ettiğinde zararı o domain ile sınırlı tutabiliriz. Ancak saldırgan yatay hareketler ile Forest yetkilerini (Enterprise Admin) ele geçirirse, içerideki tüm domain ve tree yapıları saniyeler içinde düşer. Ormanın anahtarı, krallığın anahtarıdır.

> *[Görsel Gelecek: Farklı Tree'leri barındıran Forest mimarisi]*

---

## 2. Sunucu Rolleri: Domain Controller (DC) vs. Member Server

Active Directory ortamında sunucular, üstlendikleri rollere ve barındırdıkları verilere göre kesin sınırlarla birbirlerinden ayrılırlar. Bir güvenlik mühendisi için bu ayrım, saldırı yüzeyini (attack surface) daraltmanın en temel adımıdır.

### Domain Controller (DC) - Ağın Merkezi Otoritesi
Sıradan bir Windows Server işletim sistemine **Active Directory Domain Services (AD DS)** rolü kurulup terfi (Promote) ettirildiğinde, o sunucu artık bir Domain Controller (DC) olur. Ağdaki tüm kimlik doğrulama, yetkilendirme ve politika yönetim süreçlerinin kalbidir.

**DC'nin Teknik Anatomisi ve Temel Bileşenleri:**
*   **NTDS.dit Veritabanı:** Active Directory'nin kalbidir. Kullanıcı hesapları, bilgisayar nesneleri ve en önemlisi parola özetleri (Hash) bu dosyanın içinde tutulur. Kapatılmış veya izole edilmiş (Exclusive) bir veritabanı motoru (ESENT) üzerinde çalışır.
*   **SYSVOL Klasörü:** Ağdaki tüm Group Policy Object (GPO) dosyalarının ve giriş/çıkış (Logon/Logoff) scriptlerinin tutulduğu paylaşımlı klasördür. Tüm DC'ler arasında (DFSR protokolü ile) senkronize edilir.
*   **KDC (Key Distribution Center):** Kerberos kimlik doğrulama protokolünün ana aktörüdür. Kullanıcılara ağda işlem yapabilmeleri için bilet (TGT - Ticket Granting Ticket) dağıtan servis doğrudan DC üzerinde çalışır.

> *[Görsel Gelecek: Server Manager üzerinden AD DS kurulum ekranı veya NTDS.dit dosyasının dizin görüntüsü]*

**Neden Çoklu Domain Controller Mimarisi Kullanılır?**
Kurumsal yapılarda **asla tek bir DC kullanılmaz.** En az iki (veya lokasyonlara göre daha fazla) DC konumlandırılır. 
1.  **Yüksek Erişilebilirlik (High Availability) & Hata Toleransı:** Bir DC donanımsal olarak çökerse, ortamdaki diğer DC (Additional DC) anında kimlik doğrulamaya devam eder. Sistemde "Tek Nokta Hatası" (Single Point of Failure) oluşması engellenir.
2.  **Yük Dengeleme (Load Balancing):** Sabah mesai başlangıcında binlerce kullanıcının aynı anda ağa giriş yapması (Logon storm), tek bir sunucuyu kilitleyebilir. Çoklu DC'ler bu yükü aralarında paylaşır.
3.  **Replikasyon (Replication):** DC'ler, `NTDS.dit` ve `SYSVOL` verilerini KCC (Knowledge Consistency Checker) adı verilen bir servis yardımıyla sürekli olarak birbirleriyle eşitler. 

### Member Server (Üye Sunucu) - İzolasyon ve Servis Yönetimi
Domain'e (Örn: `sirket.local`) dahil edilmiş, merkezden (DC üzerinden) yönetilen ancak üzerinde **AD DS rolü (Domain Controller yetkisi) BULUNMAYAN** Windows Server işletim sistemleridir. 

**Neden Servisleri DC Üzerine Değil de Member Server'a Kurarız? (Güvenlik Perspektifi)**
*   **Saldırı Yüzeyini Daraltma:** IIS (Web Server), SQL Server, File Server veya Exchange Server gibi uygulamalar kendi içlerinde zafiyetler barındırabilir. Eğer bir SQL sunucusunu DC ile aynı makineye kurarsak, SQL üzerinden içeri sızan bir saldırgan doğrudan şirketin tüm parola özetlerine (`NTDS.dit`) ulaşır.
*   **Tier İzolasyonu:** Güvenlik mimarisinde DC'ler "Tier 0" (En yüksek güvenlikli) varlıklardır. Member Server'lar ise barındırdıkları servislere göre "Tier 1" veya "Tier 2" olarak sınıflandırılırlar. Member Server'lar hacklense bile, doğru bir mimaride saldırganın Domain Controller'a (Tier 0) yatay geçiş yapması engellenmiş olur.
*   **Performans İzolasyonu:** Veritabanı gibi CPU ve RAM'i yoğun tüketen servisler, DC ile aynı makinede olursa kimlik doğrulama süreçlerini yavaşlatır.

> *[Görsel Gelecek: Laboratuvarımızda DC ve Member Server'ın birbirine bağlı ama izole çalıştığını gösteren mimari şema]*

---

## 3. FSMO Rolleri (Flexible Single Master Operations)

Ortamda birden fazla Domain Controller olduğunda, tüm DC'ler kullanıcı açma/kapatma gibi işlemleri yapabilir. Ancak bazı kritik işlemler vardır ki, ağda aynı anda iki sunucu bunu yapmaya kalkarsa veritabanı bozulur. Çatışmayı önlemek için bu özel görevler sadece belirli DC'lere atanır. Bu rollere **FSMO Rolleri** denir.

FSMO rolleri 5 tanedir. İkisi tüm ormanı kapsar, üçü ise sadece bulunduğu domain'i kapsar.

### Forest Bazlı Roller (Tüm ormanda sadece 1 tane bulunur)

**1. Schema Master (Şema Yöneticisi)**
*   **Ne İşe Yarar?** Active Directory'nin veritabanı şemasını (nesne sınıfları ve nitelikleri) değiştirme yetkisine sahip tek roldür. Örneğin, ortama bir Microsoft Exchange (Mail) sunucusu kurduğunuzda, kullanıcıların özelliklerine "E-posta Adresi" gibi yeni alanlar eklenmesi gerekir. Bu güncellemeyi sadece Schema Master yapar.
*   **Çökerse Ne Olur?** Anlık hiçbir şey olmaz. Kullanıcılar işine devam eder. Sadece AD şemasında bir güncelleme veya yeni bir yapısal değişiklik yapamazsınız.

**2. Domain Naming Master (Etki Alanı İsimlendirme Yöneticisi)**
*   **Ne İşe Yarar?** Ormana (Forest) yeni bir Domain veya Tree ekleneceği zaman ya da mevcut bir domain silineceği zaman bu rol devreye girer. İsim çakışmalarını önler.
*   **Çökerse Ne Olur?** Mevcut sistem çalışmaya devam eder. Sadece ormana yeni bir domain ekleyemez veya çıkaramazsınız (ki bu işlem şirketlerde yıllarda bir kez yapılır).

### Domain Bazlı Roller (Her domain'de 1 tane bulunur)

**3. PDC Emulator (Primary Domain Controller) - **EN KRİTİK ROL****
*   **Ne İşe Yarar?** FSMO rollerinin kalbidir. 
    1.  **Zaman Senkronizasyonu:** Kerberos protokolü zaman farkına tahammül edemez (maksimum 5 dakika). Ağdaki tüm cihazlar saatini PDC Emulator'den alır.
    2.  **Parola Değişiklikleri:** Bir kullanıcı parolasını değiştirdiğinde veya hesabı kilitlendiğinde, bu bilgi anında PDC'ye iletilir.
    3.  **GPO Yönetimi:** Group Policy güncellemeleri varsayılan olarak bu rol üzerinden yapılır.
*   **Çökerse Ne Olur?** Sistemde ciddi krizler başlar. Saatler kayarsa Kerberos biletleri reddedilir ve kimse ağa giremez. Yanlış şifre girenlerin hesabı kilitlenmeyebilir veya yeni şifre belirleyen biri sisteme girmekte gecikmeler yaşar. SOC (Security Operations Center) alarmları patlamaya başlar.

**4. RID Master (Relative ID Master)**
*   **Ne İşe Yarar?** Active Directory'de açılan her kullanıcıya veya cihaza benzersiz bir kimlik numarası (SID - Security Identifier) verilir. (Örn: `S-1-5-21...-1001`). Bu numaranın sonundaki `1001` kısmı RID'dir. RID Master, ortamdaki diğer DC'lere 500'lük paketler halinde boş RID havuzu dağıtır ki, farklı DC'ler aynı anda aynı kimlik numarasını farklı kişilere vermesin.
*   **Çökerse Ne Olur?** Anlık bir kesinti olmaz. Diğer DC'ler ellerindeki 500'lük boş havuzu kullanmaya devam eder. Ancak ellerindeki havuz tükendiğinde yeni bir kullanıcı veya bilgisayar hesabı oluşturamazsınız.

**5. Infrastructure Master (Altyapı Yöneticisi)**
*   **Ne İşe Yarar?** Farklı domain'ler arasındaki nesne referanslarını günceller. Örneğin; `sirket.local` domain'indeki bir grubu, `ankara.sirket.local` domain'indeki bir kullanıcıyla eşleştirdiğinizde, kullanıcının adı değişirse bu güncellemeyi Infrastructure Master takip eder.
*   **Çökerse Ne Olur?** Çoklu domain yapılarında, farklı domain'lerden gruplara eklenen kullanıcıların isimleri "S-1-5-21..." gibi anlamsız SID numaraları olarak görünmeye başlar. Tek domain'li ortamlarda hiçbir etkisi yoktur.

> **Güvenlik Notu:** Sistemde bir FSMO rolüne sahip sunucu donanımsal olarak tamamen yanarsa, bu roller komut satırı (PowerShell / ntdsutil) üzerinden hayatta kalan diğer DC'ye zorla taşınabilir. Bu işleme **"Seize" (El koyma)** denir.

---
## 4. Active Directory Veritabanı (NTDS.dit)

AD'nin en büyük kasası ve siber güvenlik dünyasında bir saldırganın Domain Controller'a sızdığında ele geçirmeye çalıştığı `NTDS.dit` (New Technology Directory Services) dosyasıdır.

**NTDS.dit Nedir ve Nerede Durur?**
Active Directory ortamındaki tüm nesnelerin (kullanıcılar, bilgisayarlar, gruplar) bilgilerini barındıran fiziksel veritabanı dosyasıdır. 
*   Varsayılan olarak Domain Controller sunucusu üzerinde `C:\Windows\NTDS\` dizininde saklanır.
*   Dosya, işletim sistemi çalıştığı (DC açık olduğu) sürece kilitlidir. Yani doğrudan kopyala-yapıştır ile çalınamaz.

**Parolalar Nasıl Saklanır?**
`NTDS.dit` içerisinde hiçbir parola **düz metin olarak saklanmaz.** 
*   Parolalar, matematiksel algoritmalarla geri döndürülemez özetlere (Hash) dönüştürülür (Özellikle **NTLM Hash** formunda).
*   Ayrıca bu dosyanın kendisi de sunucunun donanımına özgü bir şifreleme anahtarı olan `SYSTEM` dosyası ile şifrelenmiştir. Yani dosyayı okumak için saldırganın hem `NTDS.dit` dosyasını hem de `SYSTEM` dosyasını çalması gerekir.

**Güvenlik Vizyonu: Hackerlar Neden Bunu İstiyor?**
Bir saldırgan `NTDS.dit` ve `SYSTEM` dosyalarını çalıp kendi bilgisayarına indirirse şu felaket senaryoları gerçekleşir:
1.  **Offline Cracking:** Şirketteki tüm çalışanların ve yöneticilerin NTLM hash'lerini kendi güçlü ekran kartlarıyla çevrimdışı olarak kırıp düz parolaları elde edebilir.
2.  **Pass-the-Hash (PtH):** Parolayı kıramasa bile, şifreli Hash'in kendisini kullanarak ağdaki diğer makinelere sanki o kullanıcıymış gibi giriş yapabilir.
3.  **Golden Ticket (Altın Bilet):** Bu dosyanın içinde `krbtgt` isimli, bilet dağıtan ana hesabın da şifresi vardır. Saldırgan bu şifreyi ele geçirirse, kendisine 10 yıl geçerli, krallıktaki her kapıyı açan sahte "Altın Biletler" basabilir.

> *[Görsel Gelecek: Lab ortamımızda C:\Windows\NTDS klasörünün içindeki ntds.dit dosyasının görünümü]*

---

## 5. İletişim Köprüleri: Trust (Güven) İlişkileri

Farklı şirketlerin (farklı Domain veya Forest'ların) birbirlerinin kaynaklarına (dosya sunucuları, yazıcılar, uygulamalar) erişebilmesi için aralarında kurdukları mantıksal ve şifreli köprülere "Trust" denir.

**Güvenin Yönü (Directionality):**
*   **One-Way Trust (Tek Yönlü Güven):** A domaini, B domainine güvenir. (A -> B). Bu şu demektir: B'deki kullanıcılar, A'nın krallığına girip oradaki kaynakları kullanabilir. Ancak A'daki kullanıcılar B'ye giremez. (Güvenen taraf kaynaklarını açar, güvenilen taraf erişim sağlar).
*   **Two-Way Trust (Çift Yönlü Güven):** A ve B birbirine güvenir (A <-> B). İki şirketin kullanıcıları da birbirlerinin açık olan kaynaklarına erişebilir.

**Güvenin Geçişliliği (Transitivity):**
*   **Transitive (Geçişli):** A ile B arasında güven var. B ile C arasında da güven var. Bu durumda A ile C **otomatik olarak** birbirine güvenir. ("Dostumun dostu, benim de dostumdur" mantığı). Active Directory'de aynı orman (Forest) içindeki tüm domain'ler varsayılan olarak çift yönlü ve geçişli (Two-way Transitive) güvene sahiptir.
*   **Non-Transitive (Geçişsiz):** A ile B arasında, B ile C arasında güven olsa bile, A ile C arasında güven YOKTUR. Özel olarak belirtilmediği sürece araya duvar çekilir.

**Güvenlik Vizyonu (Attack Paths):**
Güven ilişkileri bir siber güvenlikçi için **yatay hareket rotalarıdır.** Eğer devasa ve çok güvenli bir holding (Domain A), küçük ve güvenliği zayıf bir taşeron firmayla (Domain B) çift yönlü trust kurarsa; saldırgan önce zayıf olan B firmasını hackler, ardından bu "Güven Köprüsünü" kullanarak A holdinginin kalbine sızar (Trust Exploitation). Güven sınırları, güvenlik zafiyetlerinin en çok sızdığı noktalardır.

![Trust İlişkileri](gorsel_linki_buraya_gelecek) 
