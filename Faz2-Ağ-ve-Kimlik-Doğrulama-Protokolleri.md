# FAZ 2: Ağ ve Kimlik Doğrulama Protokolleri

Bu bölümde, Active Directory'nin (AD) kimlik doğrulama, isim çözümleme ve dizin sorgulama işlemlerini nasıl gerçekleştirdiğini protokol seviyesinde inceliyoruz.

## 1. Kerberos Protokolü

Kerberos, Active Directory'nin varsayılan kimlik doğrulama mekanizmasıdır. Parolaları ağ üzerinde düz metin (cleartext) veya özet (hash) olarak iletmez. Bunun yerine, simetrik şifreleme kullanan bir **"Bilet (Ticket)"** sistemine dayanır.

### Temel Kavramlar ve İşleyiş

*   **KDC (Key Distribution Center):** Biletleri dağıtan ve doğrulayan merkezi otoritedir. AD ortamında KDC rolü doğrudan Domain Controller (DC) üzerinde çalışır (Port 88).
*   **TGT (Ticket Granting Ticket):** "Kimlik Onay Bileti"dir. Kullanıcı kimliğini doğruladığında KDC, kullanıcıya bir TGT verir. TGT, kullanıcının ağda dolaşabileceğini gösteren genel bir bilettir. Varsayılan ömrü 10 saattir.
*   **TGS (Ticket Granting Service):** "Hizmet Erişim Bileti"dir. Kullanıcı bir servise (örn. dosya sunucusu) erişmek istediğinde, TGT'sini KDC'ye gösterir ve o servise özel bir TGS bileti talep eder.

### Kimlik Doğrulama Süreci (Paket Seviyesinde)

Kerberos iletişimi 4 temel adımdan oluşur:

1.  **AS-REQ (Authentication Service Request):** İstemci, KDC'ye başvurur ve bir TGT talep eder. Bu paketin içinde kullanıcının kimliği ve parolasıyla şifrelenmiş zaman damgası (Timestamp) bulunur (Pre-Authentication).
2.  **AS-REP (Authentication Service Reply):** KDC, kendi veritabanındaki hash'i kullanarak zaman damgasını çözer. Başarılıysa, KDC istemciye bir TGT gönderir. TGT, `krbtgt` hesabının parolasıyla şifrelenmiştir.
3.  **TGS-REQ (Ticket Granting Service Request):** İstemci, bir servise erişmek için TGT'yi KDC'ye yollar ve servise özel TGS bileti ister.
4.  **TGS-REP (Ticket Granting Service Reply):** KDC, TGT'yi doğrular ve istemciye talep ettiği servise özel TGS biletini verir.

> *[Görsel Gelecek: AS-REQ, AS-REP, TGS-REQ, TGS-REP paketlerinin Wireshark görüntüsü]*

---

## 2. NTLM ve Hash Mantığı

NTLM (New Technology LAN Manager), geriye dönük uyumluluk (Legacy) için AD ortamlarında desteklenen eski bir kimlik doğrulama protokolüdür. Bilet mantığıyla değil, **Meydan Okuma/Yanıt (Challenge/Response)** mantığıyla çalışır.

### NTLM Nasıl Çalışır?

1.  **Negotiation (Pazarlık):** İstemci sunucuya iletişim isteğini iletir.
2.  **Challenge (Meydan Okuma):** Sunucu istemciye rastgele 16 byte'lık bir sayı (Challenge) gönderir.
3.  **Response (Yanıt):** İstemci, kullanıcının parolasının özeti olan **NTLM Hash**'ini kullanarak bu Challenge'ı şifreler ve sunucuya geri gönderir.
4.  **Doğrulama:** Sunucu, bu yanıtı DC'ye ileterek doğrulama talep eder. Başarılıysa erişim sağlanır.

NTLM doğrulamasında parolanın kendisi değil, parolanın özeti (NTLM Hash) kullanılır.

---

## 3. DNS'in AD İçindeki Rolü ve SRV Kayıtları

Active Directory, isim çözümlemesi ve servis keşfi için tamamen **DNS (Domain Name System)** üzerine inşa edilmiştir. Bir AD ortamında servislerin bulunabilmesi için DNS SRV (Service) kayıtları şarttır.

### SRV (Service) Kayıtları

Ağa katılan bir bilgisayar, KDC veya LDAP gibi servislerin yerini bulmak için DNS'e SRV sorguları gönderir.

*   Örnek KDC Sorgusu: `_kerberos._tcp.dc._msdcs.eren.local`
*   Örnek LDAP Sorgusu: `_ldap._tcp.dc._msdcs.eren.local`

DNS sunucusu bu sorgulara yanıt olarak, ilgili hizmeti veren sunucunun adını ve IP adresini döndürür.

> *[Görsel: Wireshark'ta yakalanan DNS (A ve SRV) sorgularının ekran görüntüsü]*

---

## 4. LDAP (Lightweight Directory Access Protocol)

LDAP, Active Directory'nin dizin sorgulama protokolüdür. Kullanıcılar, gruplar, bilgisayarlar ve nesne özellikleri (Attributes) hakkındaki tüm bilgiler bu protokol üzerinden sorgulanır (Varsayılan Port: 389).

### Açık Metin İletişim (Cleartext)

LDAP'ın varsayılan versiyonu ağ üzerindeki iletişimi şifrelemeden, açık metin (Cleartext) olarak gerçekleştirir. Bu durum, `searchRequest` (Arama İstekleri) gibi paket içeriklerinin ağ üzerinde okunabilmesine neden olur. (Güvenli iletişim için TLS/SSL kullanan LDAPS - Port 636 tercih edilmelidir).

> *[Görsel: Wireshark'ta yakalanan LDAP searchRequest paketlerinin ekran görüntüsü]*
