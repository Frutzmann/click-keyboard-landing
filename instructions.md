# Bonnes pratiques : Animation Scroll par séquence d'images

## 1. Préparation des assets

### Extraction des frames

- Utiliser `ffmpeg` pour extraire les frames depuis le MP4 source
- Viser entre **60 et 150 frames** selon la durée de scroll souhaitée
- Règle : environ **10-15 frames par seconde de scroll estimée**
- Exporter en résolution **1440px ou 1920px** de large maximum (pas de 4K inutile)
- Nommer les fichiers avec un padding numérique : `frame_0001.webp`, `frame_0002.webp`...

### Format et compression

- Privilégier **WebP** (quality 80-85) comme format principal
- Prévoir un **fallback JPEG** pour les navigateurs legacy si nécessaire
- AVIF offre un meilleur ratio poids/qualité mais le support navigateur est encore inégal
- Objectif : **50-150 Ko par frame** selon la complexité visuelle
- Budget total cible : **5-15 Mo** pour l'ensemble de la séquence

### Organisation des fichiers

- Stocker toutes les frames dans un dossier dédié : `/assets/frames/`
- Servir les images depuis un **CDN** avec cache long (immutable)
- Activer **HTTP/2** sur le serveur pour paralléliser les requêtes

---

## 2. Preloading

### Stratégie de chargement progressif

- Charger d'abord la **frame 1** (affichage immédiat au-dessus de la ligne de flottaison)
- Puis charger un **squelette** de frames espacées (1, 10, 20, 30...) pour couvrir tout le range
- Ensuite charger les frames restantes en arrière-plan
- Ne rendre la section scrollable **interactive qu'une fois le chargement suffisant** (>80%)

### Indicateur de progression

- Afficher un indicateur de chargement tant que les frames ne sont pas prêtes
- Verrouiller le scroll sur la section animée ou afficher un placeholder statique en attendant
- Transition douce entre le placeholder et l'animation une fois chargée

### Gestion mémoire

- Stocker les frames dans un **tableau d'objets `Image` pré-décodés**
- Utiliser `image.decode()` (promesse) pour décoder en amont sans bloquer le thread principal
- Sur mobile, envisager de réduire le nombre de frames (÷2) pour limiter l'empreinte mémoire

---

## 3. Rendu Canvas

### Setup

- Utiliser un élément `<canvas>` dimensionné à **100% du viewport**
- Le canvas doit être en `position: sticky` ou `position: fixed` dans un conteneur de scroll
- Respecter le **devicePixelRatio** pour un rendu net sur écrans Retina
- Redimensionner le canvas dynamiquement au `resize` (avec debounce)

### Dessin des frames

- Utiliser `context.drawImage(image, x, y, width, height)` pour dessiner chaque frame
- Centrer l'image dans le canvas avec un calcul de **cover** (comme `background-size: cover`)
- Ne redessiner que si la **frame a changé** par rapport au dernier rendu (éviter les draws inutiles)

### Performance de rendu

- Utiliser `requestAnimationFrame` pour synchroniser les draws avec le cycle de rendu
- Ne jamais dessiner directement dans le handler de scroll — toujours passer par rAF
- Éviter de créer des objets dans la boucle de rendu (pré-allouer tout)

---

## 4. Mapping Scroll → Frame

### Calcul de la progression

- Définir un **conteneur de scroll** dont la hauteur dépasse le viewport (ex : `height: 300vh`)
- Calculer la progression de 0 à 1 selon la position de scroll dans ce conteneur
- Mapper cette progression à l'index de frame : `frameIndex = Math.floor(progress * (totalFrames - 1))`

### Hauteur du conteneur

- Plus le conteneur est haut, plus le scroll sera **lent et contrôlé**
- Ratio recommandé : **200vh à 500vh** selon le nombre de frames et l'effet souhaité
- Règle empirique : environ **3-5vh par frame** pour une sensation fluide

### Lissage

- Appliquer un **lerp (interpolation linéaire)** entre la frame actuelle et la frame cible
- Cela adoucit les transitions et évite les sauts brusques lors d'un scroll rapide
- Facteur de lerp recommandé : **0.1 à 0.2** (plus bas = plus doux, plus haut = plus réactif)
- Alternative : utiliser `IntersectionObserver` combiné avec `scroll` pour détecter l'entrée/sortie de la zone d'animation

---

## 5. Responsive et mobile

### Adaptation mobile

- Réduire le nombre de frames sur mobile (**60-80 frames** max)
- Servir des frames en résolution plus basse (**960-1280px**)
- Utiliser `<picture>` ou un chargement conditionnel basé sur `window.innerWidth`
- Tester sur des appareils réels — les émulateurs ne reflètent pas la mémoire disponible

### Touch et inertie

- Le scroll tactile a de l'**inertie** (momentum scroll) — le lerp aide à lisser cet effet
- Tester avec `-webkit-overflow-scrolling: touch` activé et désactivé
- S'assurer que le conteneur de scroll n'entre pas en conflit avec le **pull-to-refresh** natif

### Orientation

- Gérer le changement d'orientation (portrait ↔ paysage) avec un recalcul du canvas
- Prévoir éventuellement deux jeux de frames (paysage / portrait) pour les visuels critiques

---

## 6. UX et intégration dans la page

### Entrée et sortie de l'animation

- L'animation ne doit pas démarrer brutalement — prévoir une **zone tampon** avant/après
- Figer la première frame en haut et la dernière frame en bas pendant le scroll hors zone
- Utiliser des transitions CSS sur l'opacité pour fondre l'entrée/sortie

### Contenu superposé

- Superposer du texte, des titres ou des CTA **par-dessus le canvas** à des moments clés du scroll
- Synchroniser l'apparition du texte avec des seuils de progression (ex : texte A à 20%, texte B à 60%)
- Animer le texte avec des transitions CSS classiques (fade, slide) déclenchées par la progression

### Accessibilité

- Fournir un **fallback statique** si `prefers-reduced-motion` est activé
- Ajouter un `aria-label` descriptif sur la section
- Le contenu textuel superposé doit rester dans le DOM (pas uniquement dans le canvas)
- Prévoir un mode dégradé pour les connexions lentes (image statique + texte)

---

## 7. Performance globale

### Métriques cibles

- **LCP (Largest Contentful Paint)** : la première frame doit se charger en < 2.5s
- **CLS (Cumulative Layout Shift)** : le canvas doit avoir des dimensions réservées dès le départ
- **INP (Interaction to Next Paint)** : le handler de scroll ne doit jamais dépasser 50ms

### Optimisations serveur

- Headers de cache : `Cache-Control: public, max-age=31536000, immutable`
- Compression **Brotli** activée sur le serveur
- Preconnect au CDN dans le `<head>` : `<link rel="preconnect" href="...">`
- Précharger la première frame : `<link rel="preload" as="image" href="frame_0001.webp">`

### Monitoring

- Mesurer le temps de chargement complet de la séquence en conditions réelles
- Tester sur **3G throttlée** pour valider l'expérience dégradée
- Profiler la consommation mémoire sur mobile (Safari iOS est le plus contraignant)
