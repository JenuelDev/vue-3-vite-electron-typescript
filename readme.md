I did create a simple post about making [`vue 3 + vite + electron`](https://dev.to/brojenuel/vite-vue-3-electron-5h4o). But it does not support hot reload if their is a change in electron file you have to close and rerun everything.

In this post, I will show the full setup I am using when building an electron app. I will do it in step by step that way its a lot easier to follow the setup.

In this project we are going to use yarn.
If you don't have yarn you can install it by running.
```
npm install --global yarn
```

## STEP 1 - Create a project
Lets start by creating a folder and running init to setup our package.json.
```bash
make-dir electron-vue # make a new folder `electron-vue`. 
cd electron-vue # redirect to electron-vue folder
yarn init # run and just press enters for every question.
```

## STEP 2 - let us install ***Dev Dependencies*** that we need.
```bash
yarn add -D concurrently electron electron-builder electron-devtools-installer nodemon typescript wait-on
```
- concurrently - we are going to use this to run concurrent commands.
- electron - we will need this for developing our electron app.
- electron-builder - we need this to build our electron app.
- electron-devtools-installer - for adding vue devtools extension that we will need when develping.
- nodemon - we are going to need this to watch our files and reload electron whenever their is a change.
- typescript - we are going to need this for compiling TypeScript to JavaScript.
- wait-on - were going to need this to wait on frontend server before rendering our electron app.


## STEP 3: Setup Our Front End App (Vue3 + vite + script)
```bash
yarn create vite
```
run this and make the project name `FrontEndApp` and `select vue` and then select `typescript`.
After Doing that you should have a new Folder called `FrontEndApp`
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/smdi6mjwxf0oh9fzca98.png)

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

In your root project. Add a `.gitignore` file and add `dist`.

After doing all that you can try running `yarn build` in `/FrontEndApp` directory if build command works properly.

Like in this image, it successfully built the `FrontEndApp` to this `dist folder`.

![showing vue js dist folder](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fdow7a0d1ortwp6v0an1.png)

I think that the setup in our `FrontEndApp`.

## STEP 5: Setup Electron
Next We Are going to Setup Electron. Lets setup the typescript. By Creating a file `tsconfig.json` in the root folder.
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
