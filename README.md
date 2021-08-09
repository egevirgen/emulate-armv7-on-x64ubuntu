# x64 Tabanlı Ubuntu'da QEMU Üzerinde ARMv7 Tabanlı Sistem Emülasyonu

Bu dokümantasyon, QEMU üzerinde çalışan ARMv7 cihaz üzerine Raspbian işletim sistemi yükleyip çalıştırmak için gerekli adımları anlatmak için hazırlanmıştır. 

Host cihazın işletim sistemi Ubuntu 20.04.2 LTS'dir.

Guest cihazın işletim sistemi Raspbian Stretch Lite'dir.

# QEMU Kurulumu

ARM tabanlı bir sistem emülasyonu yapmak istediğimiz için qemu-system-arm paketini yüklememiz yeterlidir. Farklı guest sistemleri kullanmak istersek bunların paketlerini yüklememiz gerekecektir. Aşağıdaki kod satırını terminale kopyalayarak ARM sistemler için QEMU paketini yüklüyoruz.

`sudo apt-get install qemu-system-arm`

# Guest Cihaz İşletim Sisteminin İndirilmesi

Bu dokümantasyonda Raspbian Stretch Lite sürümü kullanılmıştır. Bu dosyayı sitesinden `wget` komutu ile çekeceğiz. Eğer cihazınızda yüklü değilse, aşağıdaki komut satırı ile yükleyiniz.

`sudo apt-get install wget`

Dosyanın indirilmesi ve Unzip Edilmesi
```
wget http://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2018-11-15/2018-11-13-raspbian-stretch-lite.zip

unzip 2018-11-13-raspbian-stretch-lite.zip
```
Zip içerisinden çıkan Raw Disk Image formatındaki dosyayı, ileride komut satırında kullanım kolaylığı sağlaması açısından `raspbian.img` olarak adlandırıyoruz.

# Kernel ve DTB Dosyaları Oluşturulması

Bu dosyalar, emüle etmek istediğimiz board için özel olarak oluşturulmak durumundadır. Buildroot isimli bir araç, Kernel, RootFS, DTB, Bootloader oluşturmakta kolaylık sağlamaktadır. Bu aracı aşağıdaki komutlar ile cihazımıza indireceğiz.
```
wget http://www.buildroot.org/downloads/buildroot-2019.02.tar.bz2

tar xf buildroot-2019.02.tar.bz2
```

Her şey yolunda ise, home klasörümüzün içinde `buildroot-2019.02` dosyası oluşmuş olmalıdır. Bu dokümantasyonda, ARMv7 tabanlı cihazın simüle edilmesi için `Arm Versatile Express Geliştirme Kartı` seçilmiştir. Gerekli konfigürasyon dosyalarını oluşturmaya başlamadan önce, sisteminizde `make, gcc, g++, python` paketlerinin yüklü olduğundan emin olmalısınız. Bu kontrolü şu şekilde yapabilirsiniz.

`sudo apt-get install make gcc g++ python`

Ardından konfigürasyon dosyalarını oluşturuyoruz.

```
cd buildroot-2019.02/

make qemu_arm_vexpress_defconfig

make
```

Make işlemi 15-20 dakika kadar sürmektedir. Her şey yolunda ise make işlemi sonunda `buildroot-2019.02/output/images` yolu içerisinde `rootfs, zImage ve vexpress-v2p-ca9.dtb` dosyaları oluşmuş olmalıdır. Kullanacak olduğumuz dosyalar `zImage ve vexpress-v2p-ca9.dtb` dosyalarıdır.

# QEMU Üzerinde Raspbian Başlatılması

Komut satırı kolaylığı açısından, az önce üretmiş olduğumuz `zImage ve vexpress-v2p-ca9.dtb` dosyalarını, bulundukları klasörden kopyalayıp `buildroot-2019.02` ana dizinine yapıştırıyoruz. Aynı şekilde, home klasörüne indirip extract etmiş olduğumuz ve `raspbian.img` olarak adlandırdığımız Raspbian dosyasını da `buildroot-2019.02` ana dizinine yapıştırıyoruz. Bunlar şart olmayan fakat komut satırını daha temiz tutan değişiklikler olup yapıp yapmamak geliştiricinin insiyatifindedir.

`buildroot-2019.02` ana dizini içerisinde yeni bir terminal başlatınız. Terminalde aşağıdaki komut satırını başlattığınızda, sistem boot edilmeye başlanacaktır. 

```
qemu-system-arm \
        -M vexpress-a9 \
        -m 256 \
        -kernel zImage \
        -dtb vexpress-v2p-ca9.dtb \
        -sd raspbian.img \
        -append "console=ttyAMA0,115200 root=/dev/mmcblk0p2" \
        -serial stdio
```

Her şey yolunda ise yaklaşık 1 dakika süren boot işleminin ardından Raspbian işletim sistemi sizden login bilgileri isteyecektir. Default olarak kullanıcı adı `pi` ve şifre `raspberry` dir. Sistemin açılmasının ardından `lscpu` komutuyla, cihazın ARMv7 tabanında çalıştığı bilgisine ulaşılabilir.

## -- Komut Satırı Açıklamaları --

`qemu-system-arm` ARM tabanlı bir QEMU emülasyonu başlatır.

`-M` Hangi makineyi baz alarak QEMU emülasyonunun başlatılacağını belirler. ARM emülasyonlarında default olarak bir makine belirlenmediği için bu seçenek mutlaka belirtilmelidir.

`-m` Guest cihaza ayrılacak RAM miktarını belirler. Her makine için ayrılabilecek maksimum sınır farklı olmaktadır. Bu örnekte 256MB RAM alanı ayrılmasına karşın, örneğin 512MB ayrılsa da bir problem olmayacaktır. Sınırların dışında bir miktar ayarlanmak istenirse hata mesajı ile karşılaşılacak ve sistem boot etmeyecektir. Guest sistemin RAM miktarı `free` komutu ile öğrenilebilir.

`-kernel` Seçeneği ile işletim sisteminin hangi kernel üzerinde çalışacağı belirlenir.

`-dtb` Seçeneği ile kullanılacak olan DTB dosyası belirlenir. DTB, Device Tree Blob'un kısaltması olup içerisinde kullanılan cihaz ile ilgili bilgiler barındırır.

`-sd` Seçeneği ile Guest cihaza işletim sisteminin içinde bulunduğu dosya tanıtılır. Bu örnekte .img uzantılı Raspbian dosyasını gösterdiğimiz için herhangi bir kurulum yapmadan hemen işletim sistemimizi çalıştırabildik. Bunun yerine duruma göre .qcow veya .qcow2 gibi formatlarda Guest için bir sanal harddisk oluşturup, bunun üzerine işletim sistemimizin kurulumunu yapıp, oradan çalışmasını sağlayabilirdik.

`-append` Seçeneği, Raspbian'ın Lite sürümünü kullandığımız ve bu sebeple bize bir görüntü çıktısı vermediği için, Guest bilgisayarının console çıktılarını Host bilgisayarın terminalinde serial port yardımıyla görebilmemizi sağlayan komutları iletmemizi sağlar. Bu örnekte ttyAMA0 portuna 115200 baud ile veri gönderimi yapması sağlanmıştır.

`-serial stdio` Seçeneği ile, Guest cihazın gönderdiği console satırlarının Host cihazda görüntülenmesi sağlanmıştır. 



