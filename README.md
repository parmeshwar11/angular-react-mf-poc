# 📘 Integrating a React Remote App into an Angular Host App using Module Federation

This document outlines the step-by-step process to load a **React remote application** inside an **Angular host application** using **Webpack Module Federation**. It includes required dependencies, setup procedures, and the creation of a wrapper component to embed the React app within Angular.

---

## 🏗️ A. Host Application (Angular)

### 1. ✅ Install Required Dependencies

```bash
npm install @angular-architects/module-federation --save
npm install react react-dom --save
npm install @types/react @types/react-dom --save-dev
````

> ⚠️ **Note:** Ensure that `react`, `react-dom`, `@types/react`, and `@types/react-dom` versions match in both the host and remote applications.

---

### 2. ⚙️ Configure Module Federation

In `webpack.config.js` or `webpack.config.ts`, configure the federation setup:

```js
const ModuleFederationPlugin = require("webpack/lib/container/ModuleFederationPlugin");
const mf = require("@angular-architects/module-federation/webpack");
const path = require("path");
const share = mf.share;

const sharedMappings = new mf.SharedMappings();
sharedMappings.register(
  path.join(__dirname, 'tsconfig.json'),
  [/* mapped paths to share */]
);

module.exports = {
  output: {
    uniqueName: "host",
    publicPath: "auto"
  },
  optimization: {
    runtimeChunk: false
  },
  resolve: {
    alias: {
      ...sharedMappings.getAliases(),
    }
  },
  plugins: [
    new ModuleFederationPlugin({
      name: "host",
      filename: "remoteEntry.js",
      exposes: {
        './Component': './src/app/app.component.ts',
      },
      remotes: {
        reactRemote: "reactRemote@http://localhost:3001/remoteEntry.js",
      },
      shared: share({
        "@angular/core": { singleton: true, strictVersion: true, requiredVersion: 'auto' },
        "@angular/common": { singleton: true, strictVersion: true, requiredVersion: 'auto' },
        "@angular/common/http": { singleton: true, strictVersion: true, requiredVersion: 'auto' },
        "@angular/router": { singleton: true, strictVersion: true, requiredVersion: 'auto' },
        react: { singleton: true, eager: true, requiredVersion: '^18.2.0' },
        "react-dom": { singleton: true, eager: true, requiredVersion: '^18.2.0' },
        ...sharedMappings.getDescriptors()
      })
    }),
    sharedMappings.getPlugin()
  ],
};
```

---

### 3. 🧩 Create Wrapper Component to Load React Remote App

#### `react-wrapper.component.ts`

```ts
import { Component, ElementRef, OnDestroy, OnInit, ViewChild } from '@angular/core';
import { loadRemoteModule } from '@angular-architects/module-federation';

@Component({
  selector: 'app-react-wrapper',
  template: '<div #reactRoot></div>',
})
export class ReactWrapperComponent implements OnInit, OnDestroy {
  @ViewChild('reactRoot', { static: true }) reactRoot!: ElementRef<HTMLDivElement>;

  private reactRootElement: HTMLDivElement | null = null;
  private React: typeof import('react') | null = null;
  private root: ReturnType<((typeof import('react-dom/client'))['createRoot'])> | null = null;

  ngOnInit(): void {
    void this.loadReactComponent();
  }

  private async loadReactComponent(): Promise<void> {
    this.reactRootElement = this.reactRoot.nativeElement;
    try {
      const remoteModule = await loadRemoteModule({
        type: 'script',
        remoteEntry: 'http://localhost:3001/remoteEntry.js',
        remoteName: 'reactRemote',
        exposedModule: './App'
      });

      this.React = await import('react');
      const ReactDOMClient = await import('react-dom/client');

      if (this.React && this.reactRootElement) {
        this.root = ReactDOMClient.createRoot(this.reactRootElement);
        this.root.render(this.React.createElement(remoteModule.default));
      }
    } catch (error) {
      console.error('Error loading remote React component:', error);
    }
  }

  ngOnDestroy(): void {
    if (this.root) {
      this.root.unmount();
    }
  }
}
```

---

### 4. 📦 Update `angular.json` and Enable Custom Webpack

Ensure the `angular.json` contains:

```json
"build": {
  "builder": "ngx-build-plus:browser",
  "options": {
    ...
    "extraWebpackConfig": "webpack.config.js",
    "commonChunk": false
  }
}
```

---

### 5. 🚦 Update App Routing Module

#### `app-routing.module.ts`

```ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { ReactWrapperComponent } from './react-wrapper/react-wrapper.component';

const routes: Routes = [
  {
    path: 'react-remote',
    component: ReactWrapperComponent
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
})
export class AppRoutingModule { }
```

---

## 🏗️ B. Remote Application (React)

### 1. ✅ Install Required Dependencies

```bash
npm install @babel/core @babel/preset-env @babel/preset-react \
babel-loader html-webpack-plugin react react-dom react-router-dom \
web-vitals webpack webpack-cli webpack-dev-server

npm install --save-dev css-loader style-loader
```

---

### 2. ⚙️ Webpack Configuration

#### `webpack.config.js`

```js
const { ModuleFederationPlugin } = require('webpack').container;
const HtmlWebpackPlugin = require('html-webpack-plugin');
const path = require('path');

module.exports = {
  entry: './src/index.js',
  mode: 'development',
  devServer: {
    port: 3001,
    static: path.join(__dirname, 'dist'),
    headers: {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "GET, POST, PUT, DELETE, PATCH, OPTIONS",
      "Access-Control-Allow-Headers": "X-Requested-With, content-type, Authorization"
    },
    hot: true,
  },
  output: {
    uniqueName: "host",
    publicPath: "auto"
  },
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        loader: 'babel-loader',
        exclude: /node_modules/,
        options: {
          presets: ['@babel/preset-react'],
        },
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader'],
      },
    ],
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'reactRemote',
      filename: 'remoteEntry.js',
      exposes: {
        './App': './src/App',
      },
      shared: {
        react: { singleton: true, eager: true, requiredVersion: '^18.2.0' },
        'react-dom': { singleton: true, eager: true, requiredVersion: '^18.2.0' },
      },
    }),
    new HtmlWebpackPlugin({
      template: './public/index.html',
    }),
  ],
};
```

---

## ✅ Conclusion

You now have a working integration of a React remote app inside an Angular host app using Module Federation! 🎉
Navigate to `/react-remote` in your Angular app to see the React component rendered inside Angular.
