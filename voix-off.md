# Voix off — AFP Garage Promo (34 s)

Script de voix off synchronisé avec `promo.html`.
Langue : **français**. Voix recommandée : **masculine, grave, posée — registre "cinéma allemand / pub auto premium"**. Débit : modéré, articulé.

## Conseils d'enregistrement

- **Mic** : USB cardioïde correct (Yeti, AT2020, SM7B…) ou smartphone récent en pièce calme.
- **Pop-filter** obligatoire sur les "P" et "T".
- **Distance** : ~15 cm du micro.
- **Post-prod** : compresseur léger (ratio 3:1), EQ low-cut 80 Hz, normalisation à −3 dBFS.
- **Outils IA si tu ne veux pas enregistrer toi-même** : ElevenLabs (voix française "Antoine" / "Charles"), PlayHT, Murf, ou la voix Google WaveNet `fr-FR-Wavenet-D`.

---

## Timeline complète

| Temps | Scène | Voix off | Sous-titre (auto) |
|------:|-------|----------|-------------------|
| 00:00.6 → 00:02.4 | Logo | **« AFP Garage. »** *(ton posé, marqué)* | AFP Garage |
| 00:02.6 → 00:05.0 | Logo | **« Le carnet d'entretien signé Auto France Performance. »** | Le carnet d'entretien signé Auto France Performance |
| 00:05.5 → 00:07.5 | Problème | **« La dernière vidange… c'était quand ? »** *(ton complice, légère hésitation)* | La dernière vidange… c'était quand ? |
| 00:07.7 → 00:09.5 | Problème | **« Le contrôle technique expire bientôt ? »** | Le contrôle technique expire bientôt ? |
| 00:09.7 → 00:11.0 | Problème | **« Combien j'ai dépensé cette année ? »** | Combien j'ai dépensé cette année ? |
| 00:11.5 → 00:14.3 | Solution | **« Tout votre garage… dans votre poche. »** *(ton qui s'éclaire, montée d'énergie)* | Tout votre garage… dans votre poche. |
| 00:14.6 → 00:18.0 | Solution | **« Chaque véhicule, chaque entretien, chaque rappel — au même endroit. »** | Chaque véhicule, chaque entretien, chaque rappel — au même endroit. |
| 00:18.5 → 00:20.2 | Features | **« Carnet d'entretien complet. »** *(rythme staccato, énergique)* | Carnet d'entretien complet. |
| 00:20.4 → 00:21.8 | Features | **« Rappels automatiques. »** | Rappels automatiques. |
| 00:22.0 → 00:23.4 | Features | **« OCR sur vos factures. »** | OCR sur vos factures. |
| 00:23.6 → 00:25.0 | Features | **« Multi-véhicules illimité. »** | Multi-véhicules illimité. |
| 00:25.5 → 00:27.7 | RGPD | **« Vos données restent en Europe. »** *(ton rassurant, calme)* | Vos données restent en Europe. |
| 00:27.9 → 00:30.0 | RGPD | **« Pas de pub. Pas d'analytics. Cent pour cent RGPD. »** | Pas de pub. Pas d'analytics. 100% RGPD. |
| 00:30.5 → 00:32.0 | CTA | **« Reprenez le contrôle de votre garage. »** *(ton motivant, conclusif)* | Reprenez le contrôle de votre garage. |
| 00:32.2 → 00:34.0 | CTA | **« Disponible sur App Store et Google Play. »** | App Store · Google Play |

---

## Script en bloc (pour copier-coller dans un TTS)

```
AFP Garage.
Le carnet d'entretien signé Auto France Performance.

La dernière vidange... c'était quand ?
Le contrôle technique expire bientôt ?
Combien j'ai dépensé cette année ?

Tout votre garage... dans votre poche.
Chaque véhicule, chaque entretien, chaque rappel — au même endroit.

Carnet d'entretien complet.
Rappels automatiques.
OCR sur vos factures.
Multi-véhicules illimité.

Vos données restent en Europe.
Pas de pub. Pas d'analytics. Cent pour cent RGPD.

Reprenez le contrôle de votre garage.
Disponible sur App Store et Google Play.
```

---

## Workflow complet de production vidéo

1. **Génère la voix off** :
   - Soit tu l'enregistres toi-même au micro,
   - Soit tu colles le script bloc dans **ElevenLabs** (modèle `eleven_multilingual_v2`, voix "Antoine" ou "Charles") → exporte en MP3/WAV.
2. **Lance** `promo.html` en plein écran (touche `F`).
3. **Démarre l'enregistrement écran** (OBS, QuickTime, Cmd+Shift+5…).
4. **Clique « Lancer »** dans la promo + lance simultanément ta voix off dans un autre player ou directement dans OBS comme source audio.
5. **Stop** après 34 secondes.
6. **Synchronise** dans CapCut / DaVinci Resolve / Premiere :
   - Vidéo écran enregistrée
   - Piste audio voix off
   - (optionnel) musique de fond cinématique — chercher "cinematic corporate" sur Artlist / Epidemic Sound, à −18 dB sous la voix.
7. **Export** : H.264 1080p 30fps pour YouTube/Instagram, ou 1080×1920 9:16 pour Reels/TikTok.

---

## Variante musique (suggestions libres de droits)

- "Inspiring Cinematic" (Bensound, gratuit avec attribution)
- "Tech Corporate" (Pixabay Music, gratuit)
- Pour un rendu auto-premium : chercher des tracks "Audi commercial style" / "Mercedes ad music" sur Epidemic Sound.

Voilà — tu as tout pour produire une vidéo prête à diffuser.
