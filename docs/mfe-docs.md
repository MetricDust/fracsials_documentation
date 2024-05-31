# Implementing Angular Micro-Frontends

Angular micro-frontends involve breaking down a large Angular application into smaller, independent Angular apps, each responsible for a specific part of the overall application. This approach helps in managing complexity, improving scalability, and allowing different teams to work on different parts of the application independently.

## How to Implement Angular Micro-Frontends

1. **Module Federation**:

   - Use Webpack's Module Federation to dynamically import and load micro-frontends at runtime.
   - This allows for sharing code between different Angular apps without redeploying the entire application.

2. **Angular Elements**:

   - Convert Angular components into custom elements (web components) using Angular Elements.
   - These custom elements can be used across different micro-frontends seamlessly.

3. **Routing**:

   - The main application shell manages the routing configuration.
   - Each micro-frontend has its own set of routes, and the shell loads the appropriate micro-frontend based on the URL.

4. **Shared Services**:
   - Use shared services for common functionality and state management.
   - Ensure minimal dependencies between micro-frontends to maintain their independence.

## Step-by-Step Implementation of Micro-Frontends

### Step 1: Setup Angular Projects

```bash
# Create Angular projects for each micro-frontend and one parent/shell.
ng new shell
cd shell

ng new mfe1
cd mfe1

ng new mfe2
cd mfe2
```

### Step 2: Add module federation lib

```bash
# Add module federation library to all 3 projects
ng add @angular-architects/module-federation@14.3.10 --project shell --port 4200 --type dynamic-host #shell
ng add @angular-architects/module-federation@14.3.10 --project mfe1 --port 4201 --type remote #microfrontend1
ng add @angular-architects/module-federation@14.3.10 --project mfe2 --port 4202 --type remote #microfrontend2
```

```javascript
#Add the below code inside webpack.config.js of both mfe1 and mfe2
// mfe1\webpack.config.js
const { shareAll, withModuleFederationPlugin } = require('@angular-architects/module-federation/webpack');
module.exports = withModuleFederationPlugin({
  name: 'mfe1',
  exposes: {  // List of modules that the application will export as remote to another application
    './RemoteMfeModule': './src/app/remote-mfe1/remote-mfe1.module.ts',
  },
  shared: {
    ...shareAll({ singleton: true, strictVersion: true, requiredVersion: 'auto' }),
  },
});
```

### Step 3: Create module and routing for mfe1 and mfe2

```bash
# Create module for mfe1 and mfe2
ng generate module remoteMfe1
ng generate module remoteMfe2

# Create a component in remote-mfe1 and remote-mfe2 module
ng generate component remote-mfe1/mfe-home --module remote-mfe1
ng generate component remote-mfe2/mfe-home --module remote-mfe2
```

```typescript
# Add routing of each mfe projects(adding only for mfe1,repeat same for mfe2)
// mfe1\src\app\app-routing.module.ts
import { NgModule } from '@angular/core';
import { Route, RouterModule } from '@angular/router';
export const appRoutes: Route[] = [
  {
    path: '',
    loadChildren: () =>
      import('./remote-mfe/remote-mfe1.module').then((m) => m.RemoteMfe1Module),
  },
];
@NgModule({
  imports: [RouterModule.forRoot(appRoutes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }

------------------------------------------------------------------
// mfe1\src\app\remote-mfe\remote-mfe.module.ts
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { MfeHomeComponent } from './mfe-home/mfe-home.component';
import { Route, RouterModule } from '@angular/router';

export const remoteRoutes: Route[] = [
  { path: '', component: MfeHomeComponent },   // Add route
];
@NgModule({
  declarations: [
    MfeHomeComponent
  ],
  imports: [
    CommonModule,
    RouterModule.forChild(remoteRoutes)    // forChild
  ]
})
export class RemoteMfeModule { }
```

### Step 4: Add models and other utils files in shell project

```typescript
# Create new model - mf.model.ts
// shell\src\app\model\mf.model.ts

import { Manifest, RemoteConfig } from "@angular-architects/module-federation";

export type CustomRemoteConfig = RemoteConfig & {
    isActive: boolean;
    exposedModule: string;
    displayName?: string;
    routePath?: string;
    ngModuleName?: string;
    viaRoute?: boolean;
    withInPage?: boolean;
    componentName?: string;
 };

export type CustomManifest = Manifest<CustomRemoteConfig>;
```

```json
# update mf.manifest.json to add or remove remote MFEs from the host application.
// shell\src\assets\mf.manifest.json

{
 "mfe1":
 {
  "isActive": true,
  "remoteEntry": "http://localhost:4201/remoteEntry.js",
  "exposedModule": "./RemoteMfe1Module",
  "displayName": "RemoteMfe1",
  "routePath": "mfe1",
  "ngModuleName": "RemoteMfe1Module",
  "viaRoute": true,
  "withInPage": false
  },
  "mfe2":
  {
  "isActive": true,
  "remoteEntry": "http://localhost:4202/remoteEntry.js",
  "exposedModule": "./RemoteMfe2Module",
  "displayName": "RemoteMfe2",
  "routePath": "mfe2",
  "ngModuleName": "RemoteMfe2Module",
  "viaRoute": true,
  "withInPage": false
  }
}
```

```typescript
# update main.ts so that it only considers the remote mfe marked as isActive true in mf.manifest.json
// shell\src\main.ts

import { setManifest } from '@angular-architects/module-federation';
import { CustomManifest } from './app/model/mf.model';
const mfManifestJson = fetch('/assets/mf.manifest.json');
const parseConfig = async (mfManifest: Promise<Response>) => {
  const manifest: CustomManifest = await (await mfManifest).json();
  const filterManifest: CustomManifest = {};
  for (const key of Object.keys(manifest)) {
    const value = manifest[key];
    // check more details
    if (value.isActive === true) {
      filterManifest[key] = value;
    }
  }
  return filterManifest;
};
parseConfig(mfManifestJson)
  .then((data) => setManifest(data))
  .catch((err) => console.log(err))
  .then((_) => import('./bootstrap'))
  .catch((err) => console.log(err));

setManifest method load’s all the required metadata to fetch the Micro Frontends.
```

```typescript
# Create new mfe utils file
// shell/src/app/mfe/mfe-dynamic.routes.ts

import { getManifest, loadRemoteModule } from "@angular-architects/module-federation";
import { Routes } from "@angular/router";
import { routes } from "../app-routing.module";
import { CustomManifest } from "../model/mf.model";

export function buildRoutes(): Routes {
    const lazyRoutes = Object.entries(getManifest<CustomManifest>())
        .filter(([key, value]) => {
            return value.viaRoute === true
        })
        .map(([key, value]) => {
            return {
                path: value.routePath,
                loadChildren: () => loadRemoteModule({
                    type: 'manifest',
                    remoteName: key,
                    exposedModule: value.exposedModule
                }).then(m => m[value.ngModuleName!])
            }
        });
    const notFound = [
        {
            path: '**',
            redirectTo: ''
        }]
    // { path:'**', ...} needs to be the LAST one.
    return [...routes, ...lazyRoutes, ...notFound]
}
```

```bash
# Create new mfe service

// shell/src/app/mfe/mfe-service.service.ts

import { Injectable } from '@angular/core';
import { Router } from '@angular/router';
import { buildRoutes } from './mfe-dynamic.routes';

@Injectable({
  providedIn: 'root'
})
export class MfeServiceService {
  constructor(private router: Router) { }
  init () {
    return new Promise<void>((resolve, reject) => {
      const routes = buildRoutes();
      this.router.resetConfig(routes);
      resolve();
    })
  }
}
```

```typescript
// shell/src/app/app.module.ts

import { APP_INITIALIZER, NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';
import { LandingPageComponent } from './landing-page/landing-page.component';
import { HomeComponent } from './home/home.component';
import { MfeServiceService } from './mfe/mfe-service.service';
@NgModule({
  declarations: [AppComponent, LandingPageComponent, HomeComponent],
  imports: [BrowserModule, AppRoutingModule],
  providers: [                            // Add APP_INITIALIZER
    {
      provide: APP_INITIALIZER,
      useFactory: (mfeService: MfeServiceService) => () =>
      mfeService.init(),
      deps: [MfeServiceService],
      multi: true,
    },
  ],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

### Step 5: Create components in shell and MFEs and link components from mfe to shell

```html
# Create components in shell and Mfe 

// shell/src/app/header

ng generate component header

// header.component.html

<div>
  <ul class="navbar-nav">
    <!-- static links -->
    <li class="nav-item">
      <a class="nav-link" routerLink="/home" routerLinkActive="active">Home</a>
    </li>
    <!-- dynamic links from the mf.manifest.json -->
    <li>
      <a
        class="nav-link"
        [routerLink]="['/mfe1/contact-us']"
        routerLinkActive="active"
        >Contact Us</a
      >
    </li>
     <li>
      <a
        class="nav-link"
        [routerLink]="['/mfe2/services']"
        routerLinkActive="active"
        >Services</a
      >
    </li>
  </ul>
</div>
```
```html

// mfe1/src/app/remote-mfe1/conatact-us

ng generate component contact-us

// contact-us.component.html

<p> contact us works!! </p>


// mfe2/src/app/remote-mfe2/services

ng generate component services

// services.component.html

<p> services works!! </p>
```

```typescript
# add routing in shell and Mfes

// shell/src/app/app.module.ts

import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { HeaderComponent } from './home/home.component';


export const routes: Routes = [
  {
    path: 'header',
    component: HeaderComponent
  },
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }


// mfe1/src/app/remote-mfe1/remote-mfe1.module.ts
import { ContactUsComponent } from './contact-us/contact-us.component';

export const remoteRoutes: Route[] = [
  { path: 'contact-us', component: ContactUsComponent },   // Add route
];

// mfe2/src/app/remote-mfe2/remote-mfe2.module.ts
import { ServicesComponent } from './services/services.component';

export const remoteRoutes: Route[] = [
  { path: 'services', component: ServicesComponent },   // Add route
];
```

Note : 
1. Make sure that you are running mfe projects which you want to dynamically fetch inside shell project.
2. Make sure that you have added required packages, modules, styles and assets from MFEs in shell project which are essential to dynamically fetch MFEs.


### Expected Result : 

1. When you click on Contact Us from header, you should be able to see contents from contact-us.component.html running at localhost:4201/contact-us in shell project at localhost:4200/mfe1/contact-us.

2. When you click on Services from header, you should be able to see contents from services.component.html running at localhost:4202/services in shell project at localhost:4200/mfe2/services.


### Support

For any questions or assistance, please feel free to reach out to our development team:

- [Ashish Palankar](mailto:ashishpalankar2@gmail.com)
- [Vaibhavi Shanbhag](mailto:vaibhavishanbhag15@gmail.com)
