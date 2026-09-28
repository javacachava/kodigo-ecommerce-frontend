# Kodigo Store — Frontend Next.js (Tarea 9)

Frontend en **Next.js 16 (App Router) + TypeScript** que consume la API REST de
e-commerce de Laravel (`../ecommerce-api`, Laravel 12 + Swagger + Stripe en
modo simulacion). Implementa catalogo publico, autenticacion, carrito,
checkout con pago (Stripe) e historial de compras.

## Stack

- Next.js 16 (App Router, Turbopack), React 19, TypeScript.
- Tailwind CSS v4.
- Server Components para lecturas, Server Actions para mutaciones.
- Token JWT de la API guardado en cookie **httpOnly** (nunca llega al cliente).
- `proxy.ts` (Proxy — el reemplazo de "Middleware" en Next.js 16) para checks
  optimistas de rutas protegidas.

## Configuracion

1. Copia el archivo de entorno de ejemplo:

   ```bash
   cp .env.example .env.local
   ```

2. Ajusta `API_BASE_URL` si tu API Laravel no corre en `http://localhost:8000`:

   ```bash
   # .env.local
   API_BASE_URL=http://localhost:8000/api
   ```

   Esta variable **no** lleva prefijo `NEXT_PUBLIC_`: todas las llamadas a la
   API ocurren en el servidor (Server Components / Server Actions), nunca
   desde el navegador, para que el token JWT nunca se exponga al cliente.

## Ejecutar el proyecto

Con la API Laravel corriendo (`cd ../ecommerce-api && php artisan serve`):

```bash
npm install
npm run dev
```

Abre [http://localhost:3000](http://localhost:3000) (redirige a `/products`).

Otros comandos:

```bash
npm run build     # build de produccion
npm run start     # sirve el build de produccion
npm run lint      # ESLint
npx tsc --noEmit  # chequeo de tipos
```

## Rutas implementadas

| Ruta | Tipo | Descripcion |
|---|---|---|
| `/products` | Publica | Catalogo, con busqueda (`?search=`) y orden (`?sort=`). `loading.tsx` + `Suspense` en el grid, `error.tsx` con reintento. |
| `/products/[id]` | Publica | Detalle de producto + boton "Agregar al carrito" (estado local). |
| `/login`, `/register` | Publica | Formularios con Server Actions + `useActionState`, validacion mostrada por campo (errores 422 de Laravel). |
| `/cart` | Publica | Revision del carrito (localStorage), cantidades editables. |
| `/checkout` | Protegida | Revisa el carrito y crea la orden (`POST /orders`) + intento de pago (`POST /orders/{id}/payments`). |
| `/checkout/[id]` | Protegida | Pantalla de pago: confirma el `PaymentIntent` (`POST /payments/{id}/confirm`) en modo simulacion Stripe (exito o rechazo, con reintento si falla). |
| `/checkout/[id]/success` | Protegida | Confirmacion de compra. |
| `/account/orders` | Protegida | Historial de compras, `Suspense` + `loading.tsx`. |
| `/account/orders/[id]` | Protegida | Detalle de una orden y su pago. |

"Protegida" = requiere sesion. Se verifica en dos capas:

1. **Optimista**: `proxy.ts` redirige a `/login?redirect=...` si no hay cookie de sesion.
2. **Real**: cada Server Component/Server Action llama a `requireUser()` /
   valida el usuario contra `GET /auth/me` antes de tocar datos.

## Autenticacion

- `POST /auth/register` y `POST /auth/login` se invocan desde Server Actions
  (`app/actions/auth.ts`).
- El `access_token` (JWT) se guarda con `cookies().set()` dentro de la Server
  Action, con `httpOnly: true`, `sameSite: "lax"`, `secure` en produccion y
  `maxAge` igual a `expires_in`.
- Cada llamada autenticada a la API reenvia el token como
  `Authorization: Bearer <token>` desde el servidor (`lib/api/client.ts`).
- `logoutAction` invalida el token en la API (`POST /auth/logout`) y borra la
  cookie.

## Carrito y checkout

- El carrito es **estado local del cliente** (`lib/cart/cart-context.tsx`),
  persistido en `localStorage` para sobrevivir refrescos de pagina.
- Al confirmar la orden (`placeOrderAction`) se crean, en el servidor:
  1. La orden (`POST /orders`) con las lineas del carrito.
  2. El `PaymentIntent` (`POST /orders/{id}/payments`).
- En `/checkout/[id]` se confirma el pago contra la API en modo simulacion de
  Stripe: **"Pagar con Stripe"** usa `pm_card_visa` (exito) y **"Simular
  rechazo"** usa `pm_card_chargeDeclined` (fallo controlado, la orden queda
  `pending` y se puede reintentar).

## Cache y revalidacion

- El catalogo se cachea con `next: { tags: ["products"], revalidate: 60 }`.
- Los datos de ordenes/pagos son por-usuario y sensibles al tiempo:
  `revalidate: 0` (siempre frescos).
- Tras crear una orden o confirmar un pago, las Server Actions llaman
  `revalidatePath()` sobre `/account/orders`, `/account/orders/[id]` y
  `/checkout/[id]` para que la UI nunca muestre datos desactualizados tras una
  mutacion.

## Resiliencia

- `loading.tsx` en `/products` y `/checkout`.
- `error.tsx` con boton de reintento en `/products` y `/account/orders`.
- `<Suspense>` alrededor del grid de productos y de la lista de ordenes
  (secciones pesadas que dependen de la API externa).
- Errores 404/403 de la API se traducen a `notFound()` de Next.js.

## Evidencia de pruebas

Flujo completo verificado end-to-end con Playwright contra la API real
(catalogo, busqueda, carrito, login, orden, pago rechazado + reintento exitoso,
historial, proteccion de rutas, registro, aislamiento de datos entre
usuarios). Capturas en [`docs/screenshots/`](docs/screenshots/).

Reporte Lighthouse (build de produccion, `/products`) en
[`docs/lighthouse/products-report.html`](docs/lighthouse/products-report.html):

| Categoria | Puntaje |
|---|---|
| Performance | 90 |
| Accessibility | 100 |
| Best Practices | 100 |
| SEO | 100 |

Imagenes de producto migradas de `<img>` a `next/image` (con `remotePatterns`
en `next.config.ts` para el host de imagenes de prueba) tras detectar un LCP
de 5.1s en la primera corrida; bajo a 3.3s solo con esa optimizacion.

Pendiente de captura manual (requiere Swagger UI abierto en el navegador):
capturas de los endpoints consumidos desde `/api/documentation` en
`../ecommerce-api`.

## Estructura relevante

```
app/
  actions/           Server Actions (auth, checkout)
  products/          Catalogo publico
  cart/              Carrito (cliente)
  checkout/          Flujo de compra
  account/orders/    Historial de compras
  login/, register/  Autenticacion
lib/
  api/               Cliente HTTP tipado hacia la API Laravel
  auth/              Cookie de sesion + Data Access Layer (DAL)
  cart/              Contexto de carrito (localStorage)
proxy.ts             Proteccion optimista de rutas (antes "middleware.ts")
```
