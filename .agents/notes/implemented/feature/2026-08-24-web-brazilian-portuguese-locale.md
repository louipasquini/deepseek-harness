# Agent Note: Web Brazilian Portuguese locale

Status: implemented

English | [中文](2026-08-24-web-brazilian-portuguese-locale.zh.md)

## Problem

The Web client shipped two locales — `zh` and `en` — and the Language row in Settings let a user pick between them. Brazilian Portuguese is one of the largest developer audiences the product has no surface for: a `pt-BR` browser opened in the `en` fallback and the language menu offered no way to choose Portuguese.

## Decision

Ship Brazilian Portuguese as the third locale under the primary-subtag id `pt`, with `<html lang>` resolving to the BCP 47 tag `pt-BR` — the same arrangement `zh` → `zh-CN` already uses, because the locale id is the app's own vocabulary and the document attribute wants a full tag. The Language row labels it `Português (Brasil)`.

The dictionary completeness contract extends unchanged: `LocaleId` becomes `'zh' | 'en' | 'pt'`, and the typed `register(ns, { zh, en, pt })` overload plus every package's `satisfies Record<Key, string>` check make a missing or extra `pt` key a compile error. Every one of the 27 client locale namespaces now carries a `pt` dictionary; the fallback chain (active locale → `en` → key) and `FALLBACK_LOCALE = 'en'` are untouched, so a key any locale misses still resolves to English rather than Chinese. Browser detection needs no code change: `detectBrowserLocale` already matches on the primary subtag, so `pt-BR` lands on `pt`.

Copy follows one rule: natural Brazilian Portuguese, keeping `{placeholder}` tokens verbatim and leaving product nouns (`Full access`, `plan mode`, `Session`, `API key`) untranslated the way the English side does. The welcome-notice copy and the directory-picker dialog dictionaries, which register outside the typed per-package bundles, received `pt` entries too.

## Alternatives considered

**A distinct `pt-BR` id.** Rejected for symmetry: the app's ids are primary subtags, and the existing `zh` already proves the mapping to a full document tag belongs in `DOCUMENT_LANGUAGE`, not in the id.

**Portuguese (Portugal) alongside Brazil.** Rejected as out of scope. The request is Brazilian Portuguese; a European variant can join later without reworking the mechanism, since nothing in the registry is dialect-specific.

**Leaving untyped registrations on `en` fallback.** Rejected for the same reason every other namespace ships every locale: a browser that resolves to `pt` should never read mixed-language chrome, and the fallback exists for genuine misses, not for skipped dictionaries.

## Testing

`locale.client.spec.ts` pins the three self-described labels, and the browser-detection scenario now asserts `pt-BR` resolves to `pt`; `apply.client.spec.ts` asserts the `pt` seat is occupied and the Language row mirrors all three ids. Each package's dictionary test (where present) extends to the `pt` bundle, and the existing `test:gui` suites cover the rest. The `settings-chrome` e2e goldens are unchanged: the pill renders the active locale's own name, so a `zh`/`en` golden shows no third entry.

## Consequences

**Every future dictionary ships three locales.** The `LocaleId` union enforces it at compile time; a new package that registers only `zh`/`en` fails to build. The cost is bounded because the key sets are small and the `en` side is the natural source for a translator.

**Browser resolution changed for Portuguese speakers.** A `pt-BR` (or `pt`-prefixed) browser now boots in Portuguese without a stored preference, where it previously fell through to `en`. The `fr-FR` e2e scenario still exercises the unshipped-language path.

**The label is not the id.** `pt` is persisted in `locale.preference` and travels in the settings document; the display label `Português (Brasil)` is presentation. Renaming the label never touches stored values.
