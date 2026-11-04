# Simulación de agua en 3D con Three.js

Este proyecto crea una superficie de agua animada en 3D usando `Three.js` y shaders de GPU. La idea principal es representar una malla plana cuyo eje `z` (altura) cambia con el tiempo para simular ondas. La superficie se ilumina para que parezca agua brillante y dinámica.

El archivo principal es `index.html` y todo el comportamiento está dentro de ese mismo documento. A continuación se explica, de manera clara, qué hace cada bloque del código.

---

## 1) Estructura base del HTML

```html
<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Cálculo III — Simulación matemática de agua</title>
```

- `<!doctype html>` indica que el documento es HTML5.
- `<html lang="es">` define el idioma del documento como español.
- `<meta charset="utf-8">` hace que el navegador lea adecuadamente caracteres como tildes y ñ.
- `<meta name="viewport" ...>` asegura que la página responda bien a pantallas móviles y de escritorio.
- `<title>` coloca el nombre que aparece en la pestaña del navegador.

---

## 2) Estilos CSS

```css
<style>
  * { box-sizing: border-box; }
  html, body { margin: 0; height: 100%; overflow: hidden; background:#071522; font-family: Arial, sans-serif; color:#eef7ff; }
  #scene { width:100%; height:100%; display:block; }
```

- `* { box-sizing: border-box; }` hace que todos los elementos respeten bien sus tamaños.
- `html, body` elimina los márgenes y hace que la página tenga altura completa.
- `overflow: hidden` evita que aparezcan barras de desplazamiento.
- `background:#071522` pone un fondo oscuro azul-gris.
- `#scene` es el contenedor principal donde se renderiza la escena 3D.

Luego sigue el estilo del panel lateral:

```css
.panel { position:absolute; top:16px; left:16px; z-index:2; background:rgba(3,18,34,.9); border:1px solid #35536d; border-radius:12px; padding:18px; width:min(350px, calc(100vw - 32px)); box-shadow:0 15px 35px #0005; }
```

- `position:absolute` hace que el panel “flote” sobre la escena.
- `z-index:2` lo pone por encima de la escena 3D.
- `background:rgba(...)` lo hace semi-transparente.
- `border-radius` redondea los bordes.
- `box-shadow` da una sombra elegante.

También se estilizan los textos, sliders, botones y mensajes de estado:

```css
h1 { font-size:17px; margin:0 0 7px; }
p { font-size:13px; line-height:1.45; color:#d0e2ed; margin:8px 0; }
label { display:block; font-size:12px; margin:13px 0 6px; }
input[type=range] { width:100%; accent-color:#45bfe6; }
button { border:1px solid #33708a; border-radius:7px; background:#123d55; color:#fff; padding:9px 12px; cursor:pointer; }
```

- Esto define el aspecto visual del panel de control.
- El `range` se ve como un slider con color azul.
- Los botones tienen estilo moderno y oscuro.

Finalmente, hay estilos para el footer y el mensaje de error:

```css
.footer { position:absolute; bottom:12px; right:16px; z-index:2; font-size:12px; color:#d1dfe8; background:#061625a6; padding:7px; border-radius:5px; }
.status { color:#80eeb6; font-size:12px; }
#error { display:none; color:#ffaaaa; background:#401921; border:1px solid #a44; margin-top:12px; padding:10px; border-radius:6px; font-size:12px; }
```

- El `footer` informa cómo interactuar con la cámara.
- `status` cambia el color del mensaje de estado para que se vea activo.
- `#error` es un contenedor que se puede mostrar si ocurre un problema.

---

## 3) Importación de Three.js

```html
  <script type="importmap">
    {
      "imports": {
        "three": "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js",
        "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/"
      }
    }
  </script>
</head>
<body>
```

- `importmap` sirve para configurar rutas de importación en JavaScript.
- Esto permite escribir:

```js
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
```

sin necesidad de instalar dependencias localmente.
- `three` se obtiene desde un CDN (content delivery network).

---

## 4) Cuerpo HTML del panel interactivo

```html
<body>
  <div id="scene" aria-label="Simulación 3D animada de ondas de agua"></div>
  <section class="panel">
    <h1>Proyecto de Cálculo III</h1>
    <p><strong>Simulación de superficie de agua</strong> mediante ondas sinusoidales, derivadas parciales, gradiente, normal e iluminación.</p>
    <p><strong>Modelo:</strong> h(x,y,t) = Σ Aᵢ sen(kᵢ(dᵢ·r) − ωᵢt + φᵢ)</p>
    <p><strong>Derivadas:</strong> hₓ = ∂h/∂x, hᵧ = ∂h/∂y</p>
    <p><strong>Normal:</strong> N = normalizar(−hₓ, −hᵧ, 1)</p>
```

- `#scene` es el área donde la escena 3D se dibujará.
- El `<section class="panel">` contiene todos los controles del usuario.
- Los textos explican el concepto matemático del proyecto.
- La ecuación en el `p` muestra la idea clave:
  - la altura de la superficie (`h`) depende del espacio (`x, y`) y del tiempo (`t`)
  - se construye como suma de varias ondas sinusoidales

Luego vienen los controles:

```html
    <label for="amplitude">Altura de las olas: <output id="amplitudeValue">1.00</output>×</label>
    <input id="amplitude" type="range" min="0" max="2" step="0.05" value="1">
    <label for="speed">Velocidad de la animación: <output id="speedValue">1.00</output>×</label>
    <input id="speed" type="range" min="0" max="3" step="0.1" value="1">
    <div class="buttons"><button id="toggle">Pausar</button><button id="reset">Restablecer</button></div>
    <p class="status" id="status">● Iniciando simulación…</p>
    <div id="error"></div>
  </section>
  <div class="footer">Arrastrar: girar · Rueda: zoom · Botón derecho: desplazar</div>
```

- `input type="range"` crea deslizadores que permiten modificar la amplitud y la velocidad.
- `output` muestra el valor actual de cada control.
- Los botones sirven para pausar y reiniciar la simulación.
- `#status` muestra mensajes de estado.
- `#error` es un contenedor oculto que puede usarse para mostrar errores.
- El `footer` explica cómo interactuar con la cámara.

---

## 5) Script principal: importar librerías

```html
  <script type="module">
    import * as THREE from 'three';
    import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
```

- `type="module"` permite usar módulos ES en el navegador.
- `THREE` contiene todas las herramientas principales de la librería 3D.
- `OrbitControls` permite mover la cámara con el mouse.

---

## 6) Crear la escena 3D

```js
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x071522);
    scene.fog = new THREE.Fog(0x071522, 13, 29);
```

- `new THREE.Scene()` crea el mundo 3D.
- Se define el fondo oscuro azul.
- `Fog` añade niebla para que la escena vea profundidad y tenga sensación de distancia.

```js
    const camera = new THREE.PerspectiveCamera(48, window.innerWidth / window.innerHeight, .1, 100);
    camera.up.set(0, 0, 1);
    camera.position.set(7.4, -9.4, 7.6);
```

- `PerspectiveCamera` crea una cámara realista con perspectiva.
- `camera.up.set(0, 0, 1)` cambia la orientación “vertical” del sistema para que el eje `Z` quede apuntando hacia arriba.
- `camera.position.set(...)` coloca la cámara en una posición inicial adecuada para ver la superficie.

```js
    const renderer = new THREE.WebGLRenderer({antialias:true});
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    document.getElementById('scene').appendChild(renderer.domElement);
```

- `WebGLRenderer` es el motor que dibuja la escena en el navegador.
- `antialias:true` suaviza bordes.
- `setPixelRatio` ajusta la resolución.
- `setSize` hace que el render ocupe toda la pantalla.
- `outputColorSpace` hace que los colores se vean correctamente en pantalla.
- El canvas resultante se inserta dentro del div `#scene`.

---

## 7) Controles de la cámara

```js
    const orbit = new OrbitControls(camera, renderer.domElement);
    orbit.target.set(0, 0, 0);
    orbit.enableDamping = true;
    orbit.maxPolarAngle = Math.PI * .49;
    orbit.minDistance = 4;
    orbit.maxDistance = 24;
    orbit.update();
```

- `OrbitControls` hace que la cámara se pueda girar y mover con el mouse.
- `target` es el punto hacia el cual vemos.
- `enableDamping` da un efecto suave al mover la cámara.
- `maxPolarAngle` limita la inclinación de la vista.
- `minDistance` y `maxDistance` limitan el zoom.
- `orbit.update()` actualiza la orientación actual antes de renderizar.

---

## 8) Crear la geometría de la superficie del agua

```js
    const geometry = new THREE.PlaneGeometry(12, 12, 220, 220);
```

- `PlaneGeometry` crea un plano.
- Las dimensiones 12x12 hacen la superficie grande.
- `220, 220` indican la cantidad de subdivisiones.
- Cuantos más vértices tenga la malla, más suave quedará la onda.

Esto es importante porque la onda se simula desplazando cada vértice del plano en el eje `z`.

---

## 9) Vertex shader: calcular la altura del agua

```js
    const vertexShader = `
      uniform float uTime;
      uniform float uAmplitude;
      varying vec3 vNormal;
      varying vec3 vWorldPosition;
      varying float vHeight;
      void main() {
        vec2 r = position.xy;
```

- `uniform` son variables que se pasan desde JavaScript al shader.
- `uTime` guarda el tiempo de la simulación.
- `uAmplitude` guarda la intensidad de la onda.
- `varying` son valores que se pasan del vertex shader al fragment shader.
- `position.xy` es la posición del vértice en el plano XY.

### Direcciones de onda

```js
        vec2 d1 = normalize(vec2(1.0, 0.25));
        vec2 d2 = normalize(vec2(-0.35, 1.0));
        vec2 d3 = normalize(vec2(0.75, -0.65));
```

Estas tres direcciones indican cómo se propagan las ondas en diferentes ángulos. En otras palabras, cada onda se mueve en una dirección distinta, creando una superficie más interesante que una sola ola uniforme.

### Amplitudes

```js
        float a1 = 0.22 * uAmplitude, a2 = 0.13 * uAmplitude, a3 = 0.08 * uAmplitude;
        float k1 = 1.35, k2 = 2.10, k3 = 2.80;
```

- `a1`, `a2`, `a3` son las amplitudes de cada onda.
- `k1`, `k2`, `k3` son las frecuencias de las ondas.
- Si aumentas `uAmplitude`, las olas crecen en altura.

### Funciones de tiempo

```js
        float t1 = k1 * dot(d1,r) - 1.10*uTime;
        float t2 = k2 * dot(d2,r) - 1.55*uTime + 1.2;
        float t3 = k3 * dot(d3,r) - 1.90*uTime + 2.1;
```

- `dot(d1, r)` calcula la proyección del punto sobre la dirección de la onda.
- Así, cada punto del plano tendrá una fase distinta dependiendo de su posición.
- La parte con `uTime` hace que la onda se mueva con el paso del tiempo.

### Fórmula de altura

```js
        float h = a1*sin(t1) + a2*sin(t2) + a3*sin(t3);
```

Esta es la parte central del algoritmo. La superficie del agua recibe la suma de tres ondas sinusoidales:

`h(x, y, t) = A1 sin(...) + A2 sin(...) + A3 sin(...)`

Eso produce una apariencia más realista que una sola onda simple.

### Derivadas parciales

```js
        float hx = a1*k1*d1.x*cos(t1) + a2*k2*d2.x*cos(t2) + a3*k3*d3.x*cos(t3);
        float hy = a1*k1*d1.y*cos(t1) + a2*k2*d2.y*cos(t2) + a3*k3*d3.y*cos(t3);
```

Aquí se calculan las derivadas parciales de la altura respecto a `x` y `y`:

- `hx` = cambio de altura en dirección x
- `hy` = cambio de altura en dirección y

Esto es exactamente lo que se usa en cálculo multivariable para describir la pendiente de la superficie.

### Normal de la superficie

```js
        vec3 n = normalize(vec3(-hx,-hy,1.0));
```

La normal es un vector perpendicular a la superficie. En geometría, la normal permite saber cómo está orientada cada parte del agua para calcular luz y sombras.

La fórmula tiene sentido porque la superficie es un “mapeo” de la forma:

`z = h(x, y, t)`

y la normal se deriva de su gradiente.

### Posición final y proyección 3D

```js
        vec4 worldP = modelMatrix * vec4(position.xy,h,1.0);
        vWorldPosition = worldP.xyz;
        vNormal = normalize(mat3(modelMatrix)*n);
        vHeight = h;
        gl_Position = projectionMatrix * viewMatrix * worldP;
      }
    `;
```

- Se crea la posición 3D final del vértice con `x, y, h`.
- `modelMatrix` transforma el vértice al espacio del mundo.
- `projectionMatrix * viewMatrix * worldP` convierte ese punto 3D al espacio de la cámara para dibujarlo.

Esto es lo que finalmente hace visible la onda en pantalla.

---

## 10) Fragment shader: color y luz

```js
    const fragmentShader = `
      uniform vec3 uLightDirection;
      uniform vec3 uCameraPosition;
      varying vec3 vNormal;
      varying vec3 vWorldPosition;
      varying float vHeight;
      void main() {
        vec3 N = normalize(vNormal);
        vec3 L = normalize(uLightDirection);
        vec3 V = normalize(uCameraPosition-vWorldPosition);
```

- `N` = normal de la superficie
- `L` = dirección de la luz
- `V` = dirección de la cámara

Con estos vectores se calcula qué tan iluminado está cada punto.

### Iluminación difusa

```js
        float diffuse = max(dot(N,L),0.0);
```

- `dot(N, L)` mide qué tanto la luz incide sobre la superficie.
- Si el valor es alto, más brillo.
- `max(..., 0.0)` evita valores negativos.

### Reflexión especular

```js
        vec3 R = reflect(-L,N);
        float specular = pow(max(dot(R,V),0.0),72.0);
```

- `reflect` calcula la dirección reflejada de la luz.
- `specular` genera el brillo “especular” o reflejo de la luz.
- El valor de `72.0` hace que el brillo sea muy concentrado, parecido al brillo en vidrio o agua.

### Color base del agua

```js
        vec3 base = mix(vec3(.015,.13,.25),vec3(.08,.49,.68),smoothstep(-.45,.45,vHeight));
```

- `mix` interpola entre dos colores según la altura de la onda.
- En zonas bajas, el agua es más oscuro.
- En zonas altas, se vuelve más claro y azulado.

### Color final

```js
        vec3 col = base*(.35+.86*diffuse)+vec3(.7,.88,1.0)*specular*.75;
        gl_FragColor = vec4(col,1.0);
      }
    `;
```

- El color final depende de:
  - altura del agua
  - intensidad de la luz difusa
  - brillo especular
- `gl_FragColor` es el color final que se pintará en cada píxel.

---

## 11) Crear el material y la malla

```js
    const material = new THREE.ShaderMaterial({
      vertexShader, fragmentShader,
      uniforms: {
        uTime:{value:0},
        uAmplitude:{value:1},
        uLightDirection:{value:new THREE.Vector3(-.5,-.4,1).normalize()},
        uCameraPosition:{value:camera.position}
      },
      side:THREE.DoubleSide
    });
    scene.add(new THREE.Mesh(geometry,material));
```

- `ShaderMaterial` usa los shaders escritos arriba.
- `uniforms` pasa variables como tiempo, amplitud, dirección de la luz y posición de la cámara.
- `side:THREE.DoubleSide` hace que ambas caras del plano puedan ser visibles.
- Se agrega a la escena una malla con la geometría y el material.

---

## 12) Variables de control de la animación

```js
    let running = true;
    let simulationTime = 0;
    const clock = new THREE.Clock();
    const speed = document.getElementById('speed');
    const amplitude = document.getElementById('amplitude');
    const toggle = document.getElementById('toggle');
```

- `running` indica si la simulación está activa o pausada.
- `simulationTime` lleva la cuenta del tiempo total.
- `clock` mide el tiempo que pasa entre frames.
- `speed` y `amplitude` son los elementos HTML del slider.
- `toggle` es el botón para pausar o reanudar.

---

## 13) Escuchar cambios de los sliders

```js
    speed.addEventListener('input', () => document.getElementById('speedValue').textContent = Number(speed.value).toFixed(2));
    amplitude.addEventListener('input', () => {
      material.uniforms.uAmplitude.value = Number(amplitude.value);
      document.getElementById('amplitudeValue').textContent = Number(amplitude.value).toFixed(2);
    });
```

- Cada vez que el usuario mueve un slider, se actualiza el valor visible.
- También se actualiza la amplitud del shader para que la onda cambie de tamaño.

---

## 14) Botón de pausar/reanudar

```js
    toggle.addEventListener('click', () => {
      running = !running;
      toggle.textContent = running ? 'Pausar' : 'Reanudar';
      document.getElementById('status').textContent = running ? '● Simulación activa' : '● Simulación pausada';
    });
```

- Cuando el usuario hace clic, cambia el estado de `running`.
- El texto del botón cambia para reflejar la acción disponible.
- El estado visible en pantalla también se actualiza.

---

## 15) Botón de reinicio

```js
    document.getElementById('reset').addEventListener('click', () => {
      simulationTime = 0; speed.value = 1; amplitude.value = 1;
      speed.dispatchEvent(new Event('input'));
      amplitude.dispatchEvent(new Event('input'));
      orbit.reset();
    });
```

- Reinicia el tiempo del sistema a cero.
- Vuelve los sliders a sus valores originales.
- Ejecuta manualmente el evento `input` para actualizar las etiquetas.
- `orbit.reset()` restablece la cámara a su estado inicial.

---

## 16) Función animación principal

```js
    function animate() {
      requestAnimationFrame(animate);
      const dt = Math.min(clock.getDelta(),.05);
      if (running) simulationTime += dt*Number(speed.value);
      material.uniforms.uTime.value = simulationTime;
      material.uniforms.uCameraPosition.value.copy(camera.position);
      orbit.update();
      renderer.render(scene,camera);
    }
```

Esto es el “corazón” del programa:

- `requestAnimationFrame(animate)` llama a la función cada frame.
- `clock.getDelta()` mide el tiempo real que pasó entre frames.
- Si la simulación está activa, se suma ese tiempo a `simulationTime` multiplicado por la velocidad.
- `material.uniforms.uTime.value = simulationTime` actualiza el tiempo de la onda para que se mueva.
- `material.uniforms.uCameraPosition.value.copy(camera.position)` lleva la posición de la cámara al shader.
- `orbit.update()` actualiza los movimientos de la cámara.
- `renderer.render(scene, camera)` dibuja la escena final.

---

## 17) Estado final y arranque

```js
    document.getElementById('status').textContent = '● Simulación activa';
    animate();
    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth/window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth,window.innerHeight);
    });
```

- Se indica que la simulación está activa.
- Se inicia el bucle de animación con `animate()`.
- `resize` ajusta la cámara y el render al cambiar el tamaño de la ventana.

---

## 18) Qué está ocurriendo matemáticamente en resumen

La superficie del agua se representa como una función:

`h(x, y, t)`

donde:

- `x` y `y` son las coordenadas del punto en la malla
- `t` es el tiempo
- `h` es la altura del punto en el eje `z`

La ecuación se construye como suma de ondas sinusoidales:

`h = A1 sin(k1(d1·r) − ω1 t + φ1) + A2 sin(...) + A3 sin(...)`

Esto produce una superficie deformada con movimiento dinámico.

Luego, en el shader, se calcula la derivada parcial para obtener la pendiente y la normal de la superficie. Con esa información, se calcula la iluminación para dar la apariencia de agua brillante.

---

## 19) En pocas palabras

Este código hace esto:

1. Crea un plano grande 3D
2. Modifica la altura de cada punto según una suma de ondas sinusoidales
3. Calcula la inclinación y la normal de la superficie
4. Aplica iluminación para que se vea como agua realista
5. Permite mover la cámara y ajustar la amplitud y velocidad de las olas

Es una simulación visual muy elegante de una ecuación matemática aplicada a gráficos 3D.

---

## 20) Créditos y uso

Este proyecto está pensado para enseñar conceptos de:

- cálculo multivariable
- derivadas parciales
- gradiente y normal
- iluminación 3D
- shaders
- gráficos por computadora

Puedes modificarlo para agregar:

- más olas
- más colores
- reflexión del cielo
- turbidez
- interacción con el mouse
- física más compleja

---

Si quieres, en el siguiente paso puedo ayudarte a crear una versión de este README más técnica, más visual o más amigable para GitHub, con ejemplos y capturas.
