

Kullanıcağımız toollar ;

- APK Easy Tool ( http://bit.ly/2EIuDXi )
- IDA PRO 7.0 Freeware ( http://bit.ly/1g6Wbo8 )
- Frida ( ```pip install frida``` )
- jdax ( http://bit.ly/2ClsLyu )

Yüklü olmasını beklediğimiz programlar;

- Android Studio;
  https://developer.android.com/studio/index.html
  + Windows kullanıyorsanız kurduktan sonra PATH inize  ``` C:\Users\[user]\AppData\Local\Android\Sdk\platform-tools ``` eklemeniz gerekiyor.
  + ADB yi ve Emulator'ü  Kullanıcaz.



Soruda verilen apk;
  mobil400.APK

Soruda bizden 557598******0364 şeklinde bir kredi kartı numarası girmemiz bekleniyor.
![Alt text]("./MOBILE400/apk_giris.jpg")

6 haneden 1 milyon seçenek var. Luhn algoritması sayesinde bu olasılıkları 100.000'e düşürebiliriz.Anca bunların hepsini elle girmemiz malesef mümkün değil.

Denemek isteyenler için windows scripti;
+ ```FOR /r %%X IN (luhnchecked.txt) DO (adb shell input text %%X & adb shell input keyevent 66 66 & adb shell input tap 700 800 & adb shell input 67 67 67 67 67 67 67 67 67 67 67 67 67 67 67 67)```
windows_gif

Bizim bu olasılıkları çok hızlı deneyebilmek için frida kullanmamız gerekiyor. Frida kısaca özetlersek telefonda çalışan bi uygulamaya dinamik olarak bağlanıp, aklınıza gelebilcek nerdeyse herşeyi yapabilyor.
jdax ile gördüğümüz claslların instancelarına ulaşıp, o fonksiyon çağrıldıktan sonra değerleri manipule edebiliyorsunuz.
![Alt text]("./MOBILE400/jdaxview.jpg.jpg")


uygulamayı incelediğimizde anlıyoruzki check() fonksiyonu native bir libraryde ve kontrol ettiği değeri this.m üzerinden alıyor.
k() fonksiyonuda bizim için biçilmiş kaftan. Ne verirsek m'e onu veriyor.
Yolumuz belli;
 - Frida kontrollerini bypass edicez
 - Frida ile classa bağlanıp, önce k fonksiyonuyla m değerini değiştiricez ondan sonrada check fonksiyonunu çağırıp dönen değeri kontrol edeceğiz.

+ apk'yı emulatore yüklemek için ```adb install mobil400.apk```

# Fridayı bağlamak

Fridayı bağlamak için önce emulatorun içine
+  ```frida-server-10.6.52-android-x86_64``` ekliyoruz
+ adb push ../frida-server-10.6.52-android-x86_64 /data/local/tmp/
+ adb shell
+ su
+ cd /data/local/tmp
+ ./frida-server-10.6.52-android-x86_64 &
+ Başka bi terminalde frida-ps -U yazıyoruz ve telefonda çalışan uygulamaları görebiliyoruz.
+ NFC_pay uygulamasını açıyoruz tekrar frida-ps -U yaptığımızda
![Alt text]("./MOBILE400/frida-ps.jpg")


+ frida -U five.dkhos.mob.nfc_pay ile bağlanmaya çalışıyoruz
![Alt text]("./MOBILE400/frida1.jpg")


+ Bu bi anti-debugger taktiği biraz araştırma sonrasında bunu bypass etmek için -f parametresini eklememiz gerektiğini öğreniyoruz. -f parametresi ile fridaya direk process'e inject olmak yerine Zygote a bağlanıp processi kendisi başlat diyoruz.
![Alt text]("./MOBILE400/fparameter.jpg")
fparameter.jpg

Ancak "Frida found yazısı ile karşılaşıyoruz"
frida_found.jpg
![Alt text]("./MOBILE400/frida_found.jpg")

İlk bypassımız kolay. jdax ile class yapısını anladıktan sonra APKEasyTool ile apkyı decompile ediyoruz.
decompile.jpg
![Alt text]("./MOBILE400/decompile.jpg")
 jdaxview.jpg
 ![Alt text]("./MOBILE400/jdax_view.jpg")
 + ``` .\smali\five\dkhos\mob\nfc_pay ```  içinde 2 adet smali dosyası görüyoruz. Bu aşamada jdaxdaki görüntü ve smali dosyası arasında mekik dokuyarak, hangi fonksiyon smalide nereye denk geliyor onu anlamamız gerekiyor.
 smalibypass.jpg
![Alt text]("./MOBILE400/smalibypasss.jpg")
 Anlıyoruzki 48. satırdaki if bizim hedef noktamız. Bu if'e giren değer 1 ise "Frida found" şeklinde uyarı alıyoruz. Tabi bu adımlardan önce apkya frida bağlamayı denedik. Frida found yazısını orda da gördük.
 if'e const olarak 0 verirsek çok tatlı olucak. 39. satır gözümüze çarpıyor ve 48. satırda v0 yerine v4 yazıyoruz. Herşey tamam !

 APKEasyTool ile apkyı sırasıyla Recompile-Zipalign-Sign ediyoruz. Emülatorumuze yüklüyoruz ve işlem tamam. Artık sayı girip Doğrula dediğimizde "Frida found" yazısıyla karşılaşmıyoruz.
halafrida.jpg
![Alt text]("./MOBILE400/halafrida.jpg")
 Haydaa. Demekki native libraryde de bi frida checki var.

 native-lib var ancak jdax-gui ile bu dosyaları göremiyoruz. O yüzden APKEasyTool ile decompile ediyoruz.
  decompile.jpg
 ![Alt text]("./MOBILE400/decompile.jpg")
  nativelib.jpg
 ![Alt text]("./MOBILE400/nativelib.jpg")
  ../lib/x64_64/libnative-lib dosyasını görüyoruz.

 libnative-lib.so dosyasını IDA Pro ile açıyoruz.
 secenek_ida.jpg
![Alt text]("./MOBILE400/secenek_ida.jpg")
 Frida checkini takip ettiğimizde loc_6380 fonksiyonua gidiyoruz. loc_6380 test yaptıktan sonra değere göre dallanıyor. Burda hızlıca işaretlediğimiz yere bi bypass atmamız lazım.
 Madem frida kullanıcaz o zaman sürekli sol tarafa düşeceğimizden  testteki 1'i 0'a çeviriyoruz.
  secenekler.jpg
![Alt text]("./MOBILE400/secenekler.jpg")
 Hemen açıyoruz ve ok
 bypass_ok.jpg   
![Alt text]("./MOBILE400/bypass_ok.jpg")
 Herşey tamam şimdi fridaya javascript scripti verip check ve k fonsiyonlarını kendimiz çağırıcaz.
```

function valid_credit_card(value) {
  // accept only digits, dashes or spaces
	if (/[^0-9-\s]+/.test(value)) return false;

	// The Luhn Algorithm. It's so pretty.
	var nCheck = 0, nDigit = 0, bEven = false;
	value = value.replace(/\D/g, "");

	for (var n = value.length - 1; n >= 0; n--) {
		var cDigit = value.charAt(n),
			  nDigit = parseInt(cDigit, 10);

		if (bEven) {
			if ((nDigit *= 2) > 9) nDigit -= 9;
		}

		nCheck += nDigit;
		bEven = !bEven;
	}

	return (nCheck % 10) == 0;
}

setImmediate(function() { //prevent timeout
    console.log("[*] Starting script");

    Java.perform(function() {
	     Java.choose("five.dkhos.mob.nfc_pay.MainActivity", {
            onMatch: function (instance) {  //Instance buluyor
				      basenumber = 5575980000000364;
				      for( i = 0;i<1000000;i++)
				      {
				      number = basenumber+i*10000;
				          if(valid_credit_card(String(number))){
				                instance.a(String(number))
				                if(i%100 === 0 ) console.log(number)
				                retStr = instance.check()
				                if(retStr !== "Yanlis numara!" ){
					               console.log("Found number " + number)
					               break
				                }
				          }
				      }
            },
              onComplete: function () { }

        });
    })

})
```
