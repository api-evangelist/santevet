---
name: Resolve SantéVet species and breeds
description: >-
  Turn a free-text pet description ("a 3-year-old Labrador") into the integer species and breed
  identifiers every other SantéVet API expects. Do this FIRST — the rating and quotation
  operations take integers, not names, and publish no name lookup of their own.
api: openapi/santevet-toolkit-openapi.yml
base: https://toolkit.api.santevet.com
operations:
  - getEspeceCollection
  - get_species_breedsRaceCollection
  - getRaceCollection
  - getRaceItem
generated: '2026-08-17'
method: generated
source: openapi/santevet-toolkit-openapi.yml
---

# Resolve SantéVet species and breeds

The SantéVet Toolkit API is the reference-data registry behind every other SantéVet surface.
Breeds and species are the identifiers you need before you can quote, subscribe or file a claim.

## Before you start

- **You need a partner API key.** Send it in the `Authorization` request header. There is no
  self-serve signup — keys come from the B2B partner process at
  <https://www.santevet.com/partenaire-btob>. Every operation returns `401` without it.
- **Vocabulary is mixed French/English.** `Espece` means species, `Race` means breed. Operation
  ids and schema names use the French nouns; paths use the English ones (`/species`, `/breeds`).
- **Ask for `application/ld+json`, not `application/json`.** The plain-JSON projection returns a
  bare array with no envelope, no total and no pagination links. The `ld+json` projection returns
  `hydra:member`, `hydra:totalItems` and a `hydra:view` with `hydra:first` / `hydra:last` /
  `hydra:previous` / `hydra:next`. If you negotiate plain JSON on a long collection you cannot
  tell whether you have all of it.

## Steps

1. **List the species.** Call `getEspeceCollection` (`GET /species`). This is a short list —
   SantéVet insures dogs, cats and small pets. Match the user's animal to one species `id`.
   Supports `?id=` and `?id[]=` exact filters if you already know the identifier.

2. **List the breeds for that species.** Call `get_species_breedsRaceCollection`
   (`GET /species/{id}/breeds`) with the species id from step 1. Prefer this over the unscoped
   breed list — it is much shorter and it cannot return a cat breed for a dog.

3. **Fall back to the full breed list only if you must.** `getRaceCollection` (`GET /breeds`)
   returns every breed across all species. Use `?id=` / `?id[]=` when you already hold
   identifiers. There is no name-search parameter on this collection, so do not attempt
   `?name=` — match client-side on the returned records instead.

4. **Confirm a single breed.** `getRaceItem` (`GET /breeds/{id}`) returns one breed and `404` if
   the identifier does not exist. Use it to validate an identifier you were handed rather than
   one you resolved yourself.

5. **Carry the integers forward.** The identifiers you now hold are what the acquisition API's
   rating call consumes as `breed`, `father_breed` and `mother_breed`, and what the quotation
   write consumes as `prospect[animals][][breed]`.

## Cross-breed animals

The acquisition API accepts `father_breed` and `mother_breed` alongside `breed`, so a crossbreed
is expressed as three breed identifiers resolved from the same registry. Resolve each one through
the steps above.

## Rules

- **Never invent a breed or species identifier.** They are opaque auto-increment integers with no
  prefix and no checksum, so a guessed value can silently resolve to a different animal and
  produce a wrong insurance quote. If you cannot resolve a name, say so and stop.
- **A `404` means the identifier is wrong, not that the animal is uninsurable.** Re-resolve from
  `/species/{id}/breeds` before reporting anything to the user.
- **A `401` means your key is missing or wrong**, even though no operation in the specification
  declares a `401` response. Do not read it as "breed not found".
- **Do not retry on `400` or `422`.** Both mean the request was malformed; retrying it
  unchanged will fail identically.
- **There is no rate-limit signal.** SantéVet publishes no `RateLimit-*` or `Retry-After`
  headers and no documented quota, so pace yourself conservatively and cache the reference data.
  Species and breed lists change rarely — cache them for the session rather than re-fetching per
  animal.
