# Reciclaje AR

Clasificador de residuos con realidad aumentada para el celular. Apuntás la cámara a un
objeto, la IA lo identifica y te dice **qué es**, **de qué material está hecho** y
**en qué caneca va** (blanca, verde o negra).

## Cómo funciona

- La cámara y la escena AR corren con 8th Wall (runtime y motor servidos localmente).
- La captura se hace con `world.xr.takeScreenshot()` y se reduce a JPEG de 1280 px.
- Se envía a Gemini con `responseSchema`, así la respuesta siempre trae
  `caneca`, `objeto`, `material` y `mensaje`.
- Hay dos claves de API configuradas: si la principal agota su cuota, la app
  prueba automáticamente la clave de respaldo sin interrumpir al usuario.
- Dentro de cada clave se prueban varios modelos lite: si uno falla por cuota,
  saturación (`503`) o no existe, se pasa al siguiente.
- Si no hay internet o se agotan ambas claves, se muestra un resultado de demostración.

## Requisitos

- Node.js 18 o superior.
- Para probar en el celular: celular y computadora en la misma red Wi-Fi.

## Desarrollo

```bash
npm install
npm run build
npm run preview
```

`npm run preview` compila y levanta un servidor HTTPS en el puerto `8443`.
La consola imprime la URL para abrir desde el celular, por ejemplo:

```
https://192.168.1.50:8443
```

## HTTPS con mkcert (recomendado)

La cámara sólo funciona en contexto seguro (`https://` o `localhost`). El servidor usa
[mkcert](https://github.com/FiloSottile/mkcert) si está disponible; si no, genera un
certificado autofirmado con `openssl`.

El proyecto incluye un binario de mkcert en `tools/mkcert`. Para que el celular confíe
en el certificado sin advertencias:

1. Abrí `https://<IP-DE-TU-PC>:8443/ca.crt` desde el celular.
2. Instalá el certificado como **CA de confianza**.
   - Android: Ajustes → Seguridad → Cifrado y credenciales → Instalar certificado → CA.
   - iPhone: Ajustes → General → VPN y dispositivo → instalar perfil, y luego activarlo
     en Ajustes → General → Acerca de → Ajustes de confianza de certificados.

También podés entrar aceptando el aviso de certificado, pero instalar la CA evita
advertencias y problemas con la cámara.

## Indicador de cuota de la IA

Arriba a la derecha hay un indicador con forma de píldora (`IA 3/20`). Muestra:

- Los escaneos usados hoy (contador local, guardado en `localStorage`).
- El límite diario real informado por la API cuando devuelve un error `429`.
- El estado de cada clave: en uso, disponible o sin cupo con el tiempo de reinicio.
- El tiempo estimado hasta que se reinicie el cupo.

Google no expone el saldo restante en tiempo real sin gastar una petición, por eso el
contador es local. El botón abre un panel con el detalle de las dos claves y un enlace
al uso real en AI Studio.

La persistencia se guarda en `localStorage` con la clave `reciclaje-ar-cuota-v2` y se
migra automáticamente desde el formato anterior (`reciclaje-ar-cuota-v1`) sin perder el
contador acumulado.

## Modo demostración

- Forzado por URL: `https://<IP>:8443/?demo=1`
- Automático: cuando no hay conexión o la API falla.

## Linterna

La app muestra un botón redondo con 🔦 arriba del botón de escanear cuando la cámara
está activa y el dispositivo permite controlar el torch desde el navegador. Al tocarlo
se enciende o apaga. En iPhone/Safari el torch no está expuesto a la web, así que el
botón no aparece.

## Instalar como app y usarla sin el PC

El proyecto es una PWA:

- Tiene `manifest.webmanifest`, iconos y un service worker (`sw.js`) que guarda la app,
  el motor de 8th Wall y los recursos en el celular.
- Una vez abierta desde HTTPS e instalada, se puede abrir sin el PC encendido.
- La clasificación con IA necesita internet. Sin conexión se usa el modo demostración.
- En Android, el navegador muestra el botón de instalar o la opción en el menú.
  En iPhone: Compartir → “Agregar a pantalla de inicio”.

Para que funcione sin el PC hace falta publicar `dist/` una vez en un hosting HTTPS
(Netlify Drop, GitHub Pages, Vercel, etc.). El servidor local con mkcert sirve para
desarrollar y probar, pero no para dejarla funcionando sin la computadora.

## Publicar

```bash
npm run build
```

Luego subí el contenido de `dist/` a un hosting HTTPS. Netlify Drop es la opción más
simple: arrastrá la carpeta `dist` en <https://app.netlify.com/drop>. El service worker
y la cámara requieren HTTPS, por eso no alcanza con abrir el archivo directamente.

## Estructura

- `src/index.html`: interfaz mobile-first (tarjeta flotante, guía, cuota y linterna).
- `src/app.js`: captura, Gemini, fallback de claves/modelos, linterna y estado de la UI.
- `src/.expanse.json`: escena de 8th Wall (cámara trasera, luces y pantalla de carga).
- `pwa/manifest.webmanifest` y `pwa/sw.js`: instalación y caché offline.
- `scripts/serve.js`: servidor HTTPS local con mkcert/openssl y ruta `/ca.crt`.
- `dist/`: build listo para publicar (generado con `npm run build`).
