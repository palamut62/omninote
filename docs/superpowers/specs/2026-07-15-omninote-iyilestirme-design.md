# OmniNote İyileştirme — Tasarım Dokümanı

Tarih: 2026-07-15

## Amaç
OmniNote (Expo/React Native not uygulaması) ve tanıtım web sayfasını; kararlılık,
tutarlı görsel kimlik ve yayına hazırlık açısından iyileştirmek. Yeni özellik
eklenmeyecek — mevcut işlevler sağlamlaştırılacak ve sunum düzeltilecek.

## Kapsam Dışı (YAGNI)
- Yeni özellikler (etiket, klasör, senkronizasyon, dışa aktarma vb.)
- App.js'in modüllere bölünmesi (tek dosyada kalınacak)
- expo-av → expo-audio migrasyonu (sonraya bırakıldı)
- `not-plus/` klasörünün silinmesi (şimdilik duruyor)

## Görsel Kimlik: Açık + Yeşil
Hem uygulama hem web için ortak token seti:
- Vurgu: `#34C759`, hover/koyu: `#2AA149`
- Zemin: `#F5F7FA`, kart: `#FFFFFF`, kenar: `#E2E8F0`
- Metin: başlık `#1A202C`, gövde `#4A5568`, soluk `#A0AEC0`
- Tipografi: sistem fontu, tutarlı ölçek
- Yuvarlatma/gölge/boşluk: uygulamadaki mevcut dil korunur.

Web sayfasındaki mor gradyan (`#4A00E0`/`#8E2DE2`) bu yeşil kimliğe çevrilecek.

## İş Kolu 1 — Uygulama Kararlılığı (omninote-app/App.js, tek dosya)
1. **Çizim motoru → PanResponder:** `onTouchStart/Move/End` yerine `PanResponder`.
   Hızlı çizimde koordinat kaçırma ve ScrollView gesture çakışması giderilecek.
2. **Silgi düzeltmesi:** Arka planı boyamak yerine doğru silme davranışı
   (path kaldırma veya doğru zemin rengi). Beyaz olmayan not renklerinde bozulma biter.
3. **useEffect bug'ı:** İlk yükleme + izin isteme `[playbackSound]` bağımlılığından
   çıkarılıp `[]`'e alınacak; ses temizliği (unload) ayrı bir effect'e taşınacak.
   Tekrarlı izin isteme sona erer.
4. **Tık sesi:** Uzak `soundjay.com` mp3 kaldırılacak; yerine `expo-haptics`
   dokunsal geri bildirim. Offline çalışır, gecikmesiz. (yeni küçük bağımlılık)
5. **expo-av:** SDK 54'te korunuyor; yalnızca yukarıdaki useEffect bug'ı düzeltiliyor.
6. **react-native-paper:** Kullanılmıyor, package.json'dan kaldırılacak.

### Kabul Kriterleri (İş Kolu 1)
- Hızlı çizim kesintisiz çizgi bırakır; çizim sırasında sayfa kaymaz.
- Renkli (beyaz olmayan) notta silgi görünür şekilde temizler, yanlış renk bırakmaz.
- Uygulama açılışında izin isteme yalnızca bir kez tetiklenir.
- Buton etkileşimlerinde ağ bağımlılığı olmadan dokunsal geri bildirim alınır.
- `react-native-paper` bağımlılıklardan kaldırılmış, uygulama sorunsuz derlenir/çalışır.

## İş Kolu 2 — Web Sayfası (index.html, Vercel)
- Mor gradyan → yeşil kimlik; uygulamayla tutarlı görünüm.
- Android APK butonu doğrudan `omninote.apk` dosyasını indirir (dosya sayfayla deploy edilir).
- iOS/Expo bölümü ve sahte QR kodu kaldırılır.
- "Bilinmeyen kaynak izni" içeren kısa kurulum rehberi korunur.
- Vercel notu: 66 MB APK statik olarak deploy edilecek. Hobby planı limiti sorun
  çıkarırsa B planı: APK'yı harici storage/Release'e taşıyıp linki oradan vermek.

### Kabul Kriterleri (İş Kolu 2)
- Sayfa yeşil kimlikte, uygulamayla tutarlı.
- Tek indirme butonu gerçek `omninote.apk`'yı indirir; sahte link/QR kalmaz.
- Kurulum rehberi net ve doğru.

## İş Kolu 3 — Depo Temizliği
- Kaldırılacak: `node-v24.18.0-x64.msi`, `not-plus.zip`.
- Korunacak: `not-plus/` klasörü (şimdilik), `omninote.apk` (web deploy için gerekli).

### Kabul Kriterleri (İş Kolu 3)
- Gereksiz kurulum/arşiv dosyaları depodan çıkar.
- Uygulama ve web dosyaları etkilenmez.

## Riskler
- PanResponder geçişi çizim davranışını değiştirebilir → manuel test gerekli.
- Vercel dosya boyutu limiti → B planı hazır.
- expo-av gelecekte (SDK 55) kalkacak → ayrı bir tur olarak not edildi.
