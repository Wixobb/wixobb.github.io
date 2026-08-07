+++
title = 'USB-C PD (Power Delivery) Protokolü'
date = 2026-07-16T11:27:03+03:00
draft = false
math = true
+++

"Telefonum maksimum 20W şarj destekliyor. 100W şarj aletine taksam alev alır mı?" sorusu güzel bir sorudur. Cevap **hayır**. Peki neden? Öğrenelim.


## Temel Fizik

Şunu hiçbir zaman unutmayın: **Gerilim uygulanır, akım ise ÇEKİLİR.** Yani 100W (20V-5A) adaptör, 5 amperlik gücünü telefona itmez. Telefondaki batarya ne talep ederse onu verir. Buradaki problemli arkadaş **voltaj**. 

Eğer adaptör, 5V ile çalışması gereken bir telefona aniden 20V gönderirse, anakarttaki **Güç Yönetim Entegresi (PMIC)** zarar görebilir. USB-C PD protokolünün birincil görevi, iki cihaz anlaşana kadar voltajı en güvenli seviyede (5V) kilitli tutmaktır.


## Sihirli Pim

Bir USB-C soketinin içine mikroskopla bakarsanız 24 adet pin görürsünüz. Güç aktarımı VBUS pinlerinden, veri (dosya) aktarımı D+/D- pinlerinden yapılır. Ancak telefonu yanmaktan kurtaran ve şarjın "akıllı" olmasını sağlayan sihirli pinler **CC1 ve CC2 (Configuration Channel)** pinleridir.


{{< figure src="pinnumber.png" title=" " >}}


Kabloyu taktığınızda süreç şöyle işler:

1. Kabloyu taktığınız an VBUS (Güç) hattında henüz yüksek voltaj yoktur. Önce fiziksel roller belirlenir.

    - Şarj aleti (DFP - Downstream Facing Port), CC pinine bir Pull-up (Rp) direnci uygular.
    - Telefon (UFP - Upstream Facing Port), CC pinine bir Pull-down (Rd) direnci uygular.

Bu dirençlerin buluşmasıyla adaptör der ki: "Karşıda bir cihaz var ve yönü belli oldu. Şimdilik ona sadece güvenli olan standart 5V gücü vereyim." Yani anlaşma sağlanana kadar telefonunuz 5V bariyeri ile donanımsal olarak koruma altındadır.

2. Analog olarak cihazlar birbirini tanıdıktan sonra, dijital **BMC (Biphase Mark Coding)** sinyalleşmesi başlar. CC hattı üzerinden 300 kbps veri hızıyla (yaklaşık 300 kHz taşıyıcı frekans kullanarak) 0 ve 1'ler gönderilir. Her bit periyodunun başında bir geçiş vardır. ‘1’ biti, periyot ortasında ikinci bir geçiş içerir; ‘0’ biti içermez. Bu yöntem saat sinyalini veriyle birleştirerek sadece **tek bir hat** üzerinden gürültüye dayanıklı iletişim sağlar. CRC32 hata denetimi ile de mesajların doğruluğu garanti altına alınır.


3. İletişim paketler halinde yapılır. Bu paketlerin başlangıcına **SOP (Start of Packet)** denir. Normal SOP mesajları adaptörün telefon ile konuşmasıdır. **SOP' (SOP-Prime) mesajları ise adaptörün kablonun kendisiyle konuşmasıdır.** 

> "Eyy kablo, içindeki E-Marker çipine soruyorum: Sen 5 Amper (100W) taşıyabilir misin, yoksa erir misin?" Kablo onay vermezse, adaptör telefonu asla 60W (3A) üzerine çıkarmaz.


4. Donanımlar birbirini tanıyınca **Policy Engine** devreye girer. Bunlar yazılımsal karar mekanizmaları. Şarj aleti, telefona bir **PDO (Power Data Object)** listesi gönderir. Bu liste adaptörün menüsüdür:

> Sabit PDO: 5V/3A, 9V/3A, 15V/3A, 20V/5A verebilirim

> PPS (Programlanabilir) PDO: İstersen 3.3V ile 21V arasında ne istersen onu da verebilirim. 


5. Telefonun içindeki batarya kontrolcüsü kendi sıcaklığına ve batarya yüzdesine bakar. "Bana 2. sıradaki menüyü, yani 9V/3A'yı gönder" diyerek bir **RDO (Request Data Object)** paketi yollar.


6. Adaptör bu talebi (Accept) onaylar. İçindeki güç devresini ayarlar ve voltajı 5V'tan 9V'a çıkarır. Voltaj tam olarak kararlı hale geldiğinde **"PS_RDY (Power Supply Ready)"** mesajını fırlatır. Telefon bu mesajı almadan şarj devresinin (yük anahtarının) kapılarını yüksek voltaja açmaz. Ayrıca karşı taraf mesajı hatasız aldığında "GoodCRC" onayı yollar. Onay gelmezse sistem güvende kalmak için hemen varsayılan ayarlara döner.


PDO'ları **adaptörün sunduğu menü**, RDO'ları da **telefonun verdiği siparişin fişi** olarak düşünebilirsiniz.

{{< figure src="sousink.png" title=" " >}}




## PPS (Programmable Power Supply)

Özellikle Samsung'un "Süper Hızlı Şarj" adıyla kullandığı teknolojinin arkasında **PPS** yatar. Eski sistemlerde adaptör 9V gönderir, telefonun içindeki **Buck Converter** bu 9V'u lityum bataryanın ihtiyacı olan 4.3V'a düşürürdü. Bu işlem telefonun içinde muazzam bir sıcaklık yaratırdı.


Buck converter nedir? Temelde bir DC-DC dönüştürücüsüdür. **Düşürücü dönüştürücü** de diyebilirsiniz. Girişteki yüksek voltajı daha düşük bir voltaja dönüştürür. Isıya dönüşen fazla enerjiyi harcayarak çalışan lineer regülatörlerin aksine, enerjiyi geçici olarak manyetik alanda depolayıp kontrollü bir şekilde serbest bırakır.

{{< figure src="buckconv.jpg" title=" " >}}

Devreleri MOSFET-diyot-bobin-kondansatör dörtlüsünden oluşur genelde. 

> Açık: Akım, anahtar üzerinden bobine akar. Bobin enerji depolar ve aynı zamanda çıkışa (yüke) akım sağlar. Bobin, akımdaki ani artışa karşı koyarak onu doğrusal olarak yükseltir.

> Kapalı: Kaynaktan gelen yol kesilir. Bobin, üzerindeki manyetik alan çökerken akımı aynı yönde akıtmaya devam etmek için bir voltaj üretir (bu olguya "endüktansın geri tepmesi" diyebiliriz sanırım). Akım, diyot ve yük üzerinden dolaşarak devresini tamamlar. 

Çıkış voltajı, anahtarın duty cycle'ı tarafından belirlenir. Örneğin, 20V girişi %25 duty cycle anahtarlarsanız, çıkış yaklaşık 5V olur.


PPS ile telefon, şarj cihazına "Bana tam olarak pilin şu anki ihtiyacım olan voltajı, yani 4.2V ver" diyebilir. Bu sayede telefondaki buck converter'da dönüşüm oranı 1:1’e yaklaştığı için kayıp minimuma iner, verimlilik yükselir ve cihaz çok daha az ısınır.


USB PD 3.0 ile tanıtılmıştı PPS. Şu anki en modern versiyon ise **EPR – Extended Power Range (USB PD 3.1)**. Standart PD 3.0’da üst limit 20V @ 5A = 100W idi. PD 3.1 ile SPR (Standard Power Range) aynı kalırken, EPR modu tanımlandı. EPR sabit PDO’lar **28V (140W), 36V (180W) ve 48V (240W)** seviyelerine ulaşabiliyor. Kablolarınızın daha sağlam olması ve ark korumalı konnektör gerekir bunu kullanmak için. 


**TCPM (Type-C Port Manager)** diye bir şey de var. Telefon anakartındaki PD çipi, ana işlemciyle I2C veri yolu üzerinden sürekli konuşur. Siz ağır bir oyun oynarken telefon işlemcisi çok ısınırsa, Android çekirdeğindeki TCPM modülü PD çipine: "Çok ısındık. Hemen yeni bir Request mesajı at, voltajı 5V'a çekiyoruz." der. 


