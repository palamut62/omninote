# OmniNote İyileştirme Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** OmniNote uygulamasının çizim/ses kararlılığını düzeltmek, tık sesini offline hale getirmek, tanıtım web sayfasını yeşil kimliğe uydurup gerçek APK indirmesi sunmak ve depoyu temizlemek.

**Architecture:** Uygulama tek dosyalık bir Expo/React Native bileşeni (`omninote-app/App.js`). Değişiklikler bu dosyada nokta atışı yapılacak: dokunma tabanlı çizim `PanResponder`'a taşınacak, silgi mantığı bir bayrakla düzeltilecek, izin/temizlik effect'i ayrılacak, uzak tık sesi `expo-haptics` ile değiştirilecek. Web tarafı tek dosya `index.html`. Test altyapısı olmadığından doğrulama, uygulamayı çalıştırıp gözlemleyerek (manuel) ve `npx tsc`/derleme ile yapılır.

**Tech Stack:** Expo SDK 54, React Native 0.81, react-native-svg, expo-av (korunuyor), expo-haptics (yeni), AsyncStorage. Web: statik HTML/CSS. Deploy: Vercel.

---

## Dosya Yapısı

- Modify: `omninote-app/App.js` — çizim, silgi, effect, tık sesi düzeltmeleri + palet sabiti
- Modify: `omninote-app/package.json` — `expo-haptics` ekle, `react-native-paper` kaldır
- Rewrite: `index.html` — yeşil kimlik, gerçek APK linki, iOS/QR kaldırma
- Delete: `node-v24.18.0-x64.msi`, `not-plus.zip`

---

### Task 1: Tık sesini haptics ile değiştir + kullanılmayan bağımlılığı kaldır

**Files:**
- Modify: `omninote-app/package.json`
- Modify: `omninote-app/App.js`

- [ ] **Step 1: package.json bağımlılıklarını güncelle**

`omninote-app/package.json` içindeki `dependencies` bloğunda:
- `"react-native-paper": "4.9.2",` satırını **sil**.
- `"expo-av": "~16.0.8",` satırının altına ekle: `"expo-haptics": "~15.0.7",`

- [ ] **Step 2: Bağımlılıkları kur**

Run: `cd omninote-app && npx expo install expo-haptics`
Expected: expo-haptics uyumlu sürümü kurulur; hata yok. (Ağ yoksa manuel `npm install`.)

- [ ] **Step 3: App.js — import ve tık sesi mantığını değiştir**

`omninote-app/App.js` başındaki `import { Audio } from 'expo-av';` satırının altına ekle:

```javascript
import * as Haptics from 'expo-haptics';
```

`const CLICK_SOUND_URL = 'https://www.soundjay.com/buttons/sounds/button-16.mp3';` satırını **sil**.

`const clickSound = useRef(new Audio.Sound());` satırını **sil**.

`playClickFeedback` fonksiyonunun tamamını şununla değiştir:

```javascript
  // --- DOKUNSAL GERİ BİLDİRİM (OFFLINE, GECİKMESİZ) ---
  const playClickFeedback = async () => {
    try {
      await Haptics.selectionAsync();
    } catch (error) {
      // Haptics desteklenmeyen cihazlarda sessizce geç
    }
  };
```

- [ ] **Step 4: Doğrula (derleme)**

Run: `cd omninote-app && npx expo-doctor || npx expo start --no-dev --max-workers 1` (veya `npx expo start`)
Expected: `CLICK_SOUND_URL`/`clickSound` referansı kalmadığı için derleme hatası yok; Metro bundler temiz başlar.

- [ ] **Step 5: Commit**

```bash
cd omninote-app && git add App.js package.json package-lock.json && git commit -m "fix: replace remote click sound with haptics, drop unused react-native-paper"
```

---

### Task 2: İzin isteme / ses temizliği effect bug'ını düzelt

**Files:**
- Modify: `omninote-app/App.js`

- [ ] **Step 1: İlk yükleme effect'ini `[]` bağımlılığına al ve temizliği ayır**

`omninote-app/App.js` içindeki mevcut `useEffect(() => { loadNotes(); ... }, [playbackSound]);` bloğunun tamamını şununla değiştir:

```javascript
  // --- İLK YÜKLEME (yalnızca bir kez) ---
  useEffect(() => {
    loadNotes();

    const setupAudioPermissions = async () => {
      const { status } = await Audio.requestPermissionsAsync();
      if (status !== 'granted') {
        showToast('Ses kayıt izni verilmedi. Ayarlardan açabilirsiniz.');
      }
    };

    setupAudioPermissions();
  }, []);

  // --- OYNATILAN SESİ BELLEKTEN TEMİZLE ---
  useEffect(() => {
    return () => {
      if (playbackSound) {
        playbackSound.unloadAsync();
      }
    };
  }, [playbackSound]);
```

- [ ] **Step 2: Doğrula (manuel)**

Run: `cd omninote-app && npx expo start`
Uygulamayı bir cihaz/emülatörde aç. Beklenen: Açılışta ses izni istemi **yalnızca bir kez** görünür; ses oynatıp durdurmak izni tekrar tetiklemez; oynatma sonrası bellek uyarısı/çökme olmaz.

- [ ] **Step 3: Commit**

```bash
cd omninote-app && git add App.js && git commit -m "fix: request audio permission once, isolate playback cleanup effect"
```

---

### Task 3: Çizim motorunu PanResponder'a taşı

**Files:**
- Modify: `omninote-app/App.js`

- [ ] **Step 1: PanResponder importunu ekle**

`omninote-app/App.js` en üstteki `react-native` import listesine `PanResponder` ekle (örn. `StatusBar` satırının yanına):

```javascript
  StatusBar,
  PanResponder
```

- [ ] **Step 2: Çizim handler'larını PanResponder ile değiştir**

Mevcut `handleTouchStart`, `handleTouchMove`, `handleTouchEnd` fonksiyonlarının tamamını şununla değiştir:

```javascript
  // --- SVG ÇİZİM İŞLEMLERİ (PanResponder) ---
  const finalizeCurrentPath = (pathStr) => {
    if (!pathStr) return;
    const newPathObj = {
      d: pathStr,
      color: isEraser ? canvasBackground : brushColor,
      width: isEraser ? brushWidth * 3 : brushWidth,
      isEraser: isEraser,
    };
    setPaths((prev) => [...prev, newPathObj]);
    setRedoPaths([]);
    setCurrentPath('');
  };

  const panResponder = useRef(
    PanResponder.create({
      onStartShouldSetPanResponder: () => true,
      onMoveShouldSetPanResponder: () => true,
      onPanResponderGrant: (e) => {
        const { locationX, locationY } = e.nativeEvent;
        setCurrentPath(`M ${locationX.toFixed(1)} ${locationY.toFixed(1)}`);
      },
      onPanResponderMove: (e) => {
        const { locationX, locationY } = e.nativeEvent;
        if (locationX < 0 || locationY < 0 || locationX > SCREEN_WIDTH || locationY > 300) return;
        setCurrentPath((prev) =>
          prev ? `${prev} L ${locationX.toFixed(1)} ${locationY.toFixed(1)}` : `M ${locationX.toFixed(1)} ${locationY.toFixed(1)}`
        );
      },
      onPanResponderRelease: () => {
        setCurrentPath((prev) => {
          finalizeCurrentPath(prev);
          return '';
        });
      },
      onPanResponderTerminate: () => {
        setCurrentPath((prev) => {
          finalizeCurrentPath(prev);
          return '';
        });
      },
    })
  ).current;
```

> Not: `panResponder` bir ref içinde tutulur ama içindeki closure'lar `isEraser`, `brushColor`, `brushWidth` state'lerini okur. RN'de PanResponder ref'i kalıcı olduğundan güncel değerleri okumak için bu handler'lar state setter'larının fonksiyonel formunu kullanır ve `finalizeCurrentPath` çağrısını `onPanResponderRelease` içinde `currentPath` üzerinden yapar. `isEraser`/`brushColor`/`brushWidth` doğrudan okunduğu için, ref closure eski değeri yakalayabilir — bunu önlemek için Step 3'te ref yerine `useRef` yeniden oluşturma yerine güncel değerleri ref'te tutan yardımcı ref kullanılır.

- [ ] **Step 3: Güncel çizim ayarlarını ref'te tut (stale closure önleme)**

Yukarıdaki `panResponder` tanımından **önce** şunu ekle:

```javascript
  const drawSettingsRef = useRef({ isEraser, brushColor, brushWidth });
  useEffect(() => {
    drawSettingsRef.current = { isEraser, brushColor, brushWidth };
  }, [isEraser, brushColor, brushWidth]);
```

Ardından `finalizeCurrentPath` içindeki `isEraser`, `brushColor`, `brushWidth` okumalarını `drawSettingsRef.current`'tan al:

```javascript
  const finalizeCurrentPath = (pathStr) => {
    if (!pathStr) return;
    const { isEraser: er, brushColor: bc, brushWidth: bw } = drawSettingsRef.current;
    const newPathObj = {
      d: pathStr,
      color: er ? canvasBackground : bc,
      width: er ? bw * 3 : bw,
      isEraser: er,
    };
    setPaths((prev) => [...prev, newPathObj]);
    setRedoPaths([]);
    setCurrentPath('');
  };
```

- [ ] **Step 4: SVG tuvalini panHandlers ile bağla**

`svgCanvas` View'indeki `onTouchStart={handleTouchStart}`, `onTouchMove={handleTouchMove}`, `onTouchEnd={handleTouchEnd}` proplarını **sil** ve yerine ekle:

```javascript
                <View
                  style={styles.svgCanvas}
                  {...panResponder.panHandlers}
                >
```

Aynı View içindeki canlı çizim (currentPath) Path'inde silgi rengini de güncel ref'ten okumaya gerek yok — mevcut `isEraser ? canvasBackground : brushColor` render sırasında güncel state okur, olduğu gibi kalır.

- [ ] **Step 5: Doğrula (manuel)**

Run: `cd omninote-app && npx expo start`
Editörü aç, çizim alanında **hızlı** çiz. Beklenen: Kesintisiz düz çizgi bırakır (kaçırma yok); çizim sırasında sayfa dikey kaymaz; kalem rengi/kalınlığı seçimi sonraki çizime doğru uygulanır.

- [ ] **Step 6: Commit**

```bash
cd omninote-app && git add App.js && git commit -m "fix: migrate drawing engine to PanResponder for reliable strokes"
```

---

### Task 4: Silgi bug'ını düzelt (renkli kart önizlemesi)

**Files:**
- Modify: `omninote-app/App.js`

- [ ] **Step 1: Kart mini önizlemesinde silgi çizgilerini filtrele**

Task 3'te path objelerine `isEraser` bayrağı eklendi. Kart render'ındaki mini önizleme SVG'sinde (`item.paths.slice(0, 15).map(...)`) silgi path'lerini atla. Mevcut bloğu şununla değiştir:

```javascript
            {item.paths && item.paths.length > 0 && (
              <View style={styles.miniCanvasPreview}>
                <Svg style={StyleSheet.absoluteFill}>
                  {item.paths.filter((p) => !p.isEraser).slice(0, 15).map((p, idx) => (
                    <Path
                      key={idx}
                      d={p.d}
                      stroke={p.color}
                      strokeWidth={p.width / 4}
                      fill="none"
                    />
                  ))}
                </Svg>
              </View>
            )}
```

- [ ] **Step 2: Mini önizleme zeminini nötr yap**

`styles.miniCanvasPreview` içindeki `backgroundColor: 'rgba(0,0,0,0.02)'` değerini tuval zeminiyle tutarlı olacak şekilde `backgroundColor: '#F9F9F9'` yap.

- [ ] **Step 3: Doğrula (manuel)**

Run: `cd omninote-app && npx expo start`
Beyaz olmayan bir kart rengi seç (ör. `#EAF2FF`), çiz, sonra silgiyle bir kısmını sil, kaydet. Beklenen: Kart önizlemesinde silinen bölge **beyaz/gri çizgi bırakmaz**; ana tuvalde silgi görünür şekilde temizler.

- [ ] **Step 4: Commit**

```bash
cd omninote-app && git add App.js && git commit -m "fix: skip eraser strokes in card preview, neutral preview background"
```

---

### Task 5: Uygulama içi renk paletini sabitle (tutarlılık)

**Files:**
- Modify: `omninote-app/App.js`

- [ ] **Step 1: Palet sabitini ekle**

`omninote-app/App.js` içinde `const { width: SCREEN_WIDTH, ... } = Dimensions.get('window');` satırının altına ekle:

```javascript
// --- MARKA PALETİ (açık + yeşil) ---
const BRAND = {
  accent: '#34C759',
  accentDark: '#2AA149',
  bg: '#F5F7FA',
  card: '#FFFFFF',
  border: '#E2E8F0',
  textStrong: '#1A202C',
  textBody: '#4A5568',
  textMuted: '#A0AEC0',
};
```

- [ ] **Step 2: Vurgu rengini styles'ta BRAND üzerinden kullan**

`styles` içinde vurgu yeşilinin sabit yazıldığı yerleri `BRAND.accent` ile değiştir (StyleSheet.create bir fonksiyon çağrısı olduğundan sabit obje referansı geçerlidir). Şu üç yeri güncelle:
- `categoryTabActive: { backgroundColor: '#34C759' }` → `backgroundColor: BRAND.accent`
- `floatingActionButton: { ... backgroundColor: '#34C759', ... shadowColor: '#34C759', ... }` → her ikisi `BRAND.accent`
- `editorSaveText: { color: '#34C759', ... }` → `color: BRAND.accent`

- [ ] **Step 3: Doğrula (manuel)**

Run: `cd omninote-app && npx expo start`
Beklenen: Görsel olarak hiçbir değişiklik yok (aynı yeşil); uygulama sorunsuz çalışır. Bu adım yalnızca ileride tek yerden renk yönetimi sağlar.

- [ ] **Step 4: Commit**

```bash
cd omninote-app && git add App.js && git commit -m "refactor: centralize brand palette constant"
```

---

### Task 6: Web sayfasını yeşil kimliğe uydur + gerçek APK indirmesi

**Files:**
- Rewrite: `index.html`

- [ ] **Step 1: index.html'i yeniden yaz**

`C:/Users/umuti/Desktop/Aras_app/index.html` dosyasının tamamını şu içerikle değiştir:

```html
<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>OmniNote - Fikirlerinizi Özgür Bırakın</title>
    <style>
        :root {
            --accent: #34C759;
            --accent-dark: #2AA149;
            --bg: #F5F7FA;
            --card: #FFFFFF;
            --border: #E2E8F0;
            --text-strong: #1A202C;
            --text-body: #4A5568;
            --text-muted: #A0AEC0;
        }
        * { box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            margin: 0; padding: 0; background-color: var(--bg); color: var(--text-strong);
            display: flex; flex-direction: column; align-items: center; justify-content: center;
            min-height: 100vh; text-align: center;
        }
        .container { max-width: 720px; padding: 48px 20px; }
        h1 {
            font-size: 3.5rem; margin-bottom: 10px; font-weight: 800;
            color: var(--accent);
        }
        .lede { font-size: 1.2rem; color: var(--text-body); margin-bottom: 40px; line-height: 1.5; }
        .btn-group { display: flex; gap: 16px; justify-content: center; flex-wrap: wrap; margin-bottom: 32px; }
        .btn {
            display: inline-block; padding: 16px 32px; font-size: 1.1rem; font-weight: 700;
            text-decoration: none; border-radius: 30px; transition: all 0.25s ease;
            box-shadow: 0 6px 18px rgba(52, 199, 89, 0.25);
        }
        .btn-android { background-color: var(--accent); color: #fff; }
        .btn-android:hover { background-color: var(--accent-dark); transform: translateY(-3px); }
        .instructions {
            background: var(--card); padding: 28px; border-radius: 16px; text-align: left;
            margin-top: 24px; border: 1px solid var(--border); border-left: 4px solid var(--accent);
            box-shadow: 0 4px 20px rgba(0,0,0,0.03);
        }
        .instructions h3 { margin-top: 0; color: var(--accent-dark); }
        .instructions ol { padding-left: 20px; color: var(--text-body); }
        .instructions li { margin-bottom: 12px; line-height: 1.5; }
        .foot { margin-top: 28px; color: var(--text-muted); font-size: 0.85rem; }
    </style>
</head>
<body>
    <div class="container">
        <h1>OmniNote</h1>
        <p class="lede">Hızlı, pratik ve her an yanınızda. Notlarınız, çizimleriniz ve ses kayıtlarınız tek bir uygulamada.</p>

        <div class="btn-group">
            <!-- APK web sayfasıyla birlikte deploy edilir ve doğrudan indirilir -->
            <a href="/omninote.apk" class="btn btn-android" download>🤖 Android (APK) İndir</a>
        </div>

        <div class="instructions">
            <h3>⚙️ Kurulum Rehberi (Android)</h3>
            <ol>
                <li>Yukarıdaki <strong>Android (APK) İndir</strong> butonuna basın.</li>
                <li>İndirme bitince APK dosyasını açın.</li>
                <li>Telefonunuz güvenlik uyarısı verirse, ayarlardan tarayıcınıza/dosya yöneticinize <strong>"Bilinmeyen kaynaklardan uygulama yükleme"</strong> iznini verin.</li>
                <li>Kurulumu tamamlayın ve OmniNote'u açın. 🎉</li>
            </ol>
        </div>

        <p class="foot">iOS sürümü yakında.</p>
    </div>
</body>
</html>
```

- [ ] **Step 2: APK'nın sayfa köküyle aynı yerde olduğunu doğrula**

Run: `ls -la "C:/Users/umuti/Desktop/Aras_app/omninote.apk"`
Expected: Dosya var. `/omninote.apk` linki, `index.html` ile aynı kökten servis edildiğinde çalışır.

- [ ] **Step 3: Yerel doğrulama (manuel)**

`index.html`'i tarayıcıda aç. Beklenen: Yeşil kimlik, tek "Android (APK) İndir" butonu, sahte iOS/QR yok, "iOS yakında" notu görünür.

- [ ] **Step 4: Commit**

```bash
git add index.html && git commit -m "feat: redesign landing page to green brand, serve real APK, drop fake iOS/QR" 2>/dev/null || echo "kök repo yok - Task 8'de ele alınacak"
```

---

### Task 7: Depo temizliği

**Files:**
- Delete: `node-v24.18.0-x64.msi`, `not-plus.zip`

- [ ] **Step 1: Gereksiz büyük dosyaları sil**

Run:
```bash
rm -f "C:/Users/umuti/Desktop/Aras_app/node-v24.18.0-x64.msi" "C:/Users/umuti/Desktop/Aras_app/not-plus.zip"
```
Expected: İki dosya silinir. `not-plus/` klasörü ve `omninote.apk` **korunur**.

- [ ] **Step 2: Doğrula**

Run: `ls -la "C:/Users/umuti/Desktop/Aras_app/"`
Expected: `.msi` ve `.zip` yok; `index.html`, `omninote-app/`, `not-plus/`, `omninote.apk`, `docs/` var.

---

### Task 8: Vercel deploy hazırlığı ve son doğrulama

**Files:**
- Create: `vercel.json` (kök)

- [ ] **Step 1: APK için doğru içerik tipi ve indirme ayarı**

`C:/Users/umuti/Desktop/Aras_app/vercel.json` oluştur:

```json
{
  "headers": [
    {
      "source": "/omninote.apk",
      "headers": [
        { "key": "Content-Type", "value": "application/vnd.android.package-archive" },
        { "key": "Content-Disposition", "value": "attachment; filename=omninote.apk" }
      ]
    }
  ]
}
```

- [ ] **Step 2: Vercel boyut limitini kontrol et (manuel)**

Not: `omninote.apk` ~66 MB. Vercel statik dosya limiti aşılırsa B planı: APK'yı GitHub Release veya harici storage'a yükleyip `index.html`'deki `href="/omninote.apk"` linkini o mutlak URL ile değiştir.
Run: `du -h "C:/Users/umuti/Desktop/Aras_app/omninote.apk"`
Expected: ~66M. Deploy sonrası indirme linkini gerçek tarayıcıda test et.

- [ ] **Step 3: Son uçtan uca doğrulama (manuel)**

Uygulama: `cd omninote-app && npx expo start` → çizim (hızlı), silgi (renkli kart), ses kaydet/oynat, fotoğraf ekle, not kaydet/düzenle/sil, favori, arama, kategori sekmeleri — hepsi çalışır; açılışta izin tek kez sorulur.
Web: Deploy sonrası sayfa açılır, APK butonu gerçek dosyayı indirir.

- [ ] **Step 4: Commit**

```bash
git add vercel.json 2>/dev/null && git commit -m "chore: vercel apk headers + repo cleanup" 2>/dev/null || echo "kök repo yoksa: git init sonrası tek commit"
```

---

## Notlar
- **Kök repo:** Proje kökü şu an git deposu değil. Web/kök değişikliklerini commit'lemek için `git init` gerekebilir; kullanıcı onayıyla yapılır. `omninote-app/` kendi repo'suna sahip, uygulama commit'leri oraya gider.
- **expo-av:** SDK 55'te kalkacak; ayrı bir tur olarak `expo-audio` migrasyonu ileride planlanmalı (bu planın kapsamı dışında).
