# Vizyon

## Tek Cümle

**Skillvio**, tech alanlarında çoklu dil destekli, oyunlaştırılmış bir öğrenme platformudur.
Roadmap'lerle yol gösterir, Duolingo tarzı ünitelerle pratik yaptırır, TryHackMe tarzı
laboratuvarlarla elini kirletir; sıfırdan başlayanı uzmana taşır.

## Problem

Türkiye'deki yazılımcılar / yazılıma geçmek isteyenler ortak 5 sorunla karşılaşıyor:

1. **Toplantıda duydukları İngilizce terimleri anlamamak** — *blocker*, *EOD*, *scope creep*,
   *RAG*, *LLM*…
2. **Hangi sırayla ne öğreneceklerini bilmemek** — frontend mi backend mi? hangi yol?
3. **Sertifika sınavı için yeterince Türkçe kaynağın olmaması.**
4. **Kuru tanım yerine pratik öğrenme istemeleri** — yapılandırılmış quiz, lab, roadmap.
5. **Motivasyonu sürdürememek** — XP, streak, rozet, leaderboard.

## Çözüm

Tek çatı altında **3 katman**:

### 📚 Üniteler (Duolingo tarzı)
- Konsept kartı + soru + boşluk doldurma + eşleştirme + sıralama + çeviri
- 5–10 dakikalık bite-sized dersler
- Doğru cevap → kutlama animasyonu, XP, day streak

### 🛤 Roadmap'ler (roadmap.sh tarzı)
- Kullanıcı ilgi alanını seçer (cloud / code / AI / DB / DevOps / data / security…)
- Her kariyer/teknoloji yolu **node ağacı**: kilit → açılma mekaniği
- Her node birden fazla kaynağa bağlı: lesson, lab, link, video

### 🧪 Laboratuvarlar (TryHackMe tarzı)
- **Faz 1:** Tarayıcı içi sandbox — HTML/CSS/JS (iframe), Python (Pyodide), SQL (sql.js)
- **Faz 2:** Container terminal — Bash, Docker, Linux CLI (xterm.js + Docker.DotNet)
- **Faz 3:** Cloud lab — LocalStack ile sahte AWS, gerçek CLI

## Hedef Kitle

| Persona | İhtiyacı | Nasıl çözeriz |
|---------|----------|---------------|
| **Sıfırdan başlayan** | "Frontend olmak istiyorum, nereden başlayayım?" | Roadmap |
| **Junior geliştirici** | "Sertifika almak istiyorum" | Sınav modu + AI açıklama |
| **Kariyer değiştiren** | "DevOps'a geçiyorum" | Yol + iş terimleri |
| **Çalışan profesyonel** | "Toplantıda duyduğum terimleri öğreneyim" | Glossary + IT EN |
| **Meraklı/öğrenci** | "AI nasıl çalışıyor?" | AI/ML terimler yolu |

## Değer Önerisi

- 🇹🇷 **Türkçe-öncelikli** (sonra EN, DE, AR…)
- 🛤 **Yol haritası** ile kullanıcı kayıp hissetmez
- 🎯 **Sertifika sınav simülasyonu** (gerçekçi)
- 🧪 **Editör tabanlı lab** (gerçek kod yazma)
- 🎮 **Gamification** (XP, level, day streak, hearts, leaderboard)
- 🤖 **AI tutor** (Gemini ile cümle cümle Türkçe açıklama)
- 🏅 **Domain bazlı rozet** (cloud/code/ai/db ayrı setler)
- 🎓 **Sertifika cüzdanı** (gerçek hayatta alınanlar profilde gözükür)

## İş Modeli (öneri — final değil)

| Plan | Fiyat | Kim için |
|------|-------|----------|
| **Free** | 0₺ | 1 alan, sınırlı can, reklamsız ama AI rate-limit'li |
| **Pro** | 79₺/ay | Tüm alanlar, sınırsız can, premium AI tutor, sertifika onayı öncelikli |
| **Team** | 299₺/ay (5 kişi) | Şirket içi sınıflar, ortak leaderboard, raporlama |

## Başarı Göstergeleri (12. ay hedefi)

- 5.000 aktif kullanıcı
- 500 ücretli abone
- 30+ roadmap, 100+ unit, 200+ lab
- Tek-cihazda günlük 7+ dk ortalama kullanım
- 60% 7-günlük retention

## Marka

- **İsim:** Skillvio
- **Tagline (TR):** *"Bilgide ustalaş, yolda kalma."*
- **Tagline (EN):** *"Master your skills, one path at a time."*
- **Renk paleti (mevcut):** Turuncu (brand) + Cyan (action) + Yeşil (success)
- **Çoklu dil hazır:** TR, EN, DE, AR (i18n altyapısıyla)

## Neyi Yapmıyoruz

- ❌ Video kursu üretmiyoruz (Udemy değiliz)
- ❌ Canlı eğitim yapmıyoruz (BTKakademi değiliz)
- ❌ Mentorluk satmıyoruz (köprü değiliz)
- ❌ İş ilanı sayfası değiliz

Sadece **kişisel öğrenme aracı** — hızlı, oyunlaştırılmış, ölçülebilir.
