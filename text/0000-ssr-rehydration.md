---
title: SSR Rehydration
status: DRAFTED
created_at: 2021-08-31
updated_at: 2021-08-31
pr: (leave this empty until the PR is created)
---

# SSR Rehydration

## Summary

Whereas server-side rendering (SSR) allows a web application to be rendered into HTML on the server, rehydration involves receiving that HTML on the client and reusing the resulting DOM during initial client-side rendering (CSR).

This document proposes an approach to supporting SSR rehydration in LWC.

## Problem description

### Motivation

The motivations for SSR are [well documented](https://medium.com/walmartglobaltech/the-benefits-of-server-side-rendering-over-client-side-rendering-5d07ff2cefe8) and can be boiled down to the following two points:

- SSR allows search engines to easily index URLs that would otherwise be rendered on the client as part of a web application. This results in improved [SEO](https://developers.google.com/search/docs/beginner/seo-starter-guide), with all the associated benefits.

- SSR can improve the perceived startup performance of a web application by reducing the time to [First Contentful Paint](https://web.dev/first-contentful-paint/). Optimizing perceived performance can have measurable impact on [conversion rates](https://www.cloudflare.com/learning/performance/more/website-performance-conversion-rates/), [user satisfaction](https://www.ericsson.com/en/press-releases/2016/2/streaming-delays-mentally-taxing-for-smartphone-users-ericsson-mobility-report), and [revenue](https://www.dareboost.com/en/webperf-impacts).

However, rendering on the server can itself introduce performance issues, working against the original intent. Namely, if DOM that was generated from SSR HTML is not preserved during the initial CSR, this will result in potentially expensive DOM manipulation.

The aim of rehydration is to avoid the client-side performance issues of a naive SSR implementation, reusing original DOM wherever possible.

### The Happy Path

With regard to SSR, there exists a happy path (Fig 1a) where the rendering output for a given request is identical on both the server and the client; identical VDOM is generated for the entire document in both environments.

Depending upon the actual implementation, exceptions to this happy path include:

- **Exception A:** One or more components are never rendered on the server (Fig 1b). Instead, a placeholder is rendered on the server, and later replaced on the client during CSR/hydration.
  - **Example:** The page contains a cart icon with a dynamic number of items in cart.
  - **Example:** The page contains a user avatar.
- **Exception B:** One or more components are rendered only on the server (Fig 1c). No JS is sent to the client for these components and the original SSR'd HTML remains untouched by hydration and later client-side renders.
  - **Example:** The page contains a footer that never changes.
  - **Example:** The landing page is mostly static and is cached on a CDN.
- **Exception C:** Render-critical state is changed between the time of SSR and CSR rehydration, causing an unexpected mismatch (Fig 1d).
  - **Example:** A product price is updated after SSR but before rehydration on the client.
  - **Example:** The CDN delivers a stale version of the landing page, which was updated after the SSR HTML was en route to the client.

<figure>
  <img src="../assets/0000-ssr-rehydration-fig1.png" alt="Fig. 1">
  <figcaption align = "center">
    <b>Fig.1: the happy path and its exceptions</b>
  </figcaption>
</figure>

## Design

Several pieces need to come together to unlock rehydration. The general flow is illustrated below in figure 2.

<figure>
  <img src="../assets/0000-ssr-rehydration-fig2.png" alt="Fig. 2" width="60%">
  <figcaption align = "center">
    <b>Fig.2: the proposed implementation</b>
  </figcaption>
</figure><br /><br />

In roughly chronological order with respect to a single page request, the following steps shall be taken as part of rehydration:

- As part of SSR, serialize render-critical state and bake it into the HTML response returned from the server. For each `LightningElement`, attach corresponding state to its root DOM node as `data-ssr-state`.

- Rather than calling `LWC#createElement`, required that `LWC#hydrateElement` is called to mount the root Lightning Web Component on the client.

- Generate server-side VDOM from the SSR HTML on the client.

- As the initial client-side render progresses, extract state from `data-ssr-state` attributes, instantiate `LightningElements` with extracted state, and generate client-side VDOM.

- Patch server-side VDOM with client-side VDOM.
  
  - Where `data-ssr-preserve` is detected, extract the server-side VDOM subtree and store for subsequent client-side renders.
  
  - Attach event listeners and non-serializable props.
  
  - Resolve unexpected inconsistencies in favor of the client-side VDOM, addressing [Exception C to the happy path](#the-happy-path). With the exception of placeholder components, this case should be rare, since the first CSR will rely on state baked into SSR as `data-ssr-state`.

### `hydrateElement`

In a typical LWC web application, a developer might mount their app using the following pattern:

```javascript
import { createElement } from 'lwc';
import MyApp from './my-app';

document
  .querySelector("#root")
  .appendChild(createElement('my-app', { is: MyApp }));
```

After the proposed change, if a developer wants to take advantage of rehydration, a developer would instead do:

```javascript
import { hydrateElement } from 'lwc';
import MyApp from './my-app';

hydrateElement(MyApp, document.querySelector('#root')); 
```

### Server-only Component Support

Addressing [Exception B to the happy path](#the-happy-path) will involve a solution roughly comparable to [React server components](https://reactjs.org/blog/2020/12/21/data-fetching-with-react-server-components.html). While the bulk of this can be implemented in user-land, at a level of abstraction above LWC, the `hydrateElement` patch implementation must support the use-case.

While the initial patch progresses, each SSR VDOM node must be checked for a `data-ssr-preserve` boolean attribute. When identified, the SSR VDOM subtree rooted at that node must be preserved for subsequent client-side renders. Frameworks built on top of LWC can utilize this feature to provide server-only components.

<details>
  <summary><b>Example:</b></summary>

An LWC-based framework could provide a new `LightningServerElement` class. Its behavior and underlying implementation could be switched out at build time, as a function of the intended runtime environment.

On the server, a `LightningServerElement` could behave identically to a `LightningElement` with one exception: it would attach a `data-ssr-preserve` attribute to its root node.

For a client build, a Babel plugin could identify any class declaration that extends `LightningServerElement` and replace the entire declaration with `NoopElement`.  That would look something like the following:

**Original Source:**

```javascript
import { api } from 'lwc';
import { LightningServerElement } from 'some-framework';
import { someSideEffect } from './my-app';

export default class Example extends LightningServerElement {
    @api message = 'Hello World!';

    onClick () {
      someSideEffect();
    }
}
```

**Server Build:**

```javascript
import { api } from 'lwc';
import { LightningServerElement } from 'some-framework';
import { someSideEffect } from './my-app';

export default class Example extends LightningServerElement {
    @api message = 'Hello World!';

    // LightningServerElement hooks into component lifecycle:
    //   connectedCallback () {
    //    this.setAttribute('data-ssr-preserve', true);
    //   }

    onClick () {
      someSideEffect();
    }
}
```

**SSR HTML:**

```html
<!-- ... -->
  <x-example data-ssr-preserve>
    <template shadowroot="open">
      Hello World!
    </template>
  </x-example>
<!-- ... -->
```

**Client Build:**

```javascript
import { api } from 'lwc';
import { NoopElement } from 'some-framework';
import { someSideEffect } from './my-app'; // <-- dead-code, removed with minification

// The entire class declaration is replaced.
export default class Example extends NoopElement {}
```

</details>

### SSR Placeholders

Implementation of placeholder components, as described in [Exception A to the happy path](#the-happy-path) should be practicable without changes to LWC, so long as unexpected subtree mismatches (Exception C) are dealt with properly.

Implementation in user-land is possible with simple runtime environment detection:

<details>
  <summary>Example:</summary>

```javascript
import { LightningElement } from 'lwc';
import { isNodeEnv } from './my-utils';
import tmplServerPlaceholder from './templateServer.html';
import tmplClient from './templateClient.html';

export default class ElementWithSSRPlaceholder extends LightningElement {
    render() {
        return this.isNodeEnv ? tmplServerPlaceholder : tmplClient;
    }
}
```
</details><br />

Alternately, if it is desirable _not_ to ship the placeholder in the client-side bundle, one could utilize a Babel transform or `@rollup/plugin-replace` to make the necessary changes at build time.

<details>
  <summary>Example:</summary>

```javascript
import { LightningElement } from 'lwc';
import { isNodeEnv } from './my-utils';
// ↶ removed from client builds during minification
import tmplServerPlaceholder from './templateServer.html';
import tmplClient from './templateClient.html';

export default class ElementWithSSRPlaceholder extends LightningElement {
    render() {
        /*
          On the server, the following line is transformed to:
            return 'server' === 'server'
          On the client, it is transformed to:
            return 'client' === 'server'
        */
        return process.env.BUILD_TARGET === 'server'
            ? tmplServerPlaceholder
            : tmplClient;
        /*
          For client builds, further minification iterations will result in:
            1. return 'client' === 'server' ? tmplServerPlaceholder : tmplClient
            2. return false ? tmplServerPlaceholder : tmplClient
            3. return tmplClient
        */
    }
}
```

</details>

## Alternatives

### Server-Only and Placeholder Components

Rather than pushing responsibility for this functionality to the next layer of abstraction, LWC could handle the entirety of this use-case natively.  For example, `@lwc/engine-server` could export a `LightningServerComponent` class that behaves as described above.

Proponents for this alternative might point to the principle of least astonishment: since frameworks like React are providing a primitive for this use-case, developers might expect to see something similar in LWC when SSR support is shipped. All necessary transforms could be baked into `@lwc/rollup-plugin`, providing an out-of-the-box solution.

The case against this approach mostly hinges on conforming to the existing scope of LWC. Supporting functionality beyond what is strictly required to unlock a workable solution introduces maintenance burden as well as risk that we might be asked to extend SSR functionality with additional helpers and APIs for other use cases. Providing the primitives for frameworks to implement their own solution aligns well with LWC's historical approach.

### Resilience to Unexpected Changes

As detailed below in the [Survey of Prior Art](#survey-of-prior-art), some web frameworks choose _not_ to be resilient to unexpected changes in state and the resulting HTML/VDOM. For example, React > 16 will no longer protect against unexpected mismatched in production mode. However, React _also_ considers complex state management out of scope for the library.

For performance reasons, Vue will also not repair mismatching HTML during rehydration. However, this is less of an issue in practice, because Vue will take a snapshot of state and ship it alongside the rendered application as `window.__INITIAL_STATE__`.

Most framework authors have indicated that non-support for rehydration resilience is due to performance reasons. We should measure the performance impact of SSR rehydration in a large LWC app to either 1) validate our design decision, or 2) point us in a new direction.

Additionally, if we determine that the performance impact is too great, a solution will need to be devised to address [Exception A to the happy path](#the-happy-path).

# Unresolved questions

- **How do we handle invocations of `customElement.define` with SSR-serialized custom elements already in the DOM?**
  + If `customElement.define` is called before `hydrateElement`, the browser will "hydrate" those elements itself, and information baked into the SSR-sourced DOM could be lost.
  + Non-deterministic ordering of lifecycle hooks or element hydration could result in undefined behavior.
  + If `customElement.define` is called after `hydrateElement`, hydration could be ignorant of the class backing `<x-element>`. This could result in an incomplete or incorrect hydration.
  + Strange interactions with shadow closed mode might be an issue.

## How we teach this

Much like the new functionality itself, the documentation for SSR rehydration is purely additive. We won't be changing the behavior or API of existing features.

Beneficial documentation may include:

- an introductory blog post, explaining the underlying implementation

- a new section in the [LWC Guide](https://lwc.dev/guide/introduction), explaining the SSR-specific APIs and demonstrating how to get SSR up and running

- an addition to the [LWC Recipes](https://recipes.lwc.dev/) or an example application utilizing SSR

# Survey of Prior Art

A handful of influential web frameworks and state management libraries were examined in relation to rehydration and adjacent concerns. The findings are summarized as follows:

![0000-ssr-rehydration-fig3.png](../assets/0000-ssr-rehydration-fig3.png)

**Notes:**

1. React 17 will feature enable Suspense on the server. Functionality is currently available through a third-party library.
2. Svelte stores information for associating DOM nodes with components in the `<head>`.
3. Vue will await async data fetch at the component level, using `serverPrefetch`.
4. Vue annotates root SSR nodes with HTML attrs denoting them as server rendered.
5. Prior to React 16, rehydration could handle any mismatch between SSR and CSR VDOM. After React 16, only text nodes can mismatch.
6. In development mode, Vue checks server-rendered DOM against client-rendered DOM. In production, this is disabled for performance reasons.
7. It is technically possible but difficult to utilize nested stores alongside SSR, and not provided out-of-the-box.
