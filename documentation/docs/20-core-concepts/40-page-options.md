---
title: Page options
---

스벨트킷의 기본 설정은, 서버에서 먼저 렌더링(또는 [프리랜더링](glossary#prerendering))을 하고, 클라이언트에게는 HTML로 보내주는 것이다. 그러면 그 이후에 스벨트킷은 사용자와 상호 작용이 가능하도록 브라우저에서 해당 컴포넌트를 다시 렌더링하게 되는데, 이 과정을 [**하이드레이션**](glossary#hydration).이라고 한다. 그래서 프로그래머는 컴포넌트들이 두 곳 모두에서 실행이 가능하도록 확인할 필요가 있다. 스벨트킷은 그 후에 [**라우터**](routing)를 초기화하고, 그러면 라우터는 이후의 네비게이션을 담당하게 된다.

You can control each of these on a page-by-page basis by exporting options from , or for groups of pages using a shared [`+layout.js`](routing#layout-layout-js) or [`+layout.server.js`](routing#layout-layout-server-js). To define an option for the whole app, export it from the root layout. Child layouts and pages override values set in parent layouts, so — for example — you can enable prerendering for your entire app then disable it for pages that need to be dynamically rendered.
이 각각의 과정들은 페이지별로 조절이 가능하고, [`+page.js`](routing#page-page-js) 파일이나 [`+page.server.js`](routing#page-page-server-js) 파일에서 옵션을 export 하면 되는 것이다. 여러 페이지들을 그룹별로 조절도 가능하다.

앱의 각각의 부분들에서 이런 옵션들을 적절히 조합하고 맞추어서 사용할 수 있다. 예를 들어서 마케팅 페이지는 속도를 높이기 위해 프리랜더링을 하고, SEO와 접근을 위한 동적 페이지들은 서버사이드 렌더링을 하고, 관리 페이지들은 클라이언트에서만 렌더링되는 SPA로 전환하는 식이다. SvelteKit은 이런 특성들로 인해서 매우 다용도로 활용할 수 있다.

## 사전 렌더링(prerender)

앱의 라우터들 중, 컴파일 시 단순한 HTML 파일이 생성되는 라우터들이 있을 수 있습니다. 바로 이런 라우터들을 [_사전랜더링(프리랜더링)_](glossary#prerendering)하면 됩니다.

```js
/// file: +page.js/+page.server.js/+server.js
export const prerender = true;
```

다른 방법으로는, 루트 '+layout.js'나 '+layout.server.js' 파일에 'export const prerender = true'를 세팅해서 프리렌더링을 _하지 못하게_ 따로 표시해 둔 페이지들을 제외한 모든 페이지들이 프리렌더링이 되도록 할 수 있습니다.:

```js
/// file: +page.js/+page.server.js/+server.js
export const prerender = false;
```

Routes with `prerender = true` will be excluded from manifests used for dynamic SSR, making your server (or serverless/edge functions) smaller. In some cases you might want to prerender a route but also include it in the manifest (for example, with a route like `/blog/[slug]` where you want to prerender your most recent/popular content but server-render the long tail) — for these cases, there's a third option, 'auto':
'prerender = true'가 주어진 라우트는, 동적 서버

```js
/// file: +page.js/+page.server.js/+server.js
export const prerender = 'auto';
```

> 전체 앱이 사전렌더링에 적합하다면, [`adapter-static`](https://github.com/sveltejs/kit/tree/main/packages/adapter-static)을 사용해서 다른 아무 정적 웹서버에도 사용할 수 있는 정적 파일들을 출력할 수 있습니다.

The prerenderer will start at the root of your app and generate files for any prerenderable pages or `+server.js` routes it finds. Each page is scanned for `<a>` elements that point to other pages that are candidates for prerendering — because of this, you generally don't need to specify which pages should be accessed. If you _do_ need to specify which pages should be accessed by the prerenderer, you can do so with [`config.kit.prerender.entries`](configuration#prerender), or by exporting an [`entries`](#entries) function from your dynamic route.
사전랜더링 엔진은 앱의 루트에서부터 시작해서 프리랜더링이 가능한 파일이나 '+server.js' 라우트를 찾아서 파일들을 만듭니다. 각 페이들에서 다른 페이지로 연결되는 <a> 요소들을 스캔해서 찾고(이것들이 사전 렌더링의 후보들이 됩니다.)


While prerendering, the value of `building` imported from [`$app/environment`](modules#$app-environment) will be `true`.

### Prerendering server routes

These files are _not_ affected by layouts, but will inherit default values from the pages that fetch data from them, if any.
다른 페이지 선택사항들과는 다른 점이, 'prerender' 옵션은 '+server.js' 파일에도 적용된다는 것입니다. 이 파일들은 레이아웃에 영향을 받지 _않지만_ 레이아웃에서 fetch해서 받아온 데이터들로부터 기본값을 상속받습니다. 예를 들어서, '+page.js' 파일이 아래와 같은 'load' 함수를 가지고 있다고 해 보면,

```js
/// file: +page.js
export const prerender = true;

/** @type {import('./$types').PageLoad} */
export async function load({ fetch }) {
	const res = await fetch('/my-server-route.json');
	return await res.json();
}
```

...그러면 `src/routes/my-server-route.json/+server.js` 파일은 'export const prerender = 'false'가 없더라도 사전 렌더링이 가능하다고 취급될 겁니다.
will be treated as prerenderable if it doesn't contain its own `export const prerender = false`.

### When not to prerender

기본쥭인 규칙은: 아무나 두 사용자가 클릭해도 서버로부터 동일한 내용이 생성되는 페이지의 경우에 '사전 렌더링'이 가능하다는 것입니다.

> Not all pages are suitable for prerendering. Any content that is prerendered will be seen by all users. You can of course fetch personalized data in `onMount` in a prerendered page, but this may result in a poorer user experience since it will involve blank initial content or loading indicators.

Note that you can still prerender pages that load data based on the page's parameters, such as a `src/routes/blog/[slug]/+page.svelte` route.

Accessing [`url.searchParams`](load#using-url-data-url) during prerendering is forbidden. If you need to use it, ensure you are only doing so in the browser (for example in `onMount`).

[actions](form-actions)이 있는 페이지의 경우에는 사전 랜더링이 불가능한데, 이는 서버가 반드시 action 'POST' 요청을 처리할 수 있어야 하기 때문입니다.

### 라우트 충돌

사전 렌더링은 파일시스템에 파일을 작성하기 때문에, 디렉토리 이름과 파일 이름이 같아지게 하는 종말점을 두개를 만드는 것은 안 되는 문제가 있다. 예를 들어서, 'src/routes/foo/+server.js' 파일과 '/src/routes/foo/bar/+server.js' 파일이 있을 경우에 두 파일을 모두 사전렌더링을 하게 되면 첫번째 파일은 'foo'라는 파일을 만드려고 하고, 두번째 파일은 'foo/bar'라는 파일을 만들려고 할텐데, 이미 'foo'라는 파일이 있기에 'foo/'라는 디렉토리를 만들 수 있는 것이다.

다른 이유들도 있지만, 이런 이유 때문에도 항상 파일의 확장자를 (디렉토리 이름에도) 포함하는 것을 추천합니다. 'src/routes/foo.json/+server.js' 파일과 'src/routs/foo/bar.json/+server.js' 파일을 사전 랜더링을 하게 되면 각각 'foo.json' 파일과 'foo/bar.json'파일 두개가 아무 문제없이 같이 만들어지게 될 것입니다.

_페이지들_ 의 경우에는, 'foo'대신 '/foo/index.html'을 작성해서 이런 문제를 피할 수 있습니다.



### Troubleshooting

If you encounter an error like 'The following routes were marked as prerenderable, but were not prerendered' it's because the route in question (or a parent layout, if it's a page) has `export const prerender = true` but the page wasn't reached by the prerendering crawler and thus wasn't prerendered.

Since these routes cannot be dynamically server-rendered, this will cause errors when people try to access the route in question. There are a few ways to fix it:

* Ensure that SvelteKit can find the route by following links from [`config.kit.prerender.entries`](configuration#prerender) or the [`entries`](#entries) page option. Add links to dynamic routes (i.e. pages with `[parameters]` ) to this option if they are not found through crawling the other entry points, else they are not prerendered because SvelteKit doesn't know what value the parameters should have. Pages not marked as prerenderable will be ignored and their links to other pages will not be crawled, even if some of them would be prerenderable.
* Ensure that SvelteKit can find the route by discovering a link to it from one of your other prerendered pages that have server-side rendering enabled.
* Change `export const prerender = true` to `export const prerender = 'auto'`. Routes with `'auto'` can be dynamically server rendered

## entries

SvelteKit will discover pages to prerender automatically, by starting at _entry points_ and crawling them. By default, all your non-dynamic routes are considered entry points — for example, if you have these routes...

```bash
/             # non-dynamic
/blog         # non-dynamic
/blog/[slug]  # dynamic, because of `[slug]`
```

...SvelteKit will prerender `/` and `/blog`, and in the process discover links like `<a href="/blog/hello-world">` which give it new pages to prerender.

Most of the time, that's enough. In some situations, links to pages like `/blog/hello-world` might not exist (or might not exist on prerendered pages), in which case we need to tell SvelteKit about their existence.

This can be done with [`config.kit.prerender.entries`](configuration#prerender), or by exporting an `entries` function from a `+page.js`, a `+page.server.js` or a `+server.js` belonging to a dynamic route:

```js
/// file: src/routes/blog/[slug]/+page.server.js
/** @type {import('./$types').EntryGenerator} */
export function entries() {
	return [
		{ slug: 'hello-world' },
		{ slug: 'another-blog-post' }
	];
}

export const prerender = true;
```

`entries` can be an `async` function, allowing you to (for example) retrieve a list of posts from a CMS or database, in the example above.

## ssr

Normally, SvelteKit renders your page on the server first and sends that HTML to the client where it's [hydrated](glossary#hydration). If you set `ssr` to `false`, it renders an empty 'shell' page instead. This is useful if your page is unable to be rendered on the server (because you use browser-only globals like `document` for example), but in most situations it's not recommended ([see appendix](glossary#ssr)).

```js
/// file: +page.js
export const ssr = false;
// If both `ssr` and `csr` are `false`, nothing will be rendered!
```

If you add `export const ssr = false` to your root `+layout.js`, your entire app will only be rendered on the client — which essentially means you turn your app into an SPA.

## csr

Ordinarily, SvelteKit [hydrates](glossary#hydration) your server-rendered HTML into an interactive client-side-rendered (CSR) page. Some pages don't require JavaScript at all — many blog posts and 'about' pages fall into this category. In these cases you can disable CSR:

```js
/// file: +page.js
export const csr = false;
// If both `csr` and `ssr` are `false`, nothing will be rendered!
```

Disabling CSR does not ship any JavaScript to the client. This means:

* The webpage should work with HTML and CSS only.
* `<script>` tags inside all Svelte components are removed.
* `<form>` elements cannot be [progressively enhanced](form-actions#progressive-enhancement).
* Links are handled by the browser with a full-page navigation.
* Hot Module Replacement (HMR) will be disabled.

You can enable `csr` during development (for example to take advantage of HMR) like so:

```js
/// file: +page.js
import { dev } from '$app/environment';

export const csr = dev;
```

## trailingSlash

By default, SvelteKit will remove trailing slashes from URLs — if you visit `/about/`, it will respond with a redirect to `/about`. You can change this behaviour with the `trailingSlash` option, which can be one of `'never'` (the default), `'always'`, or `'ignore'`.

As with other page options, you can export this value from a `+layout.js` or a `+layout.server.js` and it will apply to all child pages. You can also export the configuration from `+server.js` files.

```js
/// file: src/routes/+layout.js
export const trailingSlash = 'always';
```

This option also affects [prerendering](#prerender). If `trailingSlash` is `always`, a route like `/about` will result in an `about/index.html` file, otherwise it will create `about.html`, mirroring static webserver conventions.

> Ignoring trailing slashes is not recommended — the semantics of relative paths differ between the two cases (`./y` from `/x` is `/y`, but from `/x/` is `/x/y`), and `/x` and `/x/` are treated as separate URLs which is harmful to SEO.

## config

With the concept of [adapters](adapters), SvelteKit is able to run on a variety of platforms. Each of these might have specific configuration to further tweak the deployment — for example on Vercel you could choose to deploy some parts of your app on the edge and others on serverless environments.

`config` is an object with key-value pairs at the top level. Beyond that, the concrete shape is dependent on the adapter you're using. Every adapter should provide a `Config` interface to import for type safety. Consult the documentation of your adapter for more information.

```js
// @filename: ambient.d.ts
declare module 'some-adapter' {
	export interface Config { runtime: string }
}

// @filename: index.js
// ---cut---
/// file: src/routes/+page.js
/** @type {import('some-adapter').Config} */
export const config = {
	runtime: 'edge'
};
```

`config` objects are merged at the top level (but _not_ deeper levels). This means you don't need to repeat all the values in a `+page.js` if you want to only override some of the values in the upper `+layout.js`. For example this layout configuration...

```js
/// file: src/routes/+layout.js
export const config = {
	runtime: 'edge',
	regions: 'all',
	foo: {
		bar: true
	}
}
```

...is overridden by this page configuration...

```js
/// file: src/routes/+page.js
export const config = {
	regions: ['us1', 'us2'],
	foo: {
		baz: true
	}
}
```

...which results in the config value `{ runtime: 'edge', regions: ['us1', 'us2'], foo: { baz: true } }` for that page.

## Further reading

- [Tutorial: Page options](https://learn.svelte.dev/tutorial/page-options)
