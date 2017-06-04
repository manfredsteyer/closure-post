# Shrinking Angular-Bundles with Closure

> Big thanks to [Alex Eagle](https://twitter.com/Jakeherringbone) from the Angular Team and to [Carmen Popoviciu](https://twitter.com/CarmenPopoviciu) for reviewing this post. 

Closure is said to be the most sophisticated JavaScript compiler available today. Its advanced optimization mode goes far beyond the tree shaking capabilities of other tools and allows for shrinking bundles to a minimum. Google uses it to improve the performance of its own products, like Google Docs and even Microsoft is using it meanwhile for Office 365. However, its considered to be an expert tool and therefore difficult to configure. In addition to that, it assumes that the underlying JavaScript code has been written in a specific way.  

Currently, the Angular team is working hard on making [Angular work together with Closure](https://medium.com/@Jakeherringbone/what-angular-is-doing-with-bazel-and-closure-21f526f64a34) as well as with its build tool Bazel. There are some first examples available, e. g. the [Example](https://github.com/angular/closure-demo) [Alex Eagle](https://twitter.com/jakeherringbone) from the Angular Team created. 

This post uses the mentioned example to show how to use the closure compiler as well as the advantages it brings regarding bundle sizes. Furthermore, this post explains how to add own and existing packages to a Closure based project.

## Building a base line with the Angular CLI

In order to create a baseline for comparing Closure with a 'traditional' build task for Angular, let's create a new hello world application with the [Angular CLI](https://cli.angular.io):

```
ng new baseline
cd baseline
```

Now, let's create a production build:

```
ng build --prod
```

The generated bundles have a size of about 394K:

```
   1.460 inline.093de888567e5146835d.bundle.js
   9.360 main.0d097609144c942cc763.bundle.js
  60.845 polyfills.d90888e283bda7f009a0.bundle.js
 322.320 vendor.765bef7fc0b73d2d51d7.bundle.js
         
         393.985 Bytes
```

As the Closure sample in the next sections is directly importing the node.js polyfill and not importing any other ones, we should omit the polyfills bundle from this observation:

```
   1.460 inline.093de888567e5146835d.bundle.js
   9.360 main.0d097609144c942cc763.bundle.js
 322.320 vendor.765bef7fc0b73d2d51d7.bundle.js
         
         333.140 Bytes
```

This leafs about 333K.

After this we are installing Angular Material as well as the Animation package which Angular Material depends on:

```
npm i @angular/material --save
npm i @angular/animations --save
```

To import it into the application, its ``AppModule`` is referencing some of Angular Material's Modules:

```
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpModule } from '@angular/http';
import { AppComponent } from './app.component';

import { 
  MdButtonModule, 
  MdAutocompleteModule,
  MdCheckboxModule,
  MdDatepickerModule,
  MdCardModule,
  MdRadioModule,
  MdChipsModule,
  MdListModule,
  MdSnackBarModule,
  MdSliderModule,
  MdDialogModule,
  MdMenuModule,
  MdSidenavModule
} from '@angular/material';

@NgModule({
  imports: [
    BrowserModule,
    FormsModule,
    HttpModule,
    MdButtonModule, 
    MdAutocompleteModule,
    MdCheckboxModule,
    MdDatepickerModule,
    MdCardModule,
    MdRadioModule,
    MdChipsModule,
    MdListModule,
    MdSnackBarModule,
    MdSliderModule,
    MdDialogModule,
    MdMenuModule,
    MdSidenavModule
  ],
  declarations: [
    AppComponent
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

After recreating a production build (``ng build --prod``) we see that it grew to about 951K:

```
   1.460 inline.36030f130bb8b4d4d1e6.bundle.js
 246.959 main.bd55167e3a85bc1edaab.bundle.js
 702.496 vendor.9618c29d7fbb7af5a536.bundle.js
      950.915 Bytes
```

Now let's see how the sample application can be optimized with Closure.

## Using the Closure Compiler

To get started with Angular and the Closure Compiler, I'm using [Alex Eagle](https://twitter.com/jakeherringbone?lang=de)'s example. For this, I've [forked](https://github.com/manfredsteyer/closure-demo) the version available when writing this. 

I modified it to use npm instead of yarn and after running ``npm run build`` I got a bundle with just about 106K:

```
 105.934 bundle.js
```

This is an huge improvement over using the CLI with webpack which led to bundles with about 390K for a quite similar "Hello-Word"-App.

In order to make a fair comparison, we also need to include the packages ``@angular/forms`` and ``@angular/http`` as these packages are also imported into the CLI based app:

```
npm i @angular/http --save
npm i @angular/forms --save

```

In order to import them into the Angular App, the ``AppModule`` has to reference them:

```
import {NgModule} from '@angular/core';
import {HttpModule} from '@angular/http';
import {FormsModule} from '@angular/forms';
import {BrowserModule} from '@angular/platform-browser';
import {Basic} from './basic';

@NgModule({
  declarations: [Basic],
  bootstrap: [Basic],
  imports: [BrowserModule, FormsModule, HttpModule],
})
export class AppModule {
}
```

Unfortunately, the Closure Compiler does not respect the conventions introduced by NodeJS and so it does not automatically search the folder ``node_modules`` for these packages. Therefore, they have to be referenced explicitly within the file ``closure.conf``:

```
node_modules/@angular/forms/@angular/forms.js
--js_module_root=node_modules/@angular/forms

node_modules/@angular/http/@angular/http.js
--js_module_root=node_modules/@angular/http
```

Those lines reference both, the path where the package is found as well as its entry point which is also the whole content of the package due do the use of the [FESM15-Format](https://docs.google.com/document/d/1CZC2rcpxffTDfRDs6p1cfbmKNLA6x5O-NtkJglDaBVs/edit). 

After building everything again (``npm run build``) we are ending up with about 125K:

```
	125.134 bundle.js
```

This is still an huge improvement over using the CLI and/or Webpack.

> Please note that this example is using Closure's Advanced Mode. This mode provides better results as other known tools, but is also quite aggressive. That's why it can damage the generated bundle and so it's recommended to use this mode always together with good E2E testing.


## Using Closure with an Angular Package through the example of Angular Material

Now let's go on and import Angular Material. Once again, we have to load the following packages:

```
npm i @angular/material --save
npm i @angular/animations --save
```

And, of course, we have to import the same Angular Material Modules as before:

```
import {NgModule} from '@angular/core';
import {BrowserModule} from '@angular/platform-browser';
import {Basic} from './basic';
import { LibStarterModule } from 'stuff-lib';
import { HttpModule } from "@angular/http";
import { FormsModule } from "@angular/forms";

import { 
  MdButtonModule, 
  MdAutocompleteModule,
  MdCheckboxModule,
  MdDatepickerModule,
  MdCardModule,
  MdRadioModule,
  MdChipsModule,
  MdListModule,
  MdSnackBarModule,
  MdSliderModule,
  MdDialogModule,
  MdMenuModule,
  MdSidenavModule
} from '@angular/material';

@NgModule({
  declarations: [Basic],
  bootstrap: [Basic],
  imports: [
    BrowserModule, 
    HttpModule, 
    FormsModule, 
    MdButtonModule, 
    MdAutocompleteModule,
    MdCheckboxModule,
    MdDatepickerModule,
    MdCardModule,
    MdRadioModule,
    MdChipsModule,
    MdListModule,
    MdSnackBarModule,
    MdSliderModule,
    MdDialogModule,
    MdMenuModule,
    MdSidenavModule
  ],
})
export class AppModule {
}

```

In addition to that, it is also necessary to tell Closure about the imported modules. Therefore the ``closure.conf`` gets the following additional lines:

```
node_modules/@angular/animations/@angular/animations.js
--js_module_root=node_modules/@angular/animations

node_modules/@angular/material/@angular/material.js
--js_module_root=node_modules/@angular/material
```

Before we can create a build, we have to update the TypeScript Compiler, as the used version of the sample comes with 2.1 while the current version of Angular Material needs 2.2:

```
npm uninstall typescript --save-dev
npm install typescript@^2.2 --save-dev
```

After creating another build our bundles has about 200K:

```
199.970 bundle.js
```

This shows two things: First of all, using Closure brings a huge improvement over using the CLI and/or webpack which led to a bundle with about 951K. But this experiment also shows that even Closure is not able to fully shake off the imported but unused ``MaterialModule``.

## Creating an own Angular Package that can be used with Closure

Creating an own Angular Package that can be used together with the closure compiler is quite easy. All you need is to align with the conventions for the [Angular Package Format](https://docs.google.com/document/d/1CZC2rcpxffTDfRDs6p1cfbmKNLA6x5O-NtkJglDaBVs/edit). The most important thing to consider for Closure is providing a build using the FESM15 format. This means that you have to use EcmaScript 2015+ with EcmaScript Modules. Furthermore, you also have to "flatten" everything into one file which is used as the package's entry file. The Angular Package Format also tells us to provide our code in other formats, but here I'm just focusing on FESM15. 

For testing purposes, I've created such a package with some demo code. It's called [angular-stuff-lib](https://github.com/manfredsteyer/angular-stuff-lib) and just contains a simple Angular Module with some a ``DemoComponent``. 

```
@NgModule({
    imports: [
        CommonModule,
        FormsModule,
        HttpModule
    ],
    declarations: [
        DemoComponent
    ],
    exports: [
        DemoComponent
    ]
})
export class LibStarterModule {

    static forRoot(): ModuleWithProviders {
        return {
            ngModule: LibStarterModule,
            providers: [
                DemoService
            ]
        };
    }

}
```

Its ``forRoot`` method returns this module with a ``DemoService``. Using this method you can make sure that the service is only registered with your application's ``RootModule`` and not with other modules that import this one.

The package uses the following ``tsconfig.json``:

```
{
  "compilerOptions": {
    "module": "es2015",
    "target": "es2015",
    "outDir": "build",
    "noImplicitAny": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "declaration": true,
    "moduleResolution": "node",
    "noUnusedLocals": true,
    "types": [
      "hammerjs",
      "jasmine",
      "node"
    ],
    "lib": ["es2015", "dom"]
  },
  "files": [
    "index.ts"
  ],
  "angularCompilerOptions": {
    "strictMetadataEmit": true,
    "skipTemplateCodegen": true,
    "annotationsAs": "static fields",
    "annotateForClosureCompiler": true,
    "flatModuleOutFile": "stuff-lib.js",
    "flatModuleId": "stuff-lib"
  }
}
```

Please note several things here:

- The compilation target is EcmaScript 2015 with EcmaScript (2015) Modules.
- The destination folder is called ``built``. 
- The configuration directly points to the main file ``index.ts``
- Thanks to the property ``declaration``, the TypeScript compiler emits Type Declarations. This allows the package with its EcmaScript Bundles to be used within a TypeScript project.
- There are some options for the Angular Compiler in  ``angularCompilerOptions``:
	- ``strictMetadataEmit`` creates some meta data the Angular Compiler needs to do an AOT build for projects that use this package.
	- ``skipTemplateCodegen`` is set to true because we don't need compiled template for a library. The final project will create all of them.
	- ``annotateForClosureCompiler`` makes the Angular Compiler to generate comments with annotations Closure uses to optimize the emitted code.
	- ``flatModuleOutFile`` points to a file the Angular Compiler creates as an entry point. It can be used by tools like ``rollup`` to create a flat package (a package that just consists of one file).
	- ``flatModuleId`` contains the module name of the generated package. This is the name that can be used together with ``import`` statements.

To create the build, I'm using some npm scripts:

```
"scripts": {
  "build": "npm run clear && npm run ngc && npm run rollup && npm run copy",
  "clear": "rimraf build && rimraf dist",
  "ngc": "ngc",
  "rollup" : "rollup build/stuff-lib.js -o dist/stuff-lib.js",
  "copy": "npm run copy-package && npm run copy-metadata && npm run copy-typedef",
  "copy-typedef": "cpy build/**/*.d.ts dist --parents",
  "copy-metadata": "cpy build/**/*.metadata.json dist",
  "copy-package": "cpy dist-package.json dist/package.json"
}
```

The script ``build`` is triggering the whole procedure (``npm run build``). First of all, it clears the folders that are used for the compilation targets by leveraging ``rimraf``. Then it starts the Angular Compiler ``ngc`` which is also using the TypeScript Compiler underneath. The results of this compilation step can be found within the folder ``build`` after this. Then, the tool rollup generates the flat package file and puts it into the folder ``dist``. To make sure that all other files needed to distribute the package are located within ``dist`` the needed files are copied there. It also copies the ``package.json`` which contains necessary metadata for this package to the ``dist`` folder.

Among others, those meta data contain the following entries:

```
  "name": "stuff-lib",
  [...]
  "module": "stuff-lib.js",
  "es2015": "stuff-lib.js",
  "typings": "stuff-lib.d.ts",
```

The property ``name`` contains the name of the package and ``es2015`` is pointing to the generated flat ES2015 bundle. Normally, ``module`` would point to its ES5 counterpart. But as I'm focusing only on ES2015 here, it is pointing to the generated ES2015 bundle too. In addition to this, ``typings`` is pointing to the entry file of the emitted type definitions.

One last thing that isn't specific to closure or the Angular Module Format but important nevertheless: To avoid that Angular is installed as a sub dependency when one is downloading this package, Angular is only mentioned within ``peerDependencies``. This tells npm that everyone who is installing this package, should also install those packages.

```
"peerDependencies": {
  "@angular/core": "^4.0.0",
  "@angular/http": "^4.0.0"
},
```

To build this project, just run ``npm run build``. After this, you find the compiled package within the folder ``dist``.

## Using the own package together with Closure

To test your own Angular package with the Closure Compiler, switch to the ``dist`` folder after building it and call ``npm link``. Then switch to the root folder of the Closure project and run ``npm link stuff-lib`` which is creating a symbolic link to your package's ``dist`` folder.

After this, you have to tell Closure about the added package by inserting some lines into the file ``closure.conf``:

```
node_modules/stuff-lib/stuff-lib.js
--js_module_root=node_modules/stuff-lib
```

As mentioned above, the line with ``--js_module_root`` is pointing to the package's root directory located within ``node_modules``. This is necessary because Closure doesn't follow the conventions introduced by Node. The other line points to the flat bundle which is the entry point of the package and contains its whole contents. 

After this, just import the package's module into the ``AppModule``:

```
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { Basic } from './basic';
import { LibStarterModule } from 'stuff-lib';
import { MaterialModule } from '@angular/material';
import { HttpModule } from "@angular/http";
import { FormsModule } from "@angular/forms";

@NgModule({
  declarations: [Basic],
  bootstrap: [Basic],
  imports: [
	BrowserModule, 
	MaterialModule, 
	HttpModule, 
	FormsModule, 
	LibStarterModule.forRoot()
  ],
})
export class AppModule {
}
```

After building it with Closure, you will see that this isn't affecting the bundle size much, because the used package is just a minimal demo package. You can also assure yourself that the whole application works. To make sure the package is correctly used by Closure, you can inject its ``DemoService`` into the Root Component and use it:

```
import {Component, Injectable} from '@angular/core';
import { DemoService} from 'stuff-lib';

@Component({
  selector: 'basic',
  templateUrl: './basic.ng.html',
})
export class Basic {
  ctxProp: string;
  constructor(private demoService: DemoService) {
    this.ctxProp = `Hello World`;

    this.demoService.info = 'Hello World';
    console.log('demoService', this.demoService.doStuff());
  }
}
```

After this, rebuild the application again and run it using ``npm run serve``. This opens a demo web server on port 8080. Navigate to it and see that the specified message is written to the javascript console.

## Using Closure with a CommonJS/NodeJS Package

I've also tried to use a CommonJS/NodeJS Package with the Closure Compiler. For this, I've installed the package ``base64-js`` (``npm install base64-js --save``) which I also use for my ``angular-oauth2-oidc`` project. The easiest way to make Closure aware of such a library seems to be to directly point to it as you would point to your own files. Alex' sample is this doing for ``RxJs`` too. For this, add the following line to your ``closure.config``:

```
--js node_modules/base64-js/**.js
```

After reading this, Closure assumes that the folder mentioned contains a CommonJS-Module with the name ``base64-js``. The file ``index.js`` is assumed to be the entry point you can import by referencing the module itself using ``require('base64-js')`` or ``import ... from 'base64-js'``. If there was any other file, it could be referenced using the path ``base64-js/other-file``. To use another entry point, one can also point to the ``package.json`` using the ``--js`` flag. In this case, the entry point mentioned within this file in the property ``main`` is used.

After this, just import the needed parts of the library and use them:

```
import { fromByteArray } from 'base64-js';

[...]

this.ctxProp = fromByteArray(this.ctxProp);
```


## Dirty Hack: Patching a CommonJS/NodeJS Package to make it work with Closure


I've also tried to use the package ``sha256`` as I'm needing it for my ``angular-oauth2-oidc`` project. Unfortunately, it doesn't work with Closure. The reason can be found within the ``package.json`` of this package (``node_modules/sha256/package.json``):

```
"browser": "./lib/sha256.js",
"main": "index.js",
```

It contains two fields that specify an entry point. By convention, ``browser`` should be used for bundles that are executed within a browser and ``main`` should be used when compiling for NodeJS. Unfortunately, at the time of writing this, Closure only evaluates the ``main`` field. There are [some discussions](https://github.com/google/closure-compiler/issues/2433) about supporting further such properties too. The solution I've found was to manually patch this by setting ``main`` to ``./lib/sha256.js``:

```
"browser": "./lib/sha256.js",
"main": "./lib/sha256.js",
```

Of course, this is far away from a perfect solution and one has to make sure that the patched files are not overridden by npm. For this, one could copy it to another folder outside of ``node_modules``.

After this, just add the following lines to the file ``closure.conf``:

```
--js node_modules/sha256/**.js
--js node_modules/convert-string/**.js
--js node_modules/convert-hex/**.js

--js node_modules/sha256/package.json
--js node_modules/convert-string/package.json
--js node_modules/convert-hex/package.json
``` 

These lines add the files of sha256 as well as the files of two other packages it depends on. It addition to this, they add the ``package.json`` of those files so that Closure knows about them and uses them to infer the entry points.

Now you can import the package which is just a function and use it:

```
const sha256 = require('sha256');
[...]
this.ctxProp = fromByteArray(sha256(this.ctxProp));
``` 

## Repository for Packages compatible with Closure

As the last section showed, not every package can be used with Closure seamlessly. To help us with this, Alex Eagle from the Angular Team created the repository [angular-closure-compatibility](https://github.com/alexeagle/angular-closure-compatibility). The goal is to show which packages are supported as well as to provide examples that show how to use them in an Closure environment. 

## Conclusion

The Closure Compiler is a very powerful expert tool that can reduce bundle sizes dramatically. The Angular Package Format makes sure that (own) Angular Modules work together with it. As Closure assumes that the code is written in a specific way it can be challenging to use it together with other packages.  