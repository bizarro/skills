# Sanity

Sanity is the CMS on bespoke sites (Prismic came before it, until 2024). The Studio is its own app: `apps/backend` in a Lisergia monorepo, or a `<project>-backend` repository next to the frontend.

## Studio Layout

```
schemaTypes/
  index.ts       one array, grouped with comments: // Shared, // Types, // Modules, // Settings
  shared/        content (page builder), internal (case study builder), image, link, social
  general/       routable documents: page, project, article, category…
  global/        settings
  layout/        menu, footer, newsletter
  modules/       page-builder objects, one per section: hero, gallery, list, quote…
    case/        blocks only case studies use
structure/
  index.ts       the desk structure
presentation/
  resolve.ts     preview locations for the Presentation tool
```

- One `export const <name> = defineType(...)` per file, named after the type.
- Write every field with `defineField` and every array member with `defineArrayMember`.
- Sort the keys of `defineType` and `defineField` alphabetically (`fields, name, orderings, preview, title, type`), like any other object.

## Documents

Routable documents keep the same field order, with the page builder last:

```ts
export const page = defineType({
  fields: [
    defineField({
      name: 'slug',
      options: { maxLength: 96, source: 'title' },
      title: 'Slug',
      type: 'slug',
    }),
    defineField({ name: 'social', title: 'Social', type: 'social' }),
    defineField({ name: 'title', title: 'Title', type: 'string' }),
    defineField({ name: 'content', title: 'Content', type: 'content' }),
  ],
  name: 'page',
  title: 'Page',
  type: 'document',
})
```

- Orderable collections put `orderRankField({ newItemPosition: 'before', type: 'project' })` first and add `orderings: [orderRankOrdering]`. The frontend query must then sort with `| order(orderRank)`, or the order editors set in the Studio is ignored.
- `social` (`image`, `title`, `description`) is the only SEO object. It sits on every routable document and maps one to one onto `<title>`, `description`, `og:*` and `twitter:*`.
- `link` is `{ text, url }`, with `url` validated by `Rule.uri({ allowRelative: true, scheme: ['http', 'https', 'mailto'] })`. Every button in a module is `button: { type: 'link' }`.

## Images

Every image field spreads one shared configuration, so `alt` and the hotspot are never forgotten:

```ts
export const imageConfiguration = {
  fields: [
    defineField({
      description:
        'Describe the image for people who cannot see it. Leave blank only when the image is purely decorative.',
      name: 'alt',
      title: 'Alternative text',
      type: 'string',
      validation: (rule) => rule.required().warning('Alternative text improves accessibility and SEO.'),
    }),
  ],
  options: {
    hotspot: true,
  },
}

defineField({
  name: 'image',
  title: 'Image',
  type: 'image',
  ...imageConfiguration,
})
```

Older backends had neither `alt` nor `hotspot`, and their `Media` components hardcoded `alt=""`. Don't copy that.

## Page Builder and Modules

The page builder is a named array type: `content` for pages, `internal` for case studies and articles. Each member is a module.

```ts
export const content = defineType({
  name: 'content',
  of: [defineArrayMember({ name: 'hero', type: 'hero' }), defineArrayMember({ name: 'gallery', type: 'gallery' })],
  type: 'array',
})
```

- **Field vocabulary.** Modules reuse the same names: `label`, `title`, `description` (rich text, `array` of `block`), `image`, `button` (a `link`) and `list` for any repeater. Name each list member for what it holds (`item`, `link`, `project`).
- **One name, four files.** The `hero` module is `modules/hero.ts` in the Studio, `templates/sections/Hero.tsx` on the frontend, the `.hero` BEM block in `styles/sections/hero.scss`, and an optional `datasets/sections/Hero.ts` behavior registered on `.hero`. The frontend renders the builder with a switch on `_type`. Adding a section means adding all four and registering each.
- **Previews.** Show the module's name as the title and its content as the subtitle, so the builder reads like the page:

```ts
preview: {
  prepare({ title }) {
    return { subtitle: title, title: 'Hero' }
  },
  select: { title: 'title' },
},
```

Media-only modules preview the file name: `select: { media: 'image', title: 'image.asset.originalFilename' }`.

- **Media switch.** A module that holds an image or a video gets a `type` select (`image` or `video`) and hides the other fields with `hidden: ({ parent }) => parent?.type !== 'video'`. Art-directed images come as `desktopImage` and `mobileImage` pairs.
- **Videos are hosted URLs** (Vimeo direct file links), not Sanity uploads. Put the steps to find the link in the field's `description` for the editor.
- **Automatic modules.** A module that lists every document of a type (all projects, all articles) holds only a read-only string explaining that. The frontend fills it in.
- **Settings holds the microcopy**: labels like "Recent projects", newsletter placeholders, button text. Views read it from `settings`, not from hardcoded strings.
- **Every field is optional on the frontend.** A section that is missing what it needs renders nothing. Never invent copy or fall back to placeholder content.

## Desk Structure

Collections first, then the singletons, each with an icon from `@sanity/icons`:

```ts
export const structure: StructureResolver = (S, context) =>
  S.list()
    .id('root')
    .title('Project')
    .items([
      S.documentTypeListItem('page').title('Pages').icon(TiersIcon),
      orderableDocumentListDeskItem({
        context,
        S,
        title: 'Projects',
        type: 'project',
      }),
      S.divider(),
      S.listItem().title('Menu').id('menu').child(S.document().schemaType('menu').documentId('menu')).icon(CogIcon),
      S.listItem()
        .title('Footer')
        .id('footer')
        .child(S.document().schemaType('footer').documentId('footer'))
        .icon(CogIcon),
      S.listItem()
        .title('Settings')
        .id('settings')
        .child(S.document().schemaType('settings').documentId('settings'))
        .icon(CogIcon),
    ])
```

Singletons with a fixed `documentId` can still be duplicated from the "new document" menu, and the frontend reads `const [menu] = …`. Remove their templates and creation actions in `sanity.config.ts`:

```ts
const singletons = new Set(['footer', 'menu', 'settings'])

document: {
  actions: (actions, { schemaType }) => {
    if (singletons.has(schemaType)) {
      return actions.filter(({ action }) => action !== 'duplicate' && action !== 'delete')
    }

    return actions
  },
},
schema: {
  templates: (templates) => templates.filter(({ schemaType }) => !singletons.has(schemaType)),
  types: schemaTypes,
},
```

## Getting Content to the Site

**Build-time snapshot, not request-time queries.** A `download.ts` script runs before the build. It fetches each type with a plain `*[_type == "x"]`, resolves references in JavaScript and writes `content.json`. Views import it, so global data (menu, footer, settings) needs no prop drilling and rendering never waits on the CMS.

```ts
traverse(page, (object, key, value) => {
  if (key !== '_ref' || typeof value !== 'string' || value.startsWith('image-')) {
    return
  }

  const reference = references.find((document) => document._id === value)

  if (reference) {
    replacements.set(object, reference)
  }
})

replacements.forEach((reference, entry) => {
  Object.assign(entry, { ...reference })
})
```

- Routing picks the page by `slug ?? 'home'` and falls back to a `not-found` document with a 404 status.
- **Preview is the only live fetch.** The Studio's Presentation tool enables draft mode on the frontend, which then queries with the `drafts` perspective, a read token and stega encoding. Clean any value used as data rather than text with `stegaClean` (class names, alt text, comparisons).
- Content changes need a rebuild. Trigger it from a Sanity webhook.

## Image URLs

Build URLs and a `srcset` from the asset, crop-aware, with `alt` cleaned of stega:

```ts
const responsiveImageWidths = [320, 480, 640, 768, 1024, 1280, 1440, 1920, 2560, 3200]

function getImageUrl(image: SanityImageSource, width: number) {
  return imageUrlBuilder.image(image).width(width).fit('max').auto('format').quality(80).url()
}

// widths: every candidate below the cropped source width, plus the source width itself.
srcSet: widths.map((width) => `${getImageUrl(image, width)} ${width}w`).join(', ')
```

- The first section of a page renders its media with `loading="eager"` and `fetchPriority="high"`. Everything else gets `data-src` and `data-srcset`, which a `Source` dataset swaps in with a generous `rootMargin` (`100% 0px`), then adds `.loaded` for the fade.
- For blurred placeholders, generate a 20px-wide JPEG as a base64 data URI per image during `download.ts` (`sharp(...).resize(20).jpeg({ quality: 50 })`; for videos, extract the first frame with ffmpeg first) and cache the map in a JSON file. It serves as the WebGL placeholder texture and as the DOM blur under a hero video.
- Some client hosts want media off Sanity's CDN. Then `download.ts` mirrors each asset to Vercel Blob or R2 under `assets/<assetId>.<extension>`, skipping files that already exist, and also uploads the resized widths. Check the rewritten URLs against what was actually uploaded: a helper that rewrites to `.webp` while the script uploads originals breaks every image.

## Portable Text

Render rich text on the server: `toHTML` from `@portabletext/to-html` in Preact templates, `<PortableText>` in React ones. Nothing about it reaches the client bundle. Where the text sits inline (inside a heading or a link), override `block.normal` to drop the wrapping `<p>`.

## Studio Setup

- `sanity.config.ts`: `projectId`, `dataset: 'production'`, and `plugins: [structureTool({ structure }), presentationTool({ … }), visionTool()]`.
- `sanity.cli.ts` reads the project from the environment and sets `deployment.autoUpdates: true`. Deploy with `sanity deploy` to `<project>.sanity.studio`.
- Give each Studio its own dev port so it can run next to the frontend.
