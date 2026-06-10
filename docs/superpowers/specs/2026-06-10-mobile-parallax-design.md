# Mobile Parallax Interface – Design Spec

## Obiettivo
Rendere l'esperienza Poster Skate funzionale e coinvolgente su smartphone: poster proporzionato e centrato, parallax al tocco + giroscopio, effetti glitch e microfono invariati.

---

## Layout Mobile

- Il poster conserva le proporzioni 500:650.
- Su schermi con `max-width: 600px`:
  - `poster-container` diventa responsive: `width: min(95vw, 400px)`, `height: auto` con `aspect-ratio: 500/650`.
  - `body` rimane centrato (`flex` + `justify-content: center` + `align-items: center`).
  - `mic-indicator` si sposta sopra il poster (`bottom: auto; top: 10px`) per non coprire il touch.
- Su desktop (`min-width: 601px`) il layout resta esattamente come oggi.

---

## Parallax – Approccio Ibrido Touch + Gyroscope

### Giroscopio (base)
- Evento: `deviceorientation`.
- Parametri usati: `beta` (inclinazione front/back, ±45° → [-1, 1]) e `gamma` (inclinazione left/right, ±45° → [-1, 1]).
- I valori normalizzati sostituiscono `mouseX` e `mouseY` come parallax base.
- Smoothing: media mobile esponenziale (factor 0.1) per evitare tremolio.
- Fallback automatico: se `DeviceOrientationEvent` non è disponibile o il permesso è negato, il parallax base passa a zero e si attiva solo il touch.

### Touch (boost)
- Eventi: `touchstart`, `touchmove`, `touchend` sul poster.
- Calcola `touchX` e `touchY` come posizione relativa al centro del poster, normalizzati in [-1, 1].
- **Somma vettoriale**: il parallax totale = giroscopio + touch.
- Al `touchend`, il contributo touch decade a zero in 300 ms (ease-out).
- Se il giroscopio non è disponibile, il touch diventa l'unico input (parallax puro al tocco, identico al mouse desktop).

### Mouse (desktop)
- Invariato: `mousemove` e `mouseleave` come oggi.

---

## Glitch e Audio

- Il canvas glitch, il microfono e gli overlay (noise, scanlines, vignette, chromatic) restano **inalterati**.
- Il click per attivare il microfono su mobile è più semplice: un `touchstart` sul poster avvia `initAudio()` se non ancora attivo (lo stesso listener del mouse).

---

## CSS Responsive Breakpoint

```css
@media (max-width: 600px) {
  .poster-container {
    width: min(95vw, 400px);
    height: auto;
    aspect-ratio: 500 / 650;
  }
  .mic-indicator {
    bottom: auto;
    top: 10px;
  }
}
```

---

## JS – Modifiche al loop di animazione

1. **Variabili di input unificate:**
   - `inputX`, `inputY` (somma di giroscopio + touch + mouse).
   - Ogni frame: `inputX = gyroX + touchX + mouseX`, `inputY = gyroY + touchY + mouseY`.
   - Il loop usa `inputX/inputY` al posto di `mouseX/mouseY` per calcolare le trasformazioni.

2. **Gyroscope listener:**
   ```js
   window.addEventListener('deviceorientation', (e) => {
     const normBeta  = Math.max(-1, Math.min(1, e.beta  / 45));
     const normGamma = Math.max(-1, Math.min(1, e.gamma / 45));
     gyroX = gyroX * 0.9 + normGamma * 0.1;
     gyroY = gyroY * 0.9 + normBeta  * 0.1;
   });
   ```

3. **Touch listener:**
   ```js
   poster.addEventListener('touchstart', updateTouch);
   poster.addEventListener('touchmove',  updateTouch);
   poster.addEventListener('touchend',   () => { touchDecay = 1; });
   ```
   `updateTouch` calcola `touchX/touchY` dal primo `touches[0]`.  
   Se `touchDecay > 0`: `touchX *= (1 - touchDecay)`, `touchY *= (1 - touchDecay)`, `touchDecay -= 0.05` per frame.

---

## File coinvolti

- `index.html` – unico file da modificare (CSS responsive + JS touch/gyro).

---

## Self-Review Checklist

- [x] No placeholder o TBD.  
- [x] Nessuna contraddizione tra layout, parallax e glitch.  
- [x] Fallback gyro → touch chiaro.  
- [x] Desktop invariato.  
- [x] Microfono e glitch non toccati.
