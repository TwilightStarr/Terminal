# Terminal — Cihaz ve Sistem Bilgi Paneli

Godot 4 ile hazırlanmış, AMOLED ekranlara uygun (tam siyah zemin + neon vurgular) bir sistem/cihaz bilgi paneli. Arayüz `scripts/Main.gd` içinde çalışma anında koddan kuruluyor; `.tscn` dosyası bu yüzden minimal — Godot düzenleyicisinde bu scripti açıp dilediğiniz gibi genişletebilirsiniz.

## Neler var?

- **Sekmeli ana ekran**: Kartlar "Genel Bakış", "Cihaz", "Performans" ve "Kontroller" sekmelerine ayrılmış durumda; üstte özel (tema renkleriyle uyumlu) bir segmented-control ile geçiş yapılıyor
- **Pil**: durum (pilde / şarj oluyor / doldu), yüzde, kalan süre, görsel şarj çubuğu
- **Cihaz**: model, işletim sistemi, Godot sürümü, işlemci, çekirdek sayısı, dil
- **Ekran**: çözünürlük, DPI, yenileme hızı, ölçek
- **Bellek / Performans**: kullanılan bellek (görsel çubuklu), zirve bellek, FPS ve son 30 saniyeyi gösteren mini FPS çizgi grafiği
- **Depolama**: uygulama veri klasörünün boyutu ve yolu
- **Ağ**: yerel IP adresi
- **Kontroller**: ekranı açık tutma anahtarı (tercih kalıcı olarak saklanır), titreşim testi, "Gerekli İzinleri İste" butonu
- **Üst çubuk**: manuel yenileme butonu, son güncelleme saati ve hangi sekmede olunursa olunsun görünen kompakt pil rozeti (ikon + yüzde, duruma göre renklenir)

Veriler saniyede bir otomatik güncellenir; her sekmeye ilk geçişte o sekmenin kartları sırayla belirerek (fade-in) görünür.

## Geliştirme Adımları

- [Yapıldı] Depolama kartı eklendi (uygulama veri klasörü boyutu + yolu, `DirAccess` ile hesaplanıyor)
- [Yapıldı] Pil yüzdesi ve bellek kullanımı için görsel `ProgressBar` göstergeleri eklendi (pil düşükken renk kırmızıya/turuncuya dönüyor)
- [Yapıldı] Manuel "Şimdi Yenile" butonu ve "son güncelleme" saati üst çubuğa eklendi
- [Yapıldı] "Ekranı Açık Tut" tercihi `ConfigFile` ile `user://terminal_settings.cfg` içine kaydediliyor, uygulama yeniden açıldığında hatırlanıyor
- [Yapıldı] Kartlar için giriş animasyonu (sırayla fade-in) eklendi
- [Yapıldı] **Ana ekran sekme/bölüm yapısı**: Kartlar "Genel Bakış" (Pil, Depolama, Ağ), "Cihaz" (Cihaz, Ekran), "Performans" (Bellek/FPS) ve "Kontroller" olmak üzere 4 sekmeye ayrıldı. `TabContainer` yerine, projenin mevcut buton/stil diliyle (StyleBoxFlat tabanlı) tutarlı olması için özel bir segmented-control (ButtonGroup + toggle butonlar) kullanıldı. Sekme değiştirildiğinde kaydırma konumu sıfırlanıyor ve ilgili sekmenin kartları yeniden fade-in ile giriyor.
- [Yapıldı] **Üst çubukta her zaman görünen kompakt pil rozeti**: Başlık satırının yanına (yenile butonundan önce) ikon + yüzde içeren küçük bir rozet eklendi. Şarj olurken/dolduğunda ikon yıldırıma dönüyor, yüzde ve kenarlık rengi pil seviyesine göre (yeşil → turuncu ≤%50 → kırmızı ≤%20) değişiyor. Renk mantığı `_battery_accent_color()` adıyla ortak bir fonksiyona çıkarıldı; hem "Genel Bakış" sekmesindeki `ProgressBar` hem de rozet aynı fonksiyonu kullanıyor, böylece iki yerde ayrı ayrı bakım gerekmiyor. Rozet güncellemesi `_refresh_header_battery_badge()` içinde, mevcut `_refresh_dynamic_info()` akışının sonunda tetikleniyor — yeni bir zamanlayıcıya gerek kalmadı.

- [Yapıldı] **FPS için son birkaç saniyeyi gösteren mini çizgi grafik**: "Performans" sekmesindeki FPS satırının hemen altına "Son 30 Saniye" etiketli, sabit boyutlu (54 px yükseklik) bir mini çizgi grafik eklendi. Grafik, `Main.gd` içinde tanımlanan hafif bir iç sınıf olan `FPSGraphView` (Control'den türetilir) ile çiziliyor; tek sorumluluğu `_draw()` içinde kendi `history` dizisini `draw_polyline` ile çizmek. Sabit boyutlu kuyruk mantığı `_push_fps_sample()` fonksiyonunda: her `_refresh_dynamic_info()` çağrısında yeni FPS örneği `fps_history` dizisine ekleniyor, dizi `FPS_HISTORY_MAX` (30) örneği aşınca `pop_front()` ile en eski örnek atılıyor. Düşüşlerin sayı olarak değil görsel olarak da fark edilmesi için çizgi rengi anlık FPS'e göre değişiyor (`_battery_accent_color()`'daki mantığa paralel bir eşiklemeyle: 30 altı kritik/kırmızı, 50 altı uyarı/turuncu, üzeri normal/yeşil). Yeni bir `Timer`'a veya harici node/sahneye ihtiyaç duyulmadı; mevcut saniyelik güncelleme döngüsüne oturdu.

## Sıradaki Geliştirme Adımı (önerilen, en yüksek öncelik)

- [ ] **Satır değerlerine dokunarak panoya kopyalama** *(işlevselliğin artırılması)*: Özellikle "Yerel IP" ve "Model" gibi değerler genellikle başka bir yere (SSH istemcisi, destek talebi, not vb.) elle yazılıyor; bu da hataya açık ve yavaş. `_add_row()` fonksiyonuna kopyalanabilir satırlar için opsiyonel bir `copyable: bool` parametresi eklemek, değer `Label`'ını (ya da üzerine şeffaf bir `Button`/`gui_input` yakalayan bir `Control` kaplamasını) tıklanabilir hale getirip `DisplayServer.clipboard_set(value_labels[key].text)` çağırmak yeterli. Geri bildirim için değer metninin rengi kısa süreliğine `COLOR_ACCENT_CYAN`'a dönüp eski rengine fade ile dönebilir (mevcut `create_tween()` kullanım deseniyle, kartların fade-in animasyonunda olduğu gibi). Düşük karmaşıklıkta, mevcut `value_labels` sözlüğüne ve satır oluşturma deseninine doğrudan oturan bir ekleme; bu yüzden bir sonraki en yüksek öncelikli adım olarak öne çekildi.

## Diğer Geliştirme Fikirleri (sırayla ele alınabilir)

- [ ] Sekmeler arasında yatay kaydırma (swipe) jesti ile geçiş desteği (dokunmatik cihazlarda üstteki butonlara ek olarak)
- [ ] Kontroller sekmesine, uygulama açılışında hangi sekmenin gösterileceğini seçen bir ayar eklenmesi (mevcut `ConfigFile` mekanizmasıyla kalıcı saklanır)
- [ ] Kartlara dokunulduğunda genişleyip daralan (expand/collapse) detay görünümü
- [ ] Ayarlar kartına tema rengi seçici (yeşil/camgöbeği dışında ek vurgu renkleri)
- [ ] Pil geçmişi: son X dakikadaki şarj yüzdesi değişimini, FPS grafiğinde kullanılan `FPSGraphView` sınıfı genelleştirilerek (renk/aralık parametreleri dışarıdan verilebilir hale getirilip) aynı çizim mantığıyla gösterme
- [ ] Bildirim/izin durumları için tek tek liste yerine ikonlu satır göstergeleri (verildi ✓ / reddedildi ✗)
- [ ] Üst çubuktaki pil rozetine dokununca "Genel Bakış" sekmesine otomatik geçiş (rozeti yalnızca gösterge değil kısayol haline getirir)
- [ ] Ana ekrana, uygulama ilk açıldığında (henüz hiç sekme seçilmemişken) kısa bir karşılama/özet kartı — cihaz adı ve tarih/saat gibi tek bakışta özet bilgi
- [ ] Kart başlıklarındaki `›` ikonunun yanına, o kartın verisi son yenilemeden bu yana değiştiyse kısa süreliğine yanıp sönen küçük bir "güncellendi" noktası eklenmesi
- [ ] **(yeni fikir — grafik UI iyileştirmesi)** "Performans" sekmesindeki FPS mini grafiğine dokunulduğunda, aynı `FPSGraphView` mantığıyla daha uzun bir geçmişi (ör. son 60 saniye) gösteren büyütülmüş bir görünüm açılması
- [ ] **(yeni fikir — ana ekran düzenlemesi)** Uygulama ilk açıldığında, statik/dinamik veriler henüz okunmamışken kısa bir yükleniyor/iskelet (skeleton) durumu göstermek; böylece ilk karede "-" dolu kartlar yerine daha temiz bir geçiş yaşanır

## Godot çekirdeğinin sınırları (önemli)

Pil, bellek, ekran ve cihaz bilgisi Godot'un temel API'siyle doğrudan okunabildiği için bunlar gerçek veriyle çalışıyor. Ama **fener/flaş açma, ekran parlaklığını değiştirme, IMEI/sinyal gücü gibi detaylı telefon-operatör bilgisi** Godot çekirdeğinde yok — bunlar için Java/Kotlin ile yazılmış özel bir Android eklentisi (GDExtension) gerekir.

## Renk paleti (AMOLED)

| Kullanım           | Renk      |
|--------------------|-----------|
| Zemin              | `#000000` |
| Kart zemini        | `#0a0a0a` |
| Kart kenarlığı     | `#1c1c1c` |
| Ana metin          | `#e8e8e8` |
| İkincil metin      | `#7a7a7a` |
| Vurgu (yeşil)      | `#00e676` |
| Vurgu (camgöbeği)  | `#18ffff` |
| Uyarı              | `#ffab00` |
| Kritik             | `#ff5252` |

Bu renkleri `scripts/Main.gd` dosyasının en üstündeki `const COLOR_...` satırlarından değiştirebilirsiniz.

## Sonraki adım

Projeyi Godot 4.x'te açıp `scripts/Main.gd` üzerinden arayüzü test edin, sekmeler arasında geçiş yaparak yeni yapıyı doğrulayın, ardından ihtiyacınıza göre yeni bilgi kartları veya kontroller ekleyerek geliştirmeye devam edin.
# Terminal
