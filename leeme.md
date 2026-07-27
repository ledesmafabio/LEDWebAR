# Realidad Aumentada Web

Dos páginas y una carpeta de datos. Sin backend.

```
/  (raíz de GitHub Pages)
├── index.html          editor: componer, guardar, generar el QR
├── viewer.html         visor: cámara, seguimiento y capas
└── experiencias/
    ├── .gitkeep
    └── ar-xxxxxxxx.json    una por experiencia publicada
```

El editor **deduce solo** la dirección del visor a partir de dónde está abierto: `location.origin + carpeta + 'viewer.html?id=' + id`. No hay ningún campo de URL para completar. Si movés el repo o cambiás de dominio, los QR nuevos se ajustan solos.

---

## El ciclo de trabajo

1. **Componer.** Cargás capas (imágenes, GIFs, videos cortos) y las ubicás sobre la superficie del marcador: posición, tamaño, rotación, opacidad y orden.
2. **Elegir marcador.**
   - *QR como marcador* (por defecto): el editor arma una ficha imprimible con el QR adentro, marco negro y marcas de esquina. Ese papel hace las dos cosas: se escanea para abrir la experiencia y se enfoca para anclar el contenido.
   - *Imagen propia*: cualquier foto, afiche o ilustración con suficiente detalle.
3. **Guardar.** El marcador se compila con MindAR y la experiencia queda en **IndexedDB de este navegador**, con un `id` propio. El QR ya funciona en esta máquina: sirve para probar sin subir nada.
4. **Publicar.** Botón **Repo** → descarga `<id>.json` → lo ponés en `experiencias/` del repositorio → commit.

El visor resuelve `?id=` en este orden:

1. `experiencias/<id>.json` — la versión publicada.
2. IndexedDB de este navegador — el borrador local.

Por eso **el enlace del QR no cambia** entre probar y publicar. El mismo papel impreso sirve antes y después del commit. En la biblioteca del editor, cada tarjeta muestra un sello *en el repo* cuando el archivo ya está publicado.

---

## Qué estaba fallando en la versión anterior

**La cámara.** Tres causas encadenadas:

- `getUserMedia()` se llamaba desde `window.onload`, fuera de cualquier toque del usuario. iOS Safari solo entrega la cámara si el pedido nace de un gesto. Ahora el permiso se pide dentro del click en *Activar cámara*, con una pantalla previa que explica qué va a pasar.
- El `<a-scene arjs="sourceType: webcam">` estaba en el HTML estático, así que AR.js abría la cámara al cargar la página, al mismo tiempo que el `getUserMedia` manual. Dos pedidos compitiendo por el mismo dispositivo. Ahora la escena se arma con `autoStart: false` y la cámara se abre una sola vez, en `start()`.
- `aframe-ar.js` se cargaba desde `raw.githack.com/.../master/...`: sin versión fija y con límite de tasa. Cuando githack falla, `AFRAME` existe pero el componente `arjs` no, y la escena queda muda sin dar error. Ahora todo viene de jsDelivr con versión clavada.

**La responsividad del visor.** El CSS forzaba `a-scene { width:100vw !important; height:100vh !important }` y, sobre todo, `video { object-fit: cover !important }`. Esa última regla también alcanzaba al video de la cámara, y AR.js calcula por su cuenta la escala y el recorte del cuadro: al pisárselo, el seguimiento queda desalineado respecto de lo que se ve. Además `100vh` en iOS incluye la barra de Safari, así que la interfaz se cortaba por abajo. Ahora la escena se posiciona con `position:fixed; inset:0`, sin tocar el dimensionado del video, y los márgenes usan `env(safe-area-inset-*)`.

**El alcance del guardado.** `localStorage` es por navegador: el QR solo funcionaba en la máquina que lo creó — tu propio mensaje de error lo decía. La carpeta `experiencias/` resuelve eso sin agregar servidor. De paso, IndexedDB reemplaza a `localStorage` para los borradores: el límite de 5 MB se agotaba enseguida con un video en data URL.

**El marcador.** La versión anterior guardaba el *target* personalizado pero el visor siempre usaba el marcador Hiro. Ahora el marcador se compila de verdad, y la ficha con el QR es un objetivo de seguimiento válido.

---

## Publicar en GitHub Pages

1. Subí `index.html`, `viewer.html` y la carpeta `experiencias/` a la raíz del repo.
2. *Settings → Pages → Deploy from branch: main / root*.
3. Abrí `https://usuario.github.io/repo/` — el editor ya sabe su propia dirección.

Si abrís el editor con doble clic (`file://`) te avisa que el QR no serviría. En `localhost` también avisa que el enlace solo va a funcionar en esa máquina.

---

## Peso de las experiencias

Los archivos cargados desde el disco quedan embebidos como data URL. Un video de 20 MB se convierte en unos 27 MB de JSON que el celular descarga entero antes de empezar.

Para video conviene: subir el `.mp4` a una carpeta `medios/` del repo y agregarlo en el editor con el botón **URL**, escribiendo la ruta relativa (`medios/dragon.mp4`). El JSON queda en pocos KB. El indicador **PESO** en el editor muestra cuánto va a pesar la experiencia.

---

## Formatos

| Tipo | Recomendado | Notas |
|---|---|---|
| Imagen fija | PNG con transparencia, WebP | Se respeta el canal alfa |
| Imagen animada | GIF | Se copia cuadro a cuadro a un canvas para animarse en WebGL; conviene que sea chico |
| Video | MP4 (H.264) o WebM | Silenciado por defecto: iOS no reproduce solo un video con audio |

Para animación pesada, un MP4 corto rinde mucho mejor que un GIF: menos peso, más cuadros por segundo y decodificación por hardware.

---

## Qué hace un buen marcador

El seguimiento es por características naturales: busca esquinas y contrastes locales, no un código.

**Funciona bien:** collage denso, ilustración con detalle, textura irregular, texto y formas mezclados, marco oscuro cerrado.
**Funciona mal:** degradados suaves, fotos desenfocadas, patrones repetidos (damero, trama regular), zonas grandes de un solo color, simetría perfecta.

La ficha que genera el editor ya resuelve esto: marco grueso, título, QR y cuatro marcas de esquina distintas entre sí, que además le dan orientación inequívoca.

---

## Permisos de cámara

El visor pide el permiso explícitamente, dentro del toque en *Activar cámara*. Si se deniega, distingue el tipo de error y da los pasos concretos:

- **iOS / Safari:** Ajustes → Safari → Cámara → *Preguntar*. O el botón `aA` en la barra de direcciones → Ajustes del sitio web → Cámara.
- **Android / Chrome:** candado junto a la dirección → Permisos → Cámara → Permitir.

Los navegadores dentro de otras apps (Instagram, Facebook, TikTok) bloquean la cámara. El visor lo detecta y avisa que hay que abrir el enlace en Safari o Chrome.

---

## Problemas frecuentes

**El QR abre el visor y dice que la experiencia no está publicada.** Falta subir `<id>.json` a `experiencias/`, o GitHub Pages todavía no terminó de publicar (suele tardar un minuto).

**Falta HTTPS.** Estás en `http://` o `file://`. Sin certificado no hay cámara.

**El marcador no se detecta.** Poca luz, marcador cortado por el borde del encuadre, reflejo sobre papel plastificado, o impresión muy chica. Una ficha de 10 cm de lado se sigue desde unos 40 cm.

**El contenido tiembla.** Normal si el marcador está lejos. En `viewer.html` se puede subir `filterMinCF` (más estable, más latencia) o bajarlo (más reactivo).

**El video no arranca.** Que sea H.264 en `.mp4`; algunos códecs no los abre Safari. Si le sacaste el silencio, iOS lo bloquea.

**Compilar tarda mucho.** Entre 10 y 60 segundos según la imagen. Se calculan descriptores en varias escalas. Imágenes de más de 2000 px no mejoran el resultado y multiplican el tiempo.

---

## Dependencias

Se cargan desde CDN con versión fija:

- [A-Frame 1.4.2](https://aframe.io) — escena 3D
- [MindAR 1.2.5](https://hiukim.github.io/mind-ar-js-doc/) — seguimiento de imágenes y compilador
- [three.js 0.147](https://threejs.org) — motor de render
- [qrcode 1.5.3](https://github.com/soldair/node-qrcode) — generación del QR

Para que funcione sin internet, descargá esos archivos a una carpeta `lib/` y cambiá los `src` por rutas relativas. El seguimiento en sí ya corre entero en el dispositivo.

---

## Estructura de `experiencias/<id>.json`

```json
{
  "version": 2,
  "id": "ar-m8x2k91a",
  "titulo": "Anatomía de un personaje",
  "descripcion": "Apuntá la cámara a la ficha",
  "marcador": {
    "modo": "qr",
    "relacion": 0.75,
    "mind": "<base64 del objetivo compilado>",
    "vistaPrevia": "data:image/jpeg;base64,..."
  },
  "capas": [
    {
      "tipo": "video",
      "src": "medios/dragon.mp4",
      "x": 0, "y": 0.2, "ancho": 0.6,
      "rotacion": 0, "opacidad": 1,
      "visible": true, "bucle": true, "silenciado": true
    }
  ]
}
```

Las coordenadas están en **unidades de marcador**: el ancho del marcador vale `1`. Una capa con `ancho: 0.6` ocupa el 60 % del ancho del marcador; `x: 0, y: 0` es el centro.

Se puede editar a mano y volver a importar con **Importar** en el editor.
