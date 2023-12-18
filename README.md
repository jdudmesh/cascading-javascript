# Cascading Javascript - WTF?

## Overview

Cascading Javascript is a frontend JS library which aims to build on some of the great ideas introduced (re-introduced?) in HTMX. At the moment it's nothing more than a thought experiment ...there is no code... and it's not be taken too seriously.

## Motivation

I really like [HTMX](https://htmx.org/) but I don't love it. I think it's an important corrective in a world where massive Javascript frameworks seem to get ever more bloated, often creating new features to fix problems created by earlier iterations. It reminds us that there should be a single source of truth for application state and that belongs on the backend. It takes us back to simpler times where all the power was in the templating system and backend business logic. A time when types weren't copied across process/network boundaries. It reminds is that we should use HATEOAS.

Why don't I love HTMX? I have 2 key objections. Firstly, on aesthetic grounds I don't like stuffing markup with ever more functionality in the form of extended attributes. It didn't work with inline styles (note the foreshadowing) and that led to CSS and it didn't work with inline scripts. I think it's telling that HTMX has had to introduce yet another scripting language called [Hyperscript](https://hyperscript.org/) to push the extended attribute abstraction far enough (too far perhaps) to allow usable script in attributes.

My second objection relates to what's missing. I absolutely think having a single source of application state in the backend is a really important idea but there is also UI state and HTMX has nothing to say about that. HTMX has nothing to say about drag and drop, about validating file inputs, about client-side reactivity.

People are saying that [AlpineJS](https://alpinejs.dev/) make a great complement to HTMX and it does indeed add back much of the missing functionality. Again I object though. Again on aesthetic grounds but also becuase HTMX and AlpineJS don't always play nicely together. For example, if you bind some value in AlpineJS with `x-model` or `x-text` and that markup gets re-rendered by HTMX (post server-side validation for example) then Alpine will not pick up the new value. The 2 libraries can get out of sync.

I also have issues with some of the events like `htmx:configRequest` and `htmx:beforeRequest` which are globals. This makes it a pain to configure individual requests. Moreover they are asynchronous so it's not possible to do anything async in the event handlers.

## The solution

My wishlist is as follows:

- Marry HTMX and AlpineJS into a single library
- Move all the code out the markup and into standalone files
- Use a good reactivity library (Vue has a good standalone library)
- Use CSS selectors to connect code to markup
- Create scopes containing reactive state variables
- Type safe

To explain what I mean, it's probably simplest to show some code.

## Some example code

```html
<main class="">
  <div class="form-control relative">
    <label class="label">
      <span class="label-text">Search text</span>
    </label>
    <input
      id="search-input"
      name="search"
      class="input input-bordered w-96"
      type="text"
      placeholder="Search..."
    />
    <div
      id="search-results"
      class="absolute top-24 overflow-y-auto w-96 max-h-96 bg-base-200 border rounded-lg p-4"
      style="display: none"
    ></div>
  </div>
</main>
```

```css
      $: {
        $searchText: "";
        $isShowingSearchResults: false;

        #search-input: {
          @bind: $searchText;
        }

        #search-results: {
          @trigger: "custom:search";
          @get: "/actions/search";
          @swap: "innerHTML";

          @before: (config) => {
            config.params = $searchText
            return true
          }

          @after: () => {
            $isShowingSearchResults = true
            return true
          }

        }

        @watch: ($searchText) => {
          if (searchText.length < 3) {
            app.model.isShowingSearchResults = false
            return
          }
          dispatch("custom:search", "#search-results")
        };

        @watch: ($isShowingSearchResults) => {
          const el = document.querySelector("#search-results")
          el.style.display = $isShowingSearchResults ? "block" : "none"
        };
      }

```

What on earth is going on here?

- The `$` operator declares a variable scope (`$:{ ... }`) or an individual variable (`$searchText`)
- All variables are reactive so they can be watched and bound to HTML elements
- The `@` operator declares a directive (see below)
- Everything else is a CSS selector
- I think you could probably do some magic by changing CSS classes on things to swap different scopes in and out

In this example - a simple typeahead search form - there are 2 reactive variables `$searchText` and `$isShowingSearchResults`. The `$searchText` is bound to the input control via the `#search-input` CSS selector and the `@bind` directive. A `@watch` directive declares a Typescript function which watches the entered text. If the entered text is too short it just sets the `$isShowingSearchResults` variable to false. The watcher for this variable shows or hides the search results depending on its value.

If the search text is long enough then a custom event is triggered on the specified element (there's some magic here to bring `dispatch` and other global functions I haven't though of yet into the function's scope). Note that because it's inside a function scope `dispatch` isn't prefixed with an `@` - it's a function call not a directive. The `@trigger` directive listens for the custom event and the `@get` directive fetches the search results, swapping this markup into the inner HTML of the scoped element `#search-results`.

### Directives

These are largely copied from HTMX

- `@trigger` defines what triggers a backend request
- `@get`, `@post`, `@put`, `@delete` define the HTTP request verb and path. There can be only one request in each scope
- `@swap` defines what get's swapped (`innerHTML`, `outerHTML`, CSS selector etc)
- `@before` called before the request defined in the current scope. The function is scoped to this request alone and is `await`ed so it's possible to do async stuff here. Returning `false` cancels the request
- `@after` called after the request is complete but before the response is rendered. Allows inspection of the full response. Returning `false` will stop the response from being rendered.
- `@sse` (not shown in the example). Defines an SSE endpoint. A properly structured SSE event can swap in new markup.

## Final thoughts

This should probably never get built! It requires a parse or build step which makes it slow (slower than parsing the DOM for extended attributes?). I mean I could build it. What do you think?
