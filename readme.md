I did create a simple post about making [`vue 3 + vite + electron`](https://dev.to/brojenuel/vite-vue-3-electron-5h4o). But it does not support hot reload if their is a change in electron files you have to close and rerun everything.

In this post, I will show the full setup I am using when building an electron app. I will do it in step by step that way its a lot easier to follow the setup. I will paste the codes without explanation but I encourage you to take a good understanding on why I put things.

In this project we are going to use yarn.
If you don't have yarn you can install it by running.
```
npm install --global yarn
```

## STEP 1 - Create a project
Lets start by creating a folder and running init to setup our package.json.
```bash
mkdir electron-vue # make a new folder `electron-vue`. 
cd electron-vue # redirect to electron-vue folder
yarn init # run and just press enters for every question.
```

## STEP 2 - let us install ***Dev Dependencies*** that we need.

Install This packages as Dev Dependencies
```bash
yarn add -D concurrently electron electron-builder electron-devtools-installer nodemon typescript wait-on json
```
Install This packages as dependencies
```bash
yarn add electron-updater electron-store electron-log
```

- concurrently - we are going to use this to run concurrent commands.
- electron - we will need this for developing our electron app.
- electron-builder - we need this to build our electron app.
- electron-devtools-installer - for adding vue devtools extension that we will need when develping.
- nodemon - we are going to need this to watch our files and reload electron whenever their is a change.
- typescript - we are going to need this for compiling TypeScript to JavaScript.
- wait-on - were going to need this to wait on frontend server before rendering our electron app.
- electron-updater - we are going to use this for auto updating our application.
- electron-store - for storing some configurations or any data we want to save.
- electron-log - your going to need this to log some important errors or warnings in your app.
- json - for modifying our package.json file

## STEP 3: Setup Our Front End App (Vue3 + vite + script)
```bash
yarn create vite
```
run this and make the project name `FrontEndApp` and `select vue` and then select `typescript`.
After Doing that you should have a new Folder called `FrontEndApp`
![BroJenuel Screen Shot](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/smdi6mjwxf0oh9fzca98.png)

After Doing that, `cd FrontEndApp` and install the dependencies by running `yarn install`

## STEP 4: Setup FrontEndApp 
Setup This file `FrontEndApp\vite.config.ts`
```ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';

export default defineConfig(({ mode }) => ({
    build: {
        outDir: './../dist',
        chunkSizeWarningLimit: 1000,
    },
    base: mode == 'development' ? '' : './',
    plugins: [vue()],
    server: {
        port: 3000,
    },
}));
```
This will configure the ouput folder to `./../dist` whenever we run build, and also going to make the server default to 3000 this is important.

Go to This file `FrontEndApp\package.json` and edit the build script and lets add some options. `--emptyOutDir` will remove the folder whenever we build the FrontEndApp.
```bash
"scripts": {
        "dev": "vite",
        "build": "vue-tsc --noEmit &&  vite build --emptyOutDir",
        "preview": "vite preview"
    },
```

In your root project. add and edit `.gitignore`
```
dist
node_modules
```

After doing all that you can try running `yarn build` in `/FrontEndApp` directory if build command works properly.

Like in this image, it successfully built the `FrontEndApp` to this `dist folder`.

![showing vue js dist folder](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fdow7a0d1ortwp6v0an1.png)

I think that the setup in our `FrontEndApp`.

## STEP 5: Setup Electron
Lets Create a Folder `Electron` where we can put all our Electron Typescript Files.

Create a file `tsconfig.json` in the root folder with this content.
```ts
{
    "compilerOptions": {
        "target": "ESNext",
        "module": "CommonJS",
        "outDir": "./dist",
        "removeComments": true,
        "esModuleInterop": true,
        "forceConsistentCasingInFileNames": true,
        "strict": true,
        "skipLibCheck": true
    },
    "include": ["Electron/**/*.ts"]
}
```
Let us create this files.
`\Electron\main.ts`
```ts
import { app, BrowserWindow, BrowserWindowConstructorOptions, ipcMain, screen } from "electron";
import path from "path";
import { isDev } from "./config";
import { appConfig } from "./ElectronStore/Configuration";
import AppUpdater from "./AutoUpdate";

async function createWindow() {
    const { width, height } = screen.getPrimaryDisplay().workAreaSize;
    const appBounds: any = appConfig.get("setting.appBounds");
    const BrowserWindowOptions: BrowserWindowConstructorOptions = {
        width: 1200,
        minWidth: 900,
        height: 750,
        minHeight: 600,

        webPreferences: {
            preload: __dirname + "/preload.js",
            devTools: isDev,
        },
        show: false,
        alwaysOnTop: true,
        frame: true,
    };

    if (appBounds !== undefined && appBounds !== null) Object.assign(BrowserWindowOptions, appBounds);
    const mainWindow = new BrowserWindow(BrowserWindowOptions);

    // auto updated
    if (!isDev) AppUpdater();

    // and load the index.html of the app.
    // win.loadFile("index.html");
    await mainWindow.loadURL(isDev ? "http://localhost:3000" : `file://${path.join(__dirname, "./index.html")}`);

    if (appBounds !== undefined && appBounds !== null && appBounds.width > width && appBounds.height > height) mainWindow.maximize();
    else mainWindow.show();

    // this will turn off always on top after opening the application
    setTimeout(() => {
        mainWindow.setAlwaysOnTop(false);
    }, 1000);

    // Open the DevTools.
    if (isDev) {
        mainWindow.webContents.openDevTools();
    }


    ipcMain.handle('versions', () => {
        return {
            node: process.versions.chrome,
            chrome: process.versions.chrome,
            electron: process.versions.electron,
            version: app.getVersion(),
            name: app.getName(),
        };
    });
}

// This method will be called when Electron has finished
// initialization and is ready to create browser windows.
// Some APIs can only be used after this event occurs.
app.whenReady().then(async () => {
    // if dev
    if (isDev) {
        try {
            const { installExt } = await import("./installDevTool");
            await installExt();
        } catch (e) {
            console.log("Can not install extension!");
        }
    }

    createWindow();
    app.on("activate", function () {
        // On macOS it's common to re-create a window in the app when the
        // dock icon is clicked and there are no other windows open.
        if (BrowserWindow.getAllWindows().length === 0) createWindow();
    });
});

// Quit when all windows are closed, except on macOS. There, it's common
// for applications and their menu bar to stay active until the user quits
// explicitly with Cmd + Q.
app.on("window-all-closed", () => {
    if (process.platform !== "darwin") {
        app.quit();
    }
});


```
`Electron\installDevTool.ts`
```ts
import installExtension, { VUEJS_DEVTOOLS } from 'electron-devtools-installer';

export async function installExt() {
    await installExtension(VUEJS_DEVTOOLS)
        .then(() => {
            console.log('Added Extension');
        })
        .catch((err) => {
            console.log('Extension Error: ', err);
        });
}

```
`Electron\config.ts`
```ts
export const isDev = process.env.APP_IS_DEV ? true : false;
```
`Electron\AutoUpdate.ts`
```ts
import { dialog, app } from 'electron';
import { autoUpdater, UpdateInfo } from 'electron-updater';

export default () => {
    if (app.isPackaged) {
        autoUpdater.autoInstallOnAppQuit = true;
        autoUpdater.checkForUpdates();
        autoUpdater.addListener('update-downloaded', (info: UpdateInfo) => {
            dialog
                .showMessageBox({
                    title: 'Restart Believers Sword?',
                    type: 'question',
                    message: `New version "Believers Sword ${info.version}" has been successfully downloaded.`,
                    buttons: ['Yes', 'Later', 'Yes, Update'],
                    cancelId: 1,
                })
                .then(({ response }) => {
                    if (response == 0 || response == 2) {
                        autoUpdater.quitAndInstall();
                    }
                });
        });
    }
};

```
`Electron\ElectronStore\Configuration.ts`
```ts
import ElectronStore from 'electron-store';

export const appConfig = new ElectronStore({
    name: 'appConfig',
    defaults: {
        setting: {},
    },
    schema: {
        setting: {
            type: 'object',
        },
    },
});

```
`Electron/preload.ts`
```ts
import { contextBridge, ipcRenderer } from "electron";

contextBridge.exposeInMainWorld("browserWindow", {
    versions: () => ipcRenderer.invoke("versions"),
});

```

`package.json`
```json
{
  "name": "electron-vue",
  "version": "1.0.0",
  "description": "This is a sample",
  "main": "./dist/main.js",
  "license": "MIT",
  "author": "BroJenuel",
  "scripts": {
    "build": "tsc",
    "watch": "nodemon --exec yarn serve:electron",
    "start": "yarn nightly:rename && concurrently -k \"yarn serve:front\" \"yarn watch\"",
    "front:setup": "cd FrontEndApp && yarn",
    "electron:setup": "yarn",
    "setup": "yarn front:setup && yarn electron:setup",
    "serve:front": "cd FrontEndApp && yarn dev",
    "serve:electron": "set APP_IS_NIGHTLY=yes && set APP_IS_DEV=yes && yarn build && wait-on tcp:3000 && electron .",
    "build:front": "cd FrontEndApp && yarn build",
    "electron:build": "yarn build",
    "electron:builder": "electron-builder",
    "app:build": "yarn prod:rename && yarn build:front && yarn electron:build && yarn electron:builder",
    "app:build:nightly": "set APP_IS_NIGHTLY=yes && set APP_IS_DEV=no && yarn nightly:rename && yarn build:front && yarn electron:build && yarn electron:builder",
    "nightly:rename": "json -I -f package.json -e \"this.name='electron-vue'\" && json -I -f package.json -e \"this.build.productName='Electron Vue Nightly'\" && json -I -f package.json -e \"this.build.appId='com.official-electron-vue.app'\"",
    "prod:rename": "json -I -f package.json -e \"this.name='electron-vue'\" && json -I -f package.json -e \"this.build.productName='Electron Vue'\" && json -I -f package.json -e \"this.build.appId='com.official-electron-vue.app'\"",
    "electron-rebuild": "./node_modules/.bin/electron-rebuild",
    "sqlite-rebuild": "cd node_modules/better-sqlite3 && npm run build-release",
    "postinstall": "electron-builder install-app-deps",
    "up:win": "up_using_window.cmd"
  },
  "devDependencies": {
    "concurrently": "^7.6.0",
    "electron": "^22.0.0",
    "electron-builder": "^23.6.0",
    "electron-devtools-installer": "^3.2.0",
    "json": "^11.0.0",
    "nodemon": "^2.0.20",
    "typescript": "^4.9.4",
    "wait-on": "^7.0.1"
  },
  "dependencies": {
    "electron-log": "^4.4.8",
    "electron-store": "^8.1.0",
    "electron-updater": "^5.3.0"
  },
  "build": {
    "appId": "com.official-electron-vue.app",
    "productName": "Electron Vue Nightly",
    "copyright": "Copyright © 2022 ${author}",
    "publish": [
      {
        "provider": "github",
        "owner": "Bible-Projects"
      }
    ],
    "nsis": {
      "oneClick": true,
      "allowToChangeInstallationDirectory": false,
      "deleteAppDataOnUninstall": true
    },
    "files": [
      "dist/**/*"
    ],
    "extraResources": [
      "defaults/**/*"
    ],
    "directories": {
      "buildResources": "assets",
      "output": "dist_electron"
    }
  }
}

```
Next Create a Nodemon file to configure the files to watch `nodemon.json`
```json
{
    "watch": ["Electron/*"],
    "ext": "js,css,ts,json"
}
```

## STEP 6: Start Up Development Mode
You can start development mode by running. For more scripts you can review the script property in the package.json
```bash
yarn start
```
Run `yarn app:build` for production or you can run `yarn app:build:nightly` for nightly version.

when running `build commands` make sure to add github repository property on your package.json, and add your repository link as the value.

______________________

Source Code is in my Repo: https://github.com/BroJenuel/vue-3-vite-electron-typescript

___________________

Buy me coffee 😁😁😁  Thanks 💖💖

[![bmc-button](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bba6vubecqqdf2dlthjn.png)](https://www.buymeacoffee.com/BroJenuel)
 
