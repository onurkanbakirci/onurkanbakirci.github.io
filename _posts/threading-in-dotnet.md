---
layout: post
title: .NET Ortamında Threading - 1
#subtitle: .NET Ortamında Threading - 1
#cover-img: /assets/img/dotnet.png
thumbnail-img: /assets/img/dotnet.png
share-img: /assets/img/dotnet.jpg
tags: [.NET, async/await, threading]
---

Merhaba. Bu yazı serisinde, uygulama geliştiricilerin geliştirdikleri uygulamalarda yoğun bir şekilde kullandıkları asenkron programlama konseptinden bahsetmeye çalışacağız. İki kısım halinde planlanan yazının ilk kısmında OS seviyesinde işler nasıl yürütülüyor bunu inceleyeceğiz. Yazının ikinci kısmında ise arka tarafta compiler’ın kodlarımızı neye dönüştürdüğünü inceleyeceğiz. Yazı serisinde Windows işletim sistemi ve .NET ortamı baz alınarak açıklamalar yapılacaktır. Her ne kadar işletim sistemleri arası farklılıklar bulunsa da genel konsept aynıdır.

Asenkron programlama, CPU ve memory gibi kaynakları en verimli biçimde kullanarak ölçeklenebilir uygulamalar geliştirmemizi, kullanıcılarımıza daha responsive bir deneyim sunmamızı sağlar. Öncelikle geçmişte karşılaşılan sorunları incelemekle başlayalım.

Eski tip işletim sistemlerinde thread kavramı yoktu. İşletim sistemi üzerinde birçok uygulama açık olduğu zaman üzerinde anlık olarak işlem yapılan uygulama çalışırken diğer uygulamalar bekliyordu. İşletim sistemi üzerinde koşan uygulamalardan aktif olarak kullanılan uygulama herhangi bir bug’a girdiği zaman (mesela sonsuz döngü gibi) bu uygulamadan yanıt alınamadığı gibi diğer uygulamalar da pasif durumda olacağı için işletim sistemi ile olan iletişim tamamen kopuyordu. Bu durumda yapmanız gereken güç düğmesini kullanarak bilgisayarınızı yeniden başlatmak oluyordu. Bilgisayarınızı kapatıp yeniden açtığınızda da yapmış olduğunuz tüm işlemler uçmuş oluyordu. Bu durum Microsoft’u rahatsız ediyordu. Kullanıcılarına güvenilir ve performanslı bir işletim sistemi sunmak isteyen Microsoft, yeni bir işletim sistemi ilân etti.

Artık bu işletim sistemi ile beraber Windows’a thread konsepti eklenmişti. Bu yeni versiyon ile beraber OS’de process’ler thread’ler tarafından çalıştırılacağı için ve CPU bu thread’ler arasında switch edeceği için thread’lerin herhangi birinde bug oluşsa ve bunun neticesinde thread cevap veremez duruma gelse bile diğer thread’ler işletim sistemi üzerindeki diğer process’leri koşturmaya devam edebilecektir. Problemli olan process de PID’si(process id) ile kill edilebilir olacaktır. Peki bu process ve thread nedir? Biraz daha yakından inceleyelim.

İşletim sistemi üzerinde koşan her uygulama OS tarafında process (virtualizing of the RAM) olarak isimlendirilir. Process’ler birbirinden izole olup sanki işletim sistemi üzerinde sadece kendisi koşuyormuş gibi hissederler. Her bir process’in en az bir thread’e (virtualization of the CPU) sahip olması gerekmektedir. Process’in sahip olduğu ana thread primary thread (main thread) olarak isimlendirilir. Scheduler, thread’lerin sahip olduğu priority level’lına göre CPU tarafından çalıştırılma önceliklerini belirler. CPU, thread’leri quantum time olarak isimlendirilen zaman aralığında çalıştırır ve quantum time tamamlandığında CPU gerekli verileri kaydeder ve yeni thread’i çalıştırmak için bu yeni thread’in context’i CPU’a yüklenir. Bu şekilde thread’ler arasında sürekli bir geçiş olur. Bu geçişlere de Context-Switch denmektedir. Thread’ler arasında fazla geçiş olduğu zaman Context-Switch süresi CPU zamanının yüksek bir bölümünü kaplayacaktır. Çünkü bu geçiş esnasında thread’e gerekli bilgilerin kaydedilmesi ve yeni gelen thread’in context’inin CPU’ya tekrar yüklenmesi maliyetli bir işlemidir. Bunun için gereksiz yere Context-Switch’ten kaçınıp kaynakları gereksiz kullanmamak gerekmektedir. Peki bu olaylar .NET tarafında nasıl yönetilmektedir?

.NET ortamına .NET Framework 4 ile beraber Task-based Asynchronous Pattern(TAP) eklendi . C# 5 ile beraber de async/await keyword’leri duyuruldu. TAP öncesinde başka asenkron programlama modelleri de vardı. Bunlar: Event-based Asynchronous Pattern(EAP) ve Asynchronous Programming Model(APM) idi. Fakat Microsoft tarafından tavsiye edilen asenkron model, en son eklenen Task-based Asynchronous Pattern(TAP)’dır. Şimdi gelelim bu pattern’i ve C# 5 ile gelen kodu yazan kişiyi birçok şeyden soyutlayan o popüler async/await keyword’lerini ne zaman kullanmamız gerektiğine ve ne işimize yaracağını incelemeye. async/await’i kullandığımız iki iş tipi vardır. Bunlar:

1. I/O Bound
2. CPU Bound

Şimdi bu iki iş tipini örnekler ile inceleyerek async/await ‘i daha iyi anlamaya çalışalım.

## I/O Bound

Third-party bir servise network isteği yapıyorsanız, bellekteki bir dosyadan okuma veya dosyaya yazma işlemi yapıyorsanız I/O bound işlem yapıyorsunuz demektir. Peki I/O bir işlemi yaparken arka planda neler gerçekleşiyor? Mesela microsoft.com’a istek atan C# kodumuz olsun. Kodlarımız thread tarafından çalıştırılmaya başlar. Sıra microsoft.com’a istek yapılan kod satırına gelir. BCL (Base class library) kullanılarak Windows işletim sisteminin I/O işlemlerini gerçekleştiren API’si ile iletişime geçilir. OS ile iletişime geçildikten sonra, OS bu network isteği için IRP (I/O Request Package) oluşturulur ve bu paket network kartının driver’ına gider. Network kartı ile OS arasındaki iletişim bu driver ile sağlanır. Network driver’ıne gelen IRP, network driver’ının IRP Queue’sunda sıraya sokulur. Network kartı sırasıyla bu queue’daki istekleri gerçekleştirir. Peki senkron-asenkron programlama ile bu işlemleri gerçekleştirdiğimizde ne değişiyor?

OS tarafından oluşturulan paket (IRP) network kartına gittiğinde network kartı bu işlemi tamamlayana kadar paket “pending” durumunda kalacaktır. Senkron programlama yaptığımızda kullanılan thread (waiting state’de) network kartının bu işlemi tamamlamasını bekleyecektir. Aslında thread bu kısımda beklemese ve network kartı bu işlemi tamamlayana kadar başka işler yapsa verimlilik artacaktır. İşte bu bekleme durumunu ortadan kaldırmak için I/O işlemlerde async/await keyword’lerini kullanarak asenkron programlama yaparız. Bu sayede IRP, network driver’ına gittikten sonra bir süre “pending” durumunda olacağı için thread burada beklemez ve kodların kalan kısmını çalıştırmaya devam eder. awaitkeyword’ünün bulunduğu yerden Task döndürülür ve fonksiyon suspend edilir.Task sınıfı yapılan işlemin tamamlanma bilgisini tutar. Network kartı işlemi tamamladıktan interrupt oluşturarak OS’i IRP’nin “complete” duruma geçtiğine dair haberdar eder. OS de process’i haberdar eder. Sonra CLR’daki ThreadPool’dan bir thread ile gelen bu cevap alınır ve suspend edilip Task döndürülen yerden kodların kalan kısmı çalıştırılmaya devam edilir. Bunu kodlar üzerinden incelemeye çalışalım:

<p><script src="https://gist.github.com/onurkanbakirci/99668218834be9c338b1cc63ccd5968b.js?file=IOBoundAsyncImplementations.cs"></script></p>

Yukarıdaki kodlarda microsoft.com sitesine asenkron olarak istek atılmaktadır. Ayrıca belli aralıklarla da işlemleri gerçekleştiren thread’in id’si konsola basılmaktadır. Yukarıdaki örneğin konsol çıktısı da aşağıdaki gibi olacaktır:

![](/assets/img/posts/threading/iobound.png)

Konsol çıktısından da görüldüğü gibi main thread IOBoundCodeWithAsync()methodunun içine girip await’i gördükten sonra dönen Task henüz tamamlanmamış olduğu için bu fonksiyonu suspend durumuna alıp Main() methoduna dönüp kodları kaldığı yerden çalıştırmaya devam etti. Bu sayede konsola basılan 1 — secondary scopeçıktısının hemen ardından 1 — primary scopeçıktısını görebildik. Taskişlemi tamamlanınca da ThreadPool’dan gelen id’si 13 olan thread, 13 - primary scopeçıktısını konsola yazdırdı. Böylelikle yapılan I/O request uygulamamızı bloklamamış oldu. Burada geliştirdiğimiz uygulamaya göre farklı senaryolar bulunmaktadır. Çünkü context yönetiminin yapıldığı SynchronizationContext sınıfının ASP.NET, Windows Forms, WPF vs. için farklı implementasyon’ları bulunmaktadır. (ASP.NET Core’da custom SynchronizationContextyapısı bulunmamaktadır.) Mesela UI işlemlerinin gerçekleştirildiği bir uygulamamız varsa Task cevabı tamamlandıktan sonra kodların kalan kısmını da UI context’de tamamlamamız gerekmektedir. Aksi halde bir UI thread’in oluşturduğu bir nesne üzerinde ThreadPool thread ile işlem yapmaya çalışmamız durumunda InvalidOperationException hatası alırız. Bu sebeple UI context’e geri dönüp kalan kod kısmını(continuation) burada çalıştırmanız gerekecektir. Fakat context’in çok fazla önem arz etmediği senaryolarda bu context’ler arası geçişi yapmamak maliyetleri düşüreceği için kodunuz daha performanslı olacaktır. İşte bu contextler arası geçişi engelleyip performansı artırmak için awaitable olan kodların sonuna .ConfigureAwait(false); eklenir ve bu sayede tekrardan eski context’e dönmenin maliyetinden kurtulunur.

## CPU Bound

CPU Bound, performansın çok önemli olduğu durumlardır. Bu tip kodların bulunduğu uygulamalarda ağır hesaplamalar yapıyor olabilirsiniz ve milisaniyeler bile sizin için önemli olabilir. Örnek olarak bir mobil oyun düşünelim. Anlık olarak birçok hesaplama birbirine paralel olarak yapılmalı ve oyuncunun herhangi bir donma veya veri gecikmesi hissetmemesi gerekmektedir. Bu tür durumlarda yapılacak olan maliyetli işlemi yeni bir thread’e devredersiniz. Kodu çalıştıran main thread işlemlerine devam ederken ağır işlemin devredildiği bu yeni thread ise CPU’nun başka bir core’unda bu ağır işlemi gerçekleştirir. Bu sayede performans artışı sağlanır. Bunu kodlar üzerinden incelemeye çalışalım:

<p><script src="https://gist.github.com/onurkanbakirci/99668218834be9c338b1cc63ccd5968b.js?file=CPUBoundAsyncImplementations.cs"></script></p>

Yukarıdaki kod örneğinde 100.000 kez iterate edilen bir for döngüsü açıp ağır bir işlemi simüle ettik. Ayrıca belli aralıklarla da işlemleri gerçekleştiren thread’in id’si konsola basılmaktadır. Yukarıdaki örneğin konsol çıktısı da aşağıdaki gibi olacaktır:

![](/assets/img/posts/threading/cpubound.png)

Kodları sırayla çalıştıran main thread, Main() fonksiyonundaki Task.Run(() => CPUBoundCodeWithAsync()) kısmına gelince bu işlemi yeni bir ThreadPool thread’e devreder. Kendisi ise kodların kalan kısmını bloklanmaksızın çalıştırmaya devam eder. Bu sebeple CPUBoundCodeWithAsync() fonksiyonunun içinde thread’in id’sini konsola yazdırdığımızda önceki thread’in id’sinden faklı bir sonuç gördük. primary scope ‘daki kodlar main thread tarafından çalıştırılırken secondary scope ‘daki kodlar ise başka bir thread tarafından çalıştırıldı.

{: .box-note}
Not: Kod örneklerinde gerçek hayat senaryolarına daha yakın olması bakımından Thread.Sleep()ve Task.Delay() fonksiyonları kullanılmadı. Daha uç testler yapıp thread’lerin davaranışlarını daha iyi anlamak için I/O Bound kod örneklerinde Task.Delay(), CPU Bound kod örneklerinde ise Thread.Sleep() fonksiyonlarını kullanabilirsiniz.

Dikkat edilmesi gereken noktalardan biri I/O Bound kodlarda aslında yeni bir thread kullanmıyor oluşumuzdur. Sadece I/O request bittikten sonra gelen cevabı alabilmek için ThreadPool thread kullanılıyor. Fakat CPU Bound kodlarda ise işlemi gerçekleştirebilmek için kodlar ThreadPool thread’de çalıştırılıyor. Aradaki bu nüans farkını bilmek karşılaştığımız senaryolarda doğru işlemi yapabilmek açısından önemli. Çünkü kodlama kısmında yanlış bir implementasyon katkıdan ziyade ya beklenen sonucu vermeyecektir ya da negatif yönde etkiyecektir. Neticede asenkron programlama yaparken kullandığımız Task sınıfı ve kullanılan thread’ler daha fazla memory tüketimine sebep olmaktadır ve GC ‘a daha fazla iş düşmektedir.

{: .box-note}
.NET takımı Task sınıfındaki cacheleme mekanizmasından dolayı oluşan memory maliyetlerinden kazanç sağlayabilmek için belli durumlarda kullanılmak üzere ValueTask ‘ı geliştirmiştir.

Bir sonraki yazıda async/await ve Task kullandığımızda compiler’ın arkada ne yaptığına değinmeye çalışacağız.