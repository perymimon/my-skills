---
name: remult
description: General reference for using Remult — data providers, withRemult context, Express custom routes, Telegraf bot integration, and multer file uploads.
user-invokable: true
args:
  - name: topic
    description: Specific area to focus on (optional) — e.g. "multer", "telegraf", "data provider"
    required: false
---

Remult uses `AsyncLocalStorage` to carry its context through a request. Every integration point must preserve that context correctly. Here is everything learned the hard way.

---

## Data Provider Setup

```ts
import { JsonFileDataProvider } from 'remult/server'   // NOT from 'remult'
import { remult } from 'remult'

remult.dataProvider = new JsonFileDataProvider('./data')
```

- **Dev**: always use `JsonFileDataProvider` — zero config, files land in `./data`
- **Production**: the user decides where and how data is stored — ask them before choosing a provider. Common options: `BetterSqlite3DataProvider` (`remult/remult-better-sqlite3`), PostgreSQL via `createPostgresDataProvider` (`remult/postgres`), or any other Remult-supported adapter
- Set it on the global `remult` singleton so any code calling `remult.repo()` outside HTTP context (bot handlers, scripts, tests) uses the right provider

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

**Express `next` takes 3 params** (`req, res, next`) — if you write `(_, next)` then `next` receives `res` as its argument and crashes. Always use `(_req, _res, next)`.

---

## Body Parsing: MUST happen BEFORE withRemult

Stream-based body parsing fires async callbacks that escape `AsyncLocalStorage`. Parse the body first, then enter `withRemult`.

```ts
// ✅ Correct order
router.use(express.json())
router.use((_req, _res, next) => withRemult(() => next()))

// ❌ Wrong — body-parse callbacks lose remult context
router.use((_req, _res, next) => withRemult(() => next()))
router.use(express.json())
```

Same applies to `express.raw()`, `express.urlencoded()`, and multer — all must come before `withRemult`.

---

## Remult + Multer: The Core Problem

Multer's async disk write (`fs.WriteStream`) breaks `AsyncLocalStorage`. After multer finishes, the remult context is gone — `remult.repo()` throws or returns nothing.

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

**Usage:**

```ts
import { withRemultUpload } from './middleware/remultUpload'

// Single file
router.post('/upload', ...withRemultUpload(upload.single('file')), async (req, res) => {
  const record = await remult.repo(MyEntity).findId(id)  // ✅ works
})

// Multiple files
router.post('/photos', ...withRemultUpload(upload.array('photos', 20)), async (req, res) => {
  const item = await remult.repo(MyEntity).findId(id)    // ✅ works
})
```

**Spread is required** — `withRemultUpload()` returns an array; use `...` to unpack into Express's middleware chain.

---

## Multer Configuration Patterns

```ts
import multer from 'multer'
import { v4 as uuidv4 } from 'uuid'
import os from 'os'

// Temp files — move to final destination in handler (simplest)
const upload = multer({ dest: os.tmpdir() })

// Disk storage with UUID filenames
const storage = multer.diskStorage({
  destination: (_req, _file, cb) => {
    fs.ensureDirSync(uploadsDir)
    cb(null, uploadsDir)
  },
  filename: (_req, file, cb) => {
    cb(null, `${uuidv4()}-${file.originalname}`)
  },
})

// With type filtering
const imageUpload = multer({
  storage,
  limits: { fileSize: 10 * 1024 * 1024 },  // 10 MB
  fileFilter: (_req, file, cb) => {
    const ok = file.mimetype.startsWith('image/')
    ok ? cb(null, true) : cb(new Error('Images only'))
  },
})
```

**Temp-file + move pattern** (upload to tmpdir, move in handler — keeps upload config simple):

```ts
router.post('/photos', ...withRemultUpload(upload.array('photos', 20)), async (req, res) => {
  const files = req.files as Express.Multer.File[]
  const item = await remult.repo(MyEntity).findId(req.params.id)

  for (const file of files) {
    const ext = path.extname(file.originalname) || '.jpg'
    const dest = path.join(finalDir, `${uuidv4()}${ext}`)
    await fs.move(file.path, dest)
    item.photos.push(dest)
  }

  await remult.repo(MyEntity).save(item)
  res.json(item)
})
```

---

## Remult + Telegraf

```ts
import { withRemult } from 'remult'   // from 'remult', NOT 'remult/server'

// ✅ Correct — arrow function wraps next()
bot.use((_, next) => withRemult(() => next()))

// ❌ Wrong — withRemult(next) passes the remult instance as arg to next()
// Telegraf throws: "next(ctx) called with invalid context"
bot.use((_, next) => withRemult(next))
```

`withRemult(callback)` calls `callback(remultInstance)`, so `next` receives the remult instance as its first argument instead of the Telegraf context. Always wrap: `() => next()`.

---

## Testing

```ts
import { InMemoryDataProvider, remult } from 'remult'

beforeEach(() => {
  remult.dataProvider = new InMemoryDataProvider()
})
```

- `InMemoryDataProvider` from `'remult'` — fully isolated, no files written
- Run tests: `node --import tsx --test tests/*.test.ts`

---

## Quick Reference

| Situation | Pattern |
|-----------|---------|
| Custom Express router needs remult | `router.use((_req, _res, next) => withRemult(() => next()))` |
| Multer upload + remult after | `...withRemultUpload(upload.single('file'))` |
| Body parsing order | Parse body first, then enter `withRemult` |
| Telegraf bot needs remult | `bot.use((_, next) => withRemult(() => next()))` |
| Tests | `remult.dataProvider = new InMemoryDataProvider()` |
| JsonFileDataProvider import | `from 'remult/server'` (not `'remult'`) |
| withRemult import | `from 'remult'` (not `'remult/server'`) |
