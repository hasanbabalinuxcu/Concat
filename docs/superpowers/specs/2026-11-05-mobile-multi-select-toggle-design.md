# Mobil Çoklu Seçim Toggle'u — Tasarım

## Niyet
Telefonda timeline kliplerini tek tek dokunarak çoklu seçmek; böylece seçili klipler merge'lenebilsin. Bugün `additive` seçim yalnız Shift+click ile geliyor; mobilde Shift yok.

## Hedefler
- Tray'de toggle butonu: açıkken klibe dokunmak seçime ekler, tekrar dokunmak çıkarır.
- Toggle kapalıyken mevcut davranış aynen korunur (tek dokunuş tek seçim).
- Band/marquee seçimi de toggle açıkken additive olur.
- Merge butonu mevcut `selected-count` / `merge-blocked` mantığıyla otomatik çalışır.

## Hedef Değil
- Yeni bir seçim modeli yazmak (model `Studio.selection: Vec<String>` zaten hazır).
- Undo/redo veya inspector davranışında değişiklik.

## Yaklaşımlar
1. **Tray toggle + additive OR** (seçilen): razor/split toggle kalıbının aynısı; `lib.rs` handler'da `additive = shift || multi-select`. En az değişiklik, mevcut akışa tam uyum.
2. Telefonda marquee'yi yeniden etkinleştir: dokunmatik gezinmeyle çakışır, kullanıcı zaten tek tek dokunmak istiyor — elendi.
3. Long-press ile seçim modu: sürüklenme/scrub ile yarışır, keşfedilirliği düşük — elendi.

## Karar
Yaklaşım 1.

## Parçalar
1. **Rust durumu**: `panes/timeline.rs` → `TimelinePane.multi_select: bool` + `TimelineMsg::MultiSelectChanged(bool)` (pan-mode kalıbı). `lib.rs` → `on_multi_select_changed` handler; `on_clip_pressed` ve `on_band_selected` içinde `additive = pointer_additive || state.lanes.multi_select`. `publish.rs` → `set_multi_select`.
2. **Masaüstü tray**: `editor.slint` global `multi-select` property + callback; `seat.slint` → `timeline-pane.slint` → `tray.slint` zinciri (pan-mode zincirinin aynısı); tray'e `TrayButton` (`Glyph.check`, `checked: root.multi-select`).
3. **Telefon çubuğu** (`ui/phone/editor.slint`): seçim yokken görünen bara VE seçili klip barına birer `ToolItem` (`current: Editor.multi-select`, aynı glyph/etiket). İki bara da konur ki mod açıkken kullanıcı butonu her durumda görebilsin/kapatabilsin.
4. **Merge telefonda zaten var**: klip "More" (↑) menüsü `keyframes.rs` clip-context menüsünde `merge` eylemini `merge_blocked()`'a bağlı olarak taşıyor; çoklu seçim açılınca merge oradan çalışır. Yeni merge butonu eklenmez.
5. **i18n**: `tray.multiSelect` anahtarı (en + tr), `scripts/locales.py` senkronu.
6. **Test**: Depo `Studio` fixture'ı olmadan state-machine testine izin vermiyor (testler saf fonksiyonlar); doğrulama `cargo check/clippy/fmt` + `locales.py --check` + CI (nextest, clippy -D warnings, android smoke) ile.

## Derleme / Teslim
- Yerel Termux derlemesi yerine GitHub Actions: commit + push → `.github/workflows/mobile.yml` (`on: push: branches: [main]`) `Concat-android-arm64` artifact'ı üretir.
- Push için GitHub kimliği gerekir (bu ortamda `gh auth` yok); planın son adımımda kullanıcıdan giriş istenir.

## Kapsam Dışı Kararlar (builder call)
- Toggle ikonu: mevcut glyph setinden seçim (ör. seçim çerçevesi/glyph), tasarım detayı.
