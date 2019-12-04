# vue-seo-friendly-spa-template
Vue.js PWA/SPA template configured for SEO (initially scaffolded with vue-cli). You can find the React version here: [react-seo-friendly-spa-template](https://github.com/based-ghost/react-seo-friendly-spa-template)

## Demo

![demo](./demo/VueSeoFriendlyDemo.gif)

## General Overview
This template reflects some of the setup I went through when experimenting with the creation of my own static front-end personal site that was to be hosted on Netlify (using GitHub as a repository/pipeline). You can find that experiment live [here](https://basedghostdevelopment.com). After playing around with this process I figured I'd build a higher-level abstraction of that project for quick re-use in the future.

## Technology Stack Overview

### vue-cli

initial scaffolding

### vue-meta

[`vue-meta`](https://github.com/nuxt/vue-meta) - plugin that allows you to manage your app's meta information, much like [`react-helmet`](https://github.com/nfl/react-helmet) does for React. However, instead of setting your data as props passed to a proprietary component, you simply export it as part of your component's data using the metaInfo property.
  
I have it configured to use a readonly objects of type `MetaInfo` (defined in `@/config/metaInfo.config.ts`) - in the component options of the component, you must declare the `metaInfo` prop, which takes the object as its value:

`metaInfo.config.ts`
```typescript
import { MetaInfo } from 'vue-meta';

export const MetaInfoAbout = Object.freeze<MetaInfo>({
  title: 'About',
  titleTemplate: '%s | VueSeoFriendlySpaTemplate',
  meta: [
    {
      property: 'og:title',
      content: 'About',
      vmid: 'og:title',
    },
    {
      property: 'og:description',
      content: 'About page description - limit of 160 characters (try for 150-155).',
      vmid: 'og:description',
    },
    {
      name: 'description',
      content: 'About page description - limit of 160 characters (try for 150-155).',
    },
  ],
});
```

`About.vue`
```typescript
<script lang="ts">
import { Component, Vue } from "vue-property-decorator";
import { MetaInfoAbout } from "@/config/metaInfo.config";

@Component({
  metaInfo: MetaInfoAbout
})
export default class About extends Vue {}
</script>
```

### vue-analytics

[`vue-analytics`](https://github.com/MatteoGabriele/vue-analytics) - Google Analytics manager plugin. The plugin isn't just a wrapper of the Google Analytics API, but provides a solution to issues that most of the time you don't want to deal with or you not even know you have to deal with.

My preferred configuration:
- id: replace with your GA tracking code
- checkDuplicatedScript: checks if the GA script has already been added/loaded before injecting - this is especially useful when you are using prerendering or SSR. You wont need to worry about removing a duplicate script tag or delaying the injection.
- router (pass the vue-router object and the plugin will handle route tracking/collection automatically).
- debug/sendHitTask: this controls when to actually fire GA collection requests - I configure it to only fire in production builds.

`main.ts`
```typescript
import Vue from 'vue';
import router from '@/router';
import VueAnalytics from 'vue-analytics';

const isProd = (process.env.NODE_ENV === 'production');

Vue.use(VueAnalytics, {
  id: 'UA-xxxxxxxxx-x',
  checkDuplicatedScript: true,
  router,
  debug: {
    enabled: !isProd,
    sendHitTask: isProd,
  },
});
```

### prerender-spa-plugin

[`prerender-spa-plugin`](https://github.com/chrisvfritz/prerender-spa-plugin) - Prerenders static HTML in a single-page application. This is a more straightforward substitue for SSR (Server Side Rendering) and the primary benefit is SEO.

Configured in the app as follows:

`main.ts` - need to fire an event after the app is mounted to let the prerenderer know when to pick up from. 
`vue.config.js` - add the `renderAfterDocument` property to the renderer (value matching the event name dispatched in `main.ts`).

<strong>Note:</strong> `renderAfterDocument` is only needed if you need to await the result of an async request and/or any of the prerendered markup relies on javascript. In the default state of this app, it is not needed, but I left it in just in case as the impact to load time is minimal. I also found in more complex applications that the `mounted()` callback fires prematurely before some of the more deeply nested child components finish rendering - making use of `$nextTick` here solves this issue.

`main.ts`
```typescript
import Vue from "vue";
import App from "@/App.vue";
import router from "@/router";

const prerenderEventName = 'prerender-event';

// In the mounted callback dispatch the event telling prerendered app when to pick up from.
// Wrap in this.$nextTick callback to ensure all components/child components have finished mounting/rendering.
new Vue({
  router,
  render: (h) => h(App),
  mounted: function () {
    this.$nextTick(function () {
      document.dispatchEvent(new Event(prerenderEventName));
    });
  }
}).$mount('#app');
```

`vue.config.js`
```javascript
const path = require("path");
const cheerio = require("cheerio");
const PrerenderSPAPlugin = require("prerender-spa-plugin");

module.exports = {
  configureWebpack: (config) => {
    if (process.env.NODE_ENV !== "production") {
      return {};
    }

    return {
      plugins: [
        new PrerenderSPAPlugin({
          staticDir: config.output.path,
          routes: ["/", "/about", "/404"],
          renderAfterDocumentEvent: 'prerender-event',
          renderer: new PrerenderSPAPlugin.PuppeteerRenderer({}),
          postProcess(context) {
            if (context.route === "/404") {
              context.outputPath = path.join(config.output.path, "/404.html");
            }

            // Remove google analytics script (will be readded by client)
            // Add data-server-rendered="true" to #app (dynamically add here rather than hard code in index.html)
            const $ = cheerio.load(context.html);
            $('head').children('script[src*="google-analytics"]').remove();
            $('#app').attr('data-server-rendered', 'true');
            context.html = $.html();

            return context;
          }
        })
      ]
    };
  }
};
```

Remainder of the configuration takes place in `vue.config.js` file where the plugin is added and configured. In the `postProcess` callback I am editing the prerendered content using `cheerio` so you can load the raw prerendered html string into a usable document and modify it using JQuery-like syntax, rather than parsing a long string and calling .replace. I found that adding a `defer` attribute to all the `script tags` is optimal since the initial render will not need any javascript loaded - this will allow the browser to load the scripts when needed and improve initial load performance.

<strong>Note:</strong> I didn't get around to do much in the way of testing, but from what I have seen it also makes sense to remove the google-analytics script from the prerendered html and allow the client to add the script later and use it immediately. Appeared to speed up the initial load times.

 ALso, I found that dynamically adding the `data-server-rendered='true'` attribute in the `postProcess` (rather than hard-coding in the index.html file) seems to work - this lets the client know that this nodes contents was rendered on server and to hydrate the html with changes, rather than render/replace.

## Scripts

### `npm install`

After cloning the repo, run this command.  This will:

- Install Node dependencies from package.json

### `npm run serve`

To start the app (development build), run this command.  This will:

- Compile the app and run on the development server

### `npm run lint`

- Run the linter (configured in the tslint.json file found in the root of this project)

### `npm run sitemap`

- This command will execute code in the sitemap-generator.js. Using the sitemapUrl parameter defined in that file (should reflect your registered domain name) a sitemap.xml is generated and persisted under the 'public' folder - this file is referenced in the robots.txt file. This uses the `sitemap-generator` package.

### `npm run build`

This script will:
 - Build release Webpack bundles
