---
name: remult-integrate
description: Remult integration patterns — withRemult context, Express custom routes, multer uploads, Telegraf bot, and data providers. Use when integrating Remult into an Express server, bot, or file upload handler.
user-invocable: true
argument-hint: "[topic: multer | telegraf | data-provider | testing]"
---

Remult uses `AsyncLocalStorage` to carry its context through a request. Every integration point must preserve that context correctly. Here is everything learned the hard way.

If an argument is provided, focus only on that topic. Otherwise cover the full reference.

---

## Data Provider Setup

```ts
import { JsonFileDataProvider } from 'remult/server'   // NOT from 'remult'
import { remult } from 'remult'

remult.dataProvider = new JsonFileDataProvider('./data')
```

- `JsonFileDataProvider` lives in `remult/server`, not `remult`  ❌ `from 'remult'`
- Set it on the global `remult` singleton so code outside HTTP context (bots, scripts, tests) uses the right provider

---

## Remult + Express: Custom Routers

`app.use(api.withRemult)` only covers Remult's auto-generated CRUD routes. Custom routers need their own context wrapper.

```ts
import { withRemult } from 'remult'
import { Router } from 'express'

export const myRouter = Router()

// MUST be first middleware on the router
myRouter.use((_req, _res, next) => withRemult(() => next()))
```

**Express `next` takes 3 params** — if you write `(_, next)` then `next` receives `res` as its value and crashes. Always use `(_req, _res, next)`.

---

## Body Parsing: MUST happen BEFORE withRemult

Stream-based body parsing fires async callbacks that escape `AsyncLocalStorage`. Parse body first, then enter `withRemult`.

```ts
// ✅ Correct order
router.use(express.json())
router.use((_req, _res, next) => withRemult(() => next()))

// ❌ Wrong — body-parse callbacks lose remult context
router.use((_req, _res, next) => withRemult(() => next()))
router.use(express.json())
```

Same applies to `express.raw()`, `express.urlencoded()`, and multer.

---

## Remult + Multer

Multer's async disk write (`fs.WriteStream`) breaks `AsyncLocalStorage`. After multer finishes, the remult context is gone.

**Fix: re-enter withRemult after multer finishes**

```ts
// middleware/remultUpload.ts
import { withRemult } from 'remult'
import type { RequestHandler } from 'express'

export const reenterRemult: RequestHandler = (_req, _res, next) => {
  withRemult(() => Promise.resolve(next())).catch(next)
}

export function withRemultUpload(...multerMiddlewares: RequestHandler[]): RequestHandler[] {
  return [...multerMiddlewares, reenterRemult]
}
```

```ts
// Single file
router.post('/upload', ...withRemultUpload(upload.single('file')), async (req, res) => {
  const record = await remult.repo(MyEntity).findId(id)  // ✅ works
})
```

**Spread is required** — `withRemultUpload()` returns an array, use `...` to unpack.

---

## Remult + Telegraf

```ts
import { withRemult } from 'remult'   // from 'remult', NOT 'remult/server'

// ✅ Correct — arrow function wraps next()
bot.use((_, next) => withRemult(() => next()))

// ❌ Wrong — passes remult instance as arg to next()
bot.use((_, next) => withRemult(next))
```

`withRemult(callback)` calls `callback(remultInstance)` — so `next` receives the remult instance instead of the Telegraf context.

---

## Testing

```ts
import { InMemoryDataProvider, remult } from 'remult'

beforeEach(() => {
  remult.dataProvider = new InMemoryDataProvider()
})
```

Run tests: `node --import tsx --test tests/*.test.ts`

---

## Quick Reference

| Situation | Pattern |
|-----------|---------|
| Custom Express router | `router.use((_req, _res, next) => withRemult(() => next()))` |
| Multer + remult | `...withRemultUpload(upload.single('file'))` |
| Body parsing order | Parse body first, then `withRemult` |
| Telegraf bot | `bot.use((_, next) => withRemult(() => next()))` |
| Tests | `remult.dataProvider = new InMemoryDataProvider()` |
| JsonFileDataProvider import | `from 'remult/server'` |
| withRemult import | `from 'remult'` |
