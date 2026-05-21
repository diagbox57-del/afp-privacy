# AFP Garage — Kit promo vidéo

Tout est prêt pour produire une vidéo promo cinématique de 34 secondes.

## Fichiers

| Fichier | Usage |
|---------|-------|
| `promo.html` | Promo **format horizontal 16:9** (desktop, YouTube, site web) |
| `promo-vertical.html` | Promo **format vertical 9:16** (TikTok, Instagram Reels, Stories, Shorts) |
| `voix-off.md` | Script de voix off complet : 15 cues timés, ton de voix, conseils micro |
| `subtitles.vtt` | Sous-titres standard WebVTT (compatible YouTube, Vimeo, lecteurs HTML5) |

## Pour produire la vidéo

### 1. Génère la voix off
Deux options :
- **Toi-même** au micro (voir conseils dans `voix-off.md`)
- **IA TTS** : colle le bloc de `voix-off.md` dans ElevenLabs (modèle `eleven_multilingual_v2`, voix française « Antoine » ou « Charles »), exporte en MP3

### 2. Enregistre l'écran
- Ouvre `promo.html?clean=1` (ou `promo-vertical.html?clean=1`) → tous les contrôles disparaissent
- Plein écran : touche `F`
- Lance ton screen recorder (OBS, Cmd+Shift+5 sur Mac, Game Bar sur Windows)
- Clique « Lancer » (ou recharge la page : auto-lecture)
- Stop après 34s

### 3. Monte la vidéo
Dans CapCut / DaVinci Resolve / Premiere :
- Importe la capture d'écran (piste vidéo)
- Importe la voix off (piste audio)
- Optionnel : musique de fond cinématique à −18 dB
- Optionnel : importe `subtitles.vtt` si tu veux re-générer des sous-titres incrustés (la promo en a déjà mais le VTT permet de les remplacer dans le monteur)

### 4. Exporte
- **YouTube / web** : 1080p 30fps H.264 depuis `promo.html`
- **TikTok / Reels / Stories** : 1080×1920 30fps depuis `promo-vertical.html`

## Modes URL

Les deux fichiers HTML acceptent le paramètre `?clean=1` qui masque entièrement les contrôles et les indications clavier — idéal pour un screen recording sans pollution visuelle.

```
promo.html?clean=1
promo-vertical.html?clean=1
```

## Raccourcis clavier (mode normal)

- `F` — Plein écran
- `S` — Activer/désactiver les sous-titres
- `Espace` — Lancer la promo
