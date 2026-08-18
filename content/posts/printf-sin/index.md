+++
title = 'Gömülü Sistemlerde printf'
date = 2026-08-03T16:11:03+03:00
draft = false
math = true
+++

C öğrenirken karşılaşılan ilk fonksiyonlardan birisi **printf**'tir. Gigabyte'larca RAM ve GHz hızında çalışan işlemciler varken ekrana bir metin basmanın maliyetini asla düşünmeyiz. İşletim sistemi her şeyi system call'lar kullanarak halleder. Her zaman böyle olur mu? Olmaz. Bare metal sistemlerde veya bir RTOS üzerinde çalışan, 48 MHz, 32KB Flash ve 8KB RAM'e sahip zavallı bir mikrodenetleyicide (STM32F0 veya Arduino AVR gibi) **printf() çağırmak sivrisineği bazuka ile öldürmek gibidir.** Bu büyük bir israftır. İsrafı başlıklara ayıralım.



## 1. Flash Katliamı

Gömülü sistemlerdeki kod alanımız olan **Flash** çok değerlidir. Siz sadece basit bir sayıyı (%d) ekrana yazdırmak istersiniz. Ancak printf() fonksiyonunu kodunuza çağırdığınız o masum an katliamın başladığı andır. 

Çünkü printf() doğası gereği **variadic** bir fonksiyondur. Yani değişken sayıda ve türde argüman alabilir. Masaüstü yazılımlarında bu büyük bir esneklik sağlarken, gömülü sistemlerde ortalığı mahveder. Kodu derleyip linker aşamasına geçildiğinde standart C kütüphanesi (libc / stdio.h) projeye dahil olur. Derleyiciler çok zekidir ama maalesef kahin değillerdir, geleceği göremezler. 

Sizin printf() kullanırken sadece bir tam sayı (integer) yazacağınızı bilse bile "Ya bu eleman kodun başka bir yerinde veya ilerideki bir güncellemede farklı formatlar kullanmak isterse?" varsayımıyla hareket eder. Sonra ekler: "Ya ileride %x (hexadecimal), %o (octal), %p (pointer adresi), %u (işaretsiz sayı), %04d (sola sıfır ekleme/padding) veya bilimsel notasyon (%e) kullanırsa?" Bu haklı endişeleri yüzünden **standart kütüphanedeki bütün bir format çözümleme (parsing) motorunu mikrodenetleyicinizin Flash hafızasına gömer.**


## 2. Execution Katliamı

Bununla da yetinmez. printf() fonksiyonunun % işaretini bulup virgüllü veya tam sayıları ASCII karakterlerine (yani insanların okuyabileceği harflere) dönüştürmesi (String Formatting) de devasa bir matematiksel işlemdir. 

> Bir integer sayıyı (örneğin 1234) string'e ("1234") çevirmek için işlemcinin sürekli sayıyı 10'a bölmesi, kalanını (modülo) bulması, bu rakama ASCII tablosundaki '0' karakterinin değerini (0x30) eklemesi ve bunu bir döngü içinde yapması gerekir.

Gömülü dünyadaki çoğu basit işlemcide **"hardware division"** devresi bile olmaz. Bölme işlemi yazılımla, çıkarma yapılarak taklit edilir. Yani printf'in formatı çözümleyip string oluşturması binlerce "clock cycle" harcar. Saniyede binlerce kez kontrol edilmesi gereken bir sensör veya motor varsa, işlemci bu matematiği yapmaktan asıl işine zaman bulamaz. 





## 3. Performans Katliamı

Farz edelim ki derleyici aşamasını geçtik. Bellekte ekrana basılmaya hazır, görece çok kısa bir log mesajı var: **"Sicaklik: 25\n"** Tam 13 karakter. Gömülü dünyada bir ekran veya gigabaytlarca veri aktaran bir PCIe veri yolu olmadığı için, bu metni genellikle standart bir UART (Seri Port) üzerinden bilgisayara göndeririz. Bu işleme **retargeting** denir. 

Masaüstü bilgisayarınızda bir metni konsola basmak birkaç nanosaniye sürer. Ancak UART gibi fiziksel ve asenkron bir donanım işin içine girdiğinde, zamanın yavaşlayacağı tutar. Hesap yapalım:

> UART hızı endüstri standardı olan 115200 baud rate olsun. Baud rate, saniyede iletilen bit sayısını ifade eder. UART iletişiminde her bir byte'lık (8-bit) karaktere bir Başlangıç (Start) ve bir Bitiş (Stop) biti eklenir. Yani bir karakter kablo üzerinden aslında 10 bit olarak seyahat eder. Bu durumda 115200 baud hızında saniyede kabaca 11.500 karakter gönderebilirsiniz. Basit matematik yapalım; tek bir karakterin fiziksel olarak bakır kablodan iletilmesi ~86 mikrosaniye (µs) sürer. 13 karakterlik metin ise **1.1 milisaniye (ms)** sürecektir.


Peki bu 1.1 milisaniyede mikrodenetleyiciniz ne yapar? Keşke bir şey yapsa. Fazlaca sayıda kütüphanede (HAL/Standard Peripheral) printf UART'a veri gönderirken **Blocking** modda çalışır. Örnekle anlatalım.

Diyelim ki mikrodenetleyiciniz 72 MHz ile çalışıyor. Bu, işlemcinin saniyede 72 milyon cycle yapabildiği anlamına gelir. Ama 1.1 milisaniye boyunca işlemci UART donanımının veriyi göndermesini beklerken **boş boş durur.** 1.1 milisaniye boyunca sadece donanım bayrağına (flag) bakıp verinin gitmesini beklerken, **o geçen sürede 80 bine yakın instruction işleyebilirdi.** 




## 4. ISR Katliamı

Gömülü sistemler dünyasında ISR (Interrupt Service Routine) çok önemlidir. Bunların taşıması gereken özellikler basittir. **Kısa ve yalın olmalıdır. Asla bloklayıcı olmamalıdır.** Bunlara mikrodenetleyicinin yangın alarmı diyebilirsiniz. İşlemci o an ne yapıyorsa bırakır, durumu kaydeder, acil duruma müdahale etmek için ISR fonksiyonuna atlar. Yangın alarmı çaldığında binayı tahliye etmek yerine oturup roman okumaya başlarsanız yanarsınız. **İşte ISR içinde printf() çağırmak sizi tam olarak yangın anında roman okuyan kişi yapar.**



UART üzerinden mesaj iletilirken yarattığı kesilmeyi yukarıda konuştuk. **Bunu ISR içinde yaparsanız o milisaniyeler boyunca sistemdeki diğer tüm eşit veya düşük öncelikli kesmeleri sağır bırakırsınız.** Örnek verelim.

> Sisteminizde her 1 milisaniyede bir tetiklenen ve kritik bir motor kontrol algoritmasını (PID) hesaplayan bir Timer kesmeniz var. Aynı zamanda bir butona basıldığında tetiklenen harici bir kesme (EXTI) içinde **printf("Butona basildi\n");** yazıyor. O butona basıldığı an, printf'in 5 milisaniye süren UART iletimi başlar. Bu 5 milisaniye boyunca Timer kesmeniz çalışamaz. Motor kontrolünüz 5 periyot kaçırır. 


C Standart Kütüphanesindeki (libc) klasik printf() implementasyonları genellikle **Thread-Safe** veya **Reentrant** değildir. Sayıları string'e çevirmek (itoa, ftoa) için kendi içinde ortak (static/global) buffer'lar kullanırlar. Yani?

> İşlemciniz main döngüsü içinde sensör verisini ekrana basıyor: **printf("Sensor Sicakligi: %d\n", sicaklik);** Donanım tam "Sensor Sica" kısmını UART'a göndermişken, bir anda araya UART Receive kesmesi girdi ve ISR tetiklendi. O ISR'ın içinde de bir hata logu basmak istediniz: **printf("HATA!");.** Ne olur? ISR içindeki printf, main içindeki printf'in kullandığı global statik buffer'ı ve UART donanım register'larını ezer. Kesme bitip main'e dönüldüğünde ekranda göreceğiniz şey anlamsız çöp karakterlerdir: **"Sensor SicaHATA!likli?%*&"**



Eğer sistemde FreeRTOS kullanıyorsanız ve yukarıdaki **Reentrancy** sorununu çözmek için printf fonksiyonunu bir **Mutex** (kilit mekanizması) ile korumaya aldıysanız mevzu daha da kötüleşir. 

  - Ana task printf kullanmak için Mutex'i alır.
  - ISR araya girer. ISR, tasktan daha yüksek önceliğe sahip olduğu için task askıya alınır.
  - ISR içinde de bir hata ayıklama mesajı yazdırmak veya bir durum bildirmek için printf kullanılmak istenir. printf çağrılmadan önce mutex alınmaya çalışılır.
  - **Ancak ISR'lar bloke edilemez ve bir Mutex'in açılmasını bekleyemez.** Bir ISR, beklemeye geçemez; çünkü ISR’lar donanım tarafından tetiklenir ve işlemciyi meşgul eder. Eğer ISR beklemeye kalkarsa, yüksek ihtimalle **HardFault** yersiniz. 


Ana döngülerin aksine ISR'lar çok daha kısıtlı Stack belleği üzerinde çalışır (Cortex-M mimarisinde MSP - Main Stack Pointer). printf(), variadic yapısı ve içindeki karmaşık string dönüştürme fonksiyonları nedeniyle **çok ciddi bir Stack tüketicisidir.**

Siz ISR içinde printf çağırınca o anki Stack limitini aşarak hafızadaki diğer kritik verilerin (global değişkenler, işletim sistemi kontrol blokları) üzerine yazarsınız.  Bu hata derleme aşamasında değil, sistem sahada çalışırken kendini gösterir ve debug etmesi işkence gibidir. 



## 5. Floating Point Katliamı

Bilgisayarda C yazarken **printf("Sicaklik: %f", 25.46);** yazıp geçersiniz. Sistem sizin için o ondalıklı sayıyı şık bir şekilde ekrana basar. Ancak aynı kodu 32KB Flash'ı olan bir mikrodenetleyiciye yüklerseniz hoş olmayan şeyler yaşarsınız. **Çünkü ekrana virgüllü sayı basmak, çok kolay görünse de teknik bir cehennemdir.**


Bir tamsayıyı (%d) ekrana basılacak bir string’e (ASCII karaktere) çevirmek nispeten kolaydır. 123 sayısı için:

  - 123 % 10 = 3 -> Son rakam
  - 12 % 10 = 2 -> Orta rakam
  - 1 % 10 = 1 -> İlk rakam

Bu işlem, sayıyı 10’a bölüp kalanı alarak, rakamları tersten elde etmemizi sağlar. Daha sonra bu rakamları '0' karakteriyle toplayarak ASCII karşılıklarını üretir ve bir diziye yazarız. Dizi sonunda ters çevrilir ve işlem tamamlanır.


**float** ya da **double** için işler bu kadar kolay değildir. Hafızada **IEEE-754** standardına göre saklanırlar:

  - **İşaret biti** (sign): Sayının pozitif mi negatif mi olduğunu tutar.
  - **Üs** (exponent): Sayının büyüklüğünü belirler. Üs, bias (öteleme) denilen sabit bir değer eklenerek saklanır.
  - **Kesir** (mantissa): Sayının anlamlı kısmını (significant digits) tutar. Normalize edilmiş sayılarda, virgülden önceki 1 biti gizlidir ve yalnızca kesir kısmı saklanır.

float için 32 bit şu şekilde dağılır: 1 bit işaret, 8 bit üs, 23 bit kesir. double için ise 64 bit: 1 + 11 + 52.


Siz printf’e %f formatlayıcısını verdiğinizde, arka planda bu ikili (binary) formattaki sayıyı alıp 2.546 gibi onluk tabanda yan yana duran metin karakterlerine dönüştürmesi gerekir. Bu dönüştürme işlemi genellikle **dtoa (double to ascii) algoritması** olarak bilinir. Neden tam sayılar kadar basit değildir? 

  - IEEE-754, sayıyı **ikili tabanda (binary)** saklar. Ancak %f çıktısı **onluk tabanda (decimal)** istenir. İkili kesirler ile onluk kesirler arasında birebir bir eşleşme yoktur. Örneğin 0.1 sayısı ikili sistemde tam olarak temsil edilemez; yaklaşık bir değerle saklanır. Bu yüzden dönüştürme sırasında doğru yuvarlama yapılması kritik önem taşır.
  - İyi bir dtoa algoritması, ürettiği ondalık string’in aynı double değerine geri dönüştürülebilmesini sağlamak zorundadır. Ayrıca mümkün olan en kısa temsil kullanılmalıdır (shortest round-trip conversion). Aksi halde 0.1 yerine 0.10000000000000001 gibi çirkin çıktılar ortaya çıkabilir.


0.1 sayısını printf("%f", 0.1) ile yazdırmak istediğinizi düşünelim. 0.1 aslında double olarak saklanırken tam 0.1 değildir. Bellekte yaklaşık olarak 0.1000000000000000055511151231257827... gibi bir değer tutulur. Algoritma, bu sayıyı en kısa ve doğru şekilde 0.1 olarak yazdırmak için çok sayıda matematiksel işlem yapar.



İşte bu ağır matematiksel işlemler o kadar çok bellek tüketir ki, **ARM GCC** gibi profesyonel gömülü derleyiciler printf içinde %f desteğini **varsayılan olarak kapalı tutar.** Siz kodunuzda %f kullansanız bile ekranda hiçbir şey göremezsiniz.

Sorunu çözmek için internette araştırma yapar ve derleyici ayarlarına **-u _printf_float** (Linker bayrağı) eklemeniz gerektiğini öğrenirsiniz. O bayrağı eklediğiniz an ise **Flash katliamı** yaşarsınız. Sırf o dtoa algoritmasını ve gerekli matematik kütüphanelerini çalıştırabilmek için, projenizin Flash boyutu aniden 10 KB ile 15 KB arasında şişer. 


Burada çok değerli bir itiraz gelebilir: **"Ama benim kullandığım Cortex-M4F işlemcide donanımsal kayan nokta birimi (FPU) var! Neden bu kadar zorlansın?"**

Donanımsal FPU, iki float sayıyı donanım seviyesinde tek bir saat vuruşunda (clock cycle) çarpıp toplayabilen harika bir birimdir. Ancak iş bir kayan noktalı sayıyı karakter karakter ASCII metne çevirmeye (dtoa) geldiğinde, **FPU tamamen işlevsiz kalır.** Çünkü metin dönüştürme işlemi salt bir "çarpma/bölme" işlemi değil, devasa bir **dallanma (branching) ve mantıksal analiz sürecidir.**


Arka planda çalışan dönüştürme algoritması sayıyı ekrana basmadan önce şu kontrolleri yapmak zorundadır:

  - Sayı sıfır mı? Negatif sıfır (-0.0) mı? Yoksa sonsuzluk mu (INFINITY) veya belirsiz bir matematiksel hata (NaN - Not a Number) mı? Algoritma önce if (isnan(val)) veya if (isinf(val)) gibi şart bloklarına girerek string'e doğrudan "NaN" veya "inf" yazıp yazmayacağına karar verir.
  - Sayının tam kısmı kaç basamaklı? Virgülden sonraki kısmında anlamlı kaç basamak var? Algoritma sayının üs (exponent) değerini sürekli maskeleyip (bit masking) if-else bloklarıyla kontrol ederek sayının büyüklük sırasını (magnitude) bulmaya çalışır.
  - Yukarıda bahsettiğimiz 0.1 paradoksundaki gibi, ikili tabandan onluk tabana geçilirken son basamakta sayının yukarı mı yoksa aşağı mı yuvarlanacağına karar verilmesi gerekir. Bu işlem **Grisu2** veya **Dragon4** gibi son derece karmaşık dtoa algoritmaları kullanılarak yapılır. Bu algoritmalar sürekli sınır koşullarını (if (deger > sinir)) kontrol eder.
  - İşte bu bitmek bilmeyen yapılar **branching** işlemleridir. İşlemciler komutları daha hızlı işlemek için **"Pipeline"** mimarisi kullanır. Yani bir komut işlenirken, sonraki komut önceden hafızadan getirilir (pre-fetch). Ancak kodunuzda çok fazla if-else varsa, işlemci hangi yola gireceğini önceden bilemez. Yanlış yolu tahmin ettiğinde ise pipeline'ı boşaltıp (pipeline flush) diğer yoldaki komutları baştan yüklemesi gerekir. Buna **"Branch Penalty"** denir. 


Bunun çözümü nedir? Hile kullanırsınız. **Fixed-point math** denir.

```c

float temp = 25.46;

// Sayıyı 100 ile çarpıp tamsayıya (integer) çevir
// temp_int değeri 2546 olur.
int temp_int = (int)(temp * 100); 

// Ekrana tamsayı (%d) bas, ama araya kendin nokta koy
// 2546 / 100 = 25 (Tam kısım)
// 2546 % 100 = 46 (Ondalık kısım)
// (%02d kullanımı, "05" gibi tek haneli ondalıklarda sıfırı korumak içindir.)

printf("Sicaklik: %d.%02d\n", (temp_int / 100), (temp_int % 100));

```

Sonuç? Ekranda okuduğunuz çıktı kusursuz bir **Sicaklik: 25.46** olur. Fakat %f kullanmadığınız için Flash hafızanızdan 15 KB, işlemcinizden de binlerce cycle tasarruf etmiş olursunuz. 




## 6. Güvenlik

C dili size güvenmese de saygı duyar. Köprüden atlamak isterseniz size engel olmaz. printf() bu felsefenin en belirgin vücut bulduğu, hiçbir type-safety olmayan bir fonksiyondur.


printf, daha önce de bahsettiğim gibi variadic bir fonksiyondur. Yani parametreleri runtime'da Stack'ten çeker. Ama çok kritik bir problem vardır çünkü **Stack'ten çektiği verinin gerçekten ne olduğunu asla bilemez, sadece sizin format string'inize (%d, %f, %s) güvenir.**

> **printf("Degerler: %d, %d, %d\n", sensor1);** yazdınız mesela. Format string üç adet %d içeriyor ancak siz sadece bir değişken (sensor1) verdiniz. printf ne yapar? Keşke hata verse. Stack'te pointer'ı ilerletir ve o an hafızada sensor1'in arkasında ne varsa (başka bir fonksiyonun geri dönüş adresi, gizli bir anahtar veya rastgele çöp veriler) onları tamsayı zannederek acımasızca okuyup ekrana basar.


> **printf("Sicaklik: %d", 25.46f);** yazdınız diyelim. Bu daha da beter çünkü **tür uyuşmazlığı** var. Eğer bir float değişkeni %d ile basmaya çalışırsanız, printf IEEE-754 formatındaki o ikili veriyi doğrudan bir tamsayı olarak okur ve ekrana anlamsız, devasa bir sayı basar.



İnternete bağlı bir ESP32 veya dışarıdan UART ile komut alan bir Cortex-M işlemci programladığınızı düşünün. Cihaz dışarıdan bir veri alıyor ve siz de bunu debug ekranına basmak istediniz. Şunu yazarsanız sıkıntı çıkar:


```c
char rx_buffer[100];
UART_Receive(rx_buffer); // Dışarıdan veriyi al

// Rezalet kullanım
printf(rx_buffer); 
```

Bunu bu halde yazarsanız normal metin değil de **"%x %x %x %x %x"** geldiğinde ne olur? printf dışarıdan gelen bu veriyi bir **"format string"** olarak kabul eder. Her %x gördüğünde, Stack'ten (hafızadan) bir parça alır ve **bunu Hexadecimal (onaltılık) olarak UART'tan dışarı kusar.** Saldırgan, cihazınıza sadece birkaç %x göndererek mikrodenetleyicinizin bütün RAM içeriğini (Wi-Fi şifrelerini, kriptografik anahtarları, cihaz konfigürasyonlarını) okuyabilir. Güvenli kullanım **printf("%s", rx_buffer);** şeklinde olmalıdır. Bu, printf'e "ikinci argümanı bir string olarak al, olduğu gibi yazdır" der. rx_buffer içindeki hiçbir karakter format belirteci olarak yorumlanmaz. İçinde % olsa bile, sadece o karakter basılır.



Bitti mi sandınız? Daha kötüsü var: **%n**. Diğerleri sadece okuma yaparken bu **hafızaya yazma yapar.** O ana kadar ekrana basılan karakter sayısını, kendisine argüman olarak verilen bir bellek adresine kaydeder. **Dikkatlice hazırlanmış %n içeren veri paketi gelirse, mikrodenetleyicinin RAM'indeki herhangi bir adrese istediği veriyi yazabilir.**

Masaüstü işletim sistemlerinde bunu engellemek için **MMU (Hafıza Yönetim Birimi), ASLR (Adres Alanı Rastgeleleştirme) ve Stack Canary** gibi güvenlik önlemleri vardır. Ancak "Bare-metal" çalışan bir mikrodenetleyicide (çoğu zaman bir MMU yoktur) tüm hafıza korumasızdır (Flat Memory Model). Saldıran kişi, %n hilesiyle bir fonksiyonun geri dönüş adresini değiştirerek program akışını kendi istediği yere yönlendirebilir veya sistemi anında HardFault'a düşürüp cihazı servis dışı bırakabilir. 


%n arkadaşın ne kadar tehlikeli olduğu yakın geçmişe kadar fazlasıyla küçümsendiği için biraz detaylandırmak istiyorum. 

Şu soru aklınıza gelebilir: "Nasıl yani, ekrana basılan karakter sayısı sisteme nasıl zarar verebilir?" Saldırganların amacı sadece rastgele bir sayı yazdırmak değildir; **istedikleri adrese, tam olarak istedikleri sayıyı yazdırmaktır.** Peki o sayıyı nasıl belirlerler? Cevap, printf'in **padding** özelliğinde gizlidir.

Saldırgan cihazınıza şu format string'i gönderir: **%2052c%n**

  - **%2052c** : printf'e "Bana bir karakter bas ama başını boşluklarla doldurarak toplam 2052 karakter uzunluğuna tamamla" der. printf itaat eder ve hafızada 2052 karakterlik bir çıktı oluşturur.
  - **%n** : O anki karakter sayısını (Yani tam olarak 2052 değerini) Stack'te bulduğu ilk adrese yazar. (2052'nin Hexadecimal karşılığı 0x0804'tür).

Saldırgan, %n komutunu vermeden hemen önce format string zafiyetiyle (%x %x %x... kullanarak) hafızada okuma imlecini tam da **return address**'in olduğu noktaya kaydırmış olmalıdır. 

Mikrodenetleyici printf işlemini bitirip asıl programa (örneğin main döngüsüne) dönmek istediğinde, geri dönüş adresine bakar. **Ancak o adres artık orijinal yerini değil, saldırganın zorla yazdırdığı 0x0804 adresini göstermektedir.** İşlemci körü körüne o adrese atlar. **Eğer saldırgan o adrese önceden kendi zararlı kodunu (shellcode) yerleştirmişse, cihazın tüm kontrolü artık onun elindedir.** Eğer yerleştirmemişse cihaz anında HardFault verip çöker ve sistem servis dışı kalır.


Bu zafiyete literatürde **CWE-134: Use of Externally-Controlled Format String** denir. Yakın zamanda gördüğümüz **CVE-2026-18186** ve feci derecede kritik olan **CVE-2024-23113** harika örneklerdir.





## Alternatifler

Böyle bir yürüyen güvenlik açığını kullanmak zorunda mıyız? Değiliz.

**mpaland/printf, xprintf, nano_printf** gibi alternatifler mevcuttur. Bu kütüphaneler projenize eklendiğinde standart printf'in 15-20 KB'lık yükü yerine sadece 1 ile 2 KB arasında yer kaplar. İçlerinde (istenirse açılabilen ama genelde kapalı olan) çok basit bir float desteği vardır. Statik buffer kullanırlar ve asla malloc çağırmazlar. Güvenlik açığı yaratan %n formatlayıcısı genellikle içlerinde hiç yoktur.



Hafıza sorununu üstteki arkadaşlar çözüyor. Blocking mevzusu ne olacak? Real-time bozulmasın istiyorsanız donanımsal destek şart. Burada devreye **DMA (Direct Memory Access) ve Ring Buffer** giriyor. 

Siz kodunuzda (main veya RTOS task'ı içinde) log basmak istediğinizde, string formatlanır ancak **asla UART donanımına gönderilmez.** Bunun yerine, RAM'de oluşturduğunuz ring buffer'a yazılır. Mikrosaniye falan sürer. Ardından CPU, mikrodenetleyicinin içindeki özel "Kargo Şirketi" olan DMA donanımını tetikler ve işine döner.

DMA donanımı, arka planda CPU'dan tamamen bağımsız çalışır. RAM'deki o log mesajını byte byte alır ve UART üzerinden dışarı yollar. İşlemci o sırada motor kontrol ediyor, sensör okuyor veya uyku modunda güç tasarrufu yapıyor olabilir. O 1.1 milisaniyelik bekleme süresi sıfıra iner. İşlem bittiğinde DMA bir interrupt üreterek "Kargoyu teslim ettim!" der.


Aslında modern ARM Cortex-M tabanlı profesyonel projelerinde, loglama için artık pinleri ve CPU'yu meşgul eden UART donanımı kullanılmaz. Hedef karta zaten kod yüklemek için bağladığınız bir **Debugger (ST-Link, J-Link vb.)** vardır. O halde neden veriyi bu çok hızlı hata ayıklama hattından almayalım ki?

  - **ARM ITM**: İşlemcinin içindeki donanımsal bir modüldür. SWO (Serial Wire Output) adı verilen tek bir pin üzerinden, CPU'yu neredeyse hiç meşgul etmeden (nanosaniyeler içinde) bilgisayarınıza devasa hızlarda log basmanızı sağlar.
  - **SEGGER RTT**: Fazladan bir pine bile ihtiyaç duymaz (Standart SWD pinlerini kullanır). RTT, mikrodenetleyicinin RAM'inde küçük bir kontrol bloğu oluşturur. Siz log bastığınızda veri sadece bu RAM bloğuna yazılır. İşlem süresi dehşet kısadır. Peki veri bilgisayara nasıl gider? J-Link debugger, arka planda işlemciyi hiç durdurmadan ve CPU'nun ruhu bile duymadan RAM'deki o bloğu okur ve bilgisayar ekranına basar. Tamamen non-blocking, ultra hızlı, %100 real-time bir loglama mucizesidir. ISR içinde bile kullanılabilir.



Dürüst olalım, her zaman karmaşık formatlanmış string'lere ihtiyacımız var mı? Hayır yok. **puts()** kullanıp geçebilirsiniz. İçinde hiçbir format (parsing) motoru barındırmaz. Sadece karakter dizisinin sonundaki \0 karakterini görene kadar baytları sırayla donanıma basar. Flash maliyeti sıfıra yakındır. 

Bir tamsayı mı basacaksınız? Kendi yazdığınız basit ve çok hızlı bir itoa() (Integer to ASCII) fonksiyonu kullanarak sayıyı metne çevirin ve puts() ile gönderin. Böylece derleyicinin standart kütüphaneyi projenize dahil etmesini tamamen engellemiş olursunuz.



## Sonuç

Bir Raspberry Pi ile hızlıca prototip geliştirirken printf() kullanmakta özgürsünüz ve işiniz de bir hayli kolaylaşır. Ancak pille çalışan bir IoT cihazı, bir medikal alet, saniyede binlerce kez karar veren bir drone veya on binlerce adet üretilecek maliyet odaklı bir endüstriyel kart tasarlıyorsanız; **printf()'i kodunuzdan uzak tutun.** İyi bir embedded systems mühendisi her cycle'ın ve byte'ın değerini bilmelidir. Buna ürünün maliyetindeki azalıştan ziyade sizi daha iyi bir mühendis yaptığı için dikkat etmelisiniz.




## Soru

Bu konudan anlıyorsanız **Fixed-point math** kısmındaki koda dikkatli bakın ve şu soruları cevaplayın:

<details>
<summary><b>Sıcaklık -25.46 olsa çıktı nasıl olur?</b></summary>

**Sicaklik: -25.-46** şeklinde olurdu. 

</details>

<details>
<summary><b>Sıcaklık -0.46 olsa çıktı nasıl olur?</b></summary>

**Sicaklik: 0.-46** şeklinde olurdu.

</details>

<details>
<summary><b>Sorunsuz kod nasıl olmalıydı?</b></summary>

```c
float temp = -25.46;

// 1. İşareti kontrol et ve bir flag'de tut
// (Bu işlem 0.0 ile 1.0 arasındaki negatif sayıları (örn: -0.46) kaybetmemek için hayatidir)
int is_negative = (temp < 0) ? 1 : 0;

// 2. Sayı negatifse, onu pozitife (mutlak değere) çevir. Neden? Mod işlemleri çökmesin diye
if (is_negative) {
    temp = -temp;
}

// 3. Artık sayı kesinlikle pozitif. Güvenle 100 ile çarpıp tamsayıya çevir
int temp_int = (int)(temp * 100); 

// 4. Parçalama işlemini yap
int tam_kisim = (temp_int / 100);
int ondalik_kisim = (temp_int % 100);

// 5. Ekrana bas. 
// Ternary operatörü (is_negative ? "-" : "") sayesinde sayı negatifse başa "-" koy, değilse boşluk ("") koy
// (%02d kullanımı, ".05" gibi tek haneli ondalıklarda sıfırı korumak içindir.)

printf("Sicaklik: %s%d.%02d\n", (is_negative ? "-" : ""), tam_kisim, ondalik_kisim);


```
</details>























