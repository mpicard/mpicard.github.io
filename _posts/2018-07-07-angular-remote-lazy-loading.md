---
layout: post
title: Lazy Loading Angular modules from a remote server
---

Today I'm going to share a technique for lazy loading Angular modules from another server. This is very useful for projects which have grown too big to be monorepo or if monorepos aren't used in your company. It allows lazy loaded modules to be developed and deployed independently.

## Step 1 - Build a remote module

Create (or migrate an existing module to) a new Angular CLI repo (`ng new $NAME --routing`). Then install `ng-packagr` and `http-server` (`yarn add -D ng-packagr http-server`). Now create an entry point for the module called EntryModule and add routing (`ng g m entry --routing`). Add an EntryComponent and something to distinguish this module and component like `<p>Module $NAME - entry works!</p>`. Change the `build` script to `ng-packagr -p package.json` and add the followig to `package.json`:

```json
"ngPackage": {
  "lib": {
    "entryFile": "src/public_api.ts"
  }
}
```

Now create that file `touch src/public_api.ts` and add the following:

```typescript
/**
 * Entry point for the remote module
 */
export * from './app/entry/entry.module';
```

> I used the `Entry` naming convension to clearly define an entry point of the module otherwise you could have a bunch of feature modules and it's not clear which one is the root unlike normal Angular apps which always have `AppModule`.

Add a root path for `EntryComponent` in `EntryRoutingModule` (ie. `routes = [{ path: '', component: EntryComponent }]`).

In `AppRoutingModule` do the following:

```typescript
import { NgModule } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';

const routes: Routes = [
  { path: '', loadChildren: './entry/entry.module#EntryModule' }
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

Now make sure you're importing `RoutingModule.forRoot([])` and `AppRoutingModule` in `AppModule`. Ensure that `EntryModule` is not imported in `AppModule` or `AppRoutingModule`. This setup will emulate the actual app that will serve this module. We'll call `yarn build` to start `ng-packagr` and while that runs we'll move onto step 2

## Step 2 - Build the core app

The `core` app will be deployed and use SystemJs to load the remote module. So create a new app (`ng new core --routing`) called `core` (or whatever you want really). Install `systemsj` (`yarn add systemjs`) and `http-server` with `cpy-cli` (`yarn add -D cpy-cli http-server`). In your `angular.json`/`.angular-cli.json` add `"node_modules/systemjs/dist/system.js",` to "scripts".

Now open `main.ts` and add the setup for SystemJs like so. Add any modules your remote module imports like ngrx, clarity, rxjs, etc.

```typescript
// ...
import * as common from '@angular/common';
import * as commonHttp from '@angular/common/http';
import * as core from '@angular/core';
import * as router from '@angular/router';
import * as rxjs from 'rxjs';
import * as rxjsOperators from 'rxjs/operators';

declare const SystemJS;
SystemJS.set('@angular/core', SystemJS.newModule(core));
SystemJS.set('@angular/common', SystemJS.newModule(common));
SystemJS.set('@angular/common/http', SystemJS.newModule(commonHttp));
SystemJS.set('@angular/router', SystemJS.newModule(router));
SystemJS.set('rxjs', SystemJS.newModule(rxjs));
SystemJS.set('rxjs/operators', SystemJS.newModule(rxjsOperators));
// ...
```

Now here's the magic part, open `AppRoutingModule` and add the following function and routes:

```typescript
/**
 * Lazy load remote bundle (AOT compatible!)
 * @param bundleName
 */
export const loadRemoteChildren = bundleName =>
  SystemJS.import(`http://127.0.0.1:8888/${bundleName}.umd.min.js`)
    .then(module => module.EntryModule)
    .catch(err => console.error(err));

const routes: Routes = [
  {
    path: '',
    children: [
      {
        path: 'lazy',
        loadChildren: () => loadRemoteChildren('$NAME') // your module name from earlier
      }
    ]
  }
]
```

## Step 3 - Serving the bundle and app

By now the `ng-packagr` build should be done and you'll have a dist folder with a bunch of stuff. The only file(s) we care about are in the `bundles` folder. These are SystemJs compatible UMD bundles. You can use `cpy-cli` to move them or just serve them from there using `http-server dist/bundles -p 8888`.

Now start the `core` dev server using `yarn start` and go to http://localhost:4200. When we inspect the network tab we'll see an initial main.js load, then a BUNDLE.umd.min.js request and your entry component should appear on the page! Neat!

If you're looking for a complete example repo with lazy loaded modules and a lazy loaded non-routing module (eg. a navbar) [click here!](http://all-loops-considered.org/2018/07/07/angular-remote-lazy-loading/platform).

Thanks for reading!
