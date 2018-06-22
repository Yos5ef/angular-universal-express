# Angunar-Universal-Express

angular 同构 express

1.下载所需要的包

`
npm install --save @angular/platform-server @nguniversal/module-map-ngfactory-loader ts-loader @nguniversal/express-engine
`
2.更改src/app/app.module.ts

`
BrowserModule.withServerTransition({ appId: 'tour-of-heroes' })
`
`
import { PLATFORM_ID, APP_ID, Inject } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

  constructor(
    @Inject(PLATFORM_ID) private platformId: Object,
    @Inject(APP_ID) private appId: string) {
    const platform = isPlatformBrowser(platformId) ?
      'in the browser' : 'on the server';
    console.log(`Running ${platform} with appId=${appId}`);
  }
`

3.添加src/app/app.server.module.ts

`
import { NgModule } from '@angular/core';
import { ServerModule } from '@angular/platform-server';
import { ModuleMapLoaderModule } from '@nguniversal/module-map-ngfactory-loader';
 
import { AppModule } from './app.module';
import { AppComponent } from './app.component';
 
@NgModule({
  imports: [
    AppModule,
    ServerModule,
    ModuleMapLoaderModule
  ],
  providers: [
    // Add universal-only providers here
  ],
  bootstrap: [ AppComponent ],
})
export class AppServerModule {}
`

4.添加sever.ts文件
`
import 'zone.js/dist/zone-node';
import 'reflect-metadata';
 
import { enableProdMode } from '@angular/core';
 
import * as express from 'express';
import { join } from 'path';
 
// Faster server renders w/ Prod mode (dev mode never needed)
enableProdMode();
 
// Express server
const app = express();
 
const PORT = process.env.PORT || 4000;
const DIST_FOLDER = join(process.cwd(), 'dist');
 
// * NOTE :: leave this as require() since this file is built Dynamically from webpack
const { AppServerModuleNgFactory, LAZY_MODULE_MAP } = require('./dist/server/main');
 
// Express Engine
import { ngExpressEngine } from '@nguniversal/express-engine';
// Import module map for lazy loading
import { provideModuleMap } from '@nguniversal/module-map-ngfactory-loader';
 
app.engine('html', ngExpressEngine({
  bootstrap: AppServerModuleNgFactory,
  providers: [
    provideModuleMap(LAZY_MODULE_MAP)
  ]
}));
 
app.set('view engine', 'html');
app.set('views', join(DIST_FOLDER, 'browser'));
 
// TODO: implement data requests securely
app.get('/api/*', (req, res) => {
  res.status(404).send('data requests are not supported');
});
 
// Server static files from /browser
app.get('*.*', express.static(join(DIST_FOLDER, 'browser')));
 
// All regular routes use the Universal engine
app.get('*', (req, res) => {
  res.render('index', { req });
});
 
// Start up the Node server
app.listen(PORT, () => {
  console.log(`Node server listening on http://localhost:${PORT}`);
});
`

5.添加src/tsconfig.server.json文件

`
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "outDir": "../out-tsc/app",
    "baseUrl": "./",
    "module": "commonjs",
    "types": []
  },
  "exclude": [
    "test.ts",
    "**/*.spec.ts"
  ],
  "angularCompilerOptions": {
    "entryModule": "app/app.server.module#AppServerModule"
  }
}
`
6.添加webpack.server.config.js文件
`
const path = require('path');
const webpack = require('webpack');
 
module.exports = {
  entry: { server: './server.ts' },
  resolve: { extensions: ['.js', '.ts'] },
  target: 'node',
  mode: 'none',
  // this makes sure we include node_modules and other 3rd party libraries
  externals: [/node_modules/],
  output: {
    path: path.join(__dirname, 'dist'),
    filename: '[name].js'
  },
  module: {
    rules: [{ test: /\.ts$/, loader: 'ts-loader' }]
  },
  plugins: [
    // Temporary Fix for issue: https://github.com/angular/angular/issues/11580
    // for 'WARNING Critical dependency: the request of a dependency is an expression'
    new webpack.ContextReplacementPlugin(
      /(.+)?angular(\\|\/)core(.+)?/,
      path.join(__dirname, 'src'), // location of your src
      {} // a map of your routes
    ),
    new webpack.ContextReplacementPlugin(
      /(.+)?express(\\|\/)(.+)?/,
      path.join(__dirname, 'src'),
      {}
    )
  ]
};
`
7.添加src/main.server.ts
`
export { AppServerModule } from './app/app.server.module';
`
8.更改angular.json文件
`
添加
"server": {
          "builder": "@angular-devkit/build-angular:server",
          "options": {
            "outputPath": "dist/server",
            "main": "src/main.server.ts",
            "tsConfig": "src/tsconfig.server.json"
          }
        }
`
在这个位置加上
`
"lint": {
          "builder": "@angular-devkit/build-angular:tslint",
          "options": {
            "tsConfig": [
              "src/tsconfig.app.json",
              "src/tsconfig.spec.json"
            ],
            "exclude": [
              "**/node_modules/**"
            ]
          }
        },
        "server": {
          "builder": "@angular-devkit/build-angular:server",
          "options": {
            "outputPath": "dist/server",
            "main": "src/main.server.ts",
            "tsConfig": "src/tsconfig.server.json"
          }
        }
`
9.更改package.json
在scripts区域里加上
`
"build:ssr": "npm run build:client-and-server-bundles && npm run webpack:server",
"serve:ssr": "node dist/server",
"build:client-and-server-bundles": "ng build --prod && ng run client:server",
"webpack:server": "webpack --config webpack.server.config.js --progress --colors"
`
注意
client位置要改成自己的项目名 比如 项目名为 project-name 就改成 project-name:ssr

10.最后运行两条命令
`
npm run build:ssr
npm run serve:ssr
`


