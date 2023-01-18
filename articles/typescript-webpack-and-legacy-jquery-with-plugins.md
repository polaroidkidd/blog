# Typescript, Webpack and legacy jquery with plugins
I recently got to experience the absolute joy of trying to migrate a legacy gulp script to webpack, including moving to Typescript, while still handling all the dependencies (jquery, jquery plugins such as `own.carousel` V1 and jquery-ellipsis). I'm writing this up because most of the documentation around jquery and wepback focuses on the `webpack.ProvidePlugin`, which barely scratches the surface in the context of legacy code.


**Author: Daniel Einars**

**Date Published: 17.01.2023**



## 1. Project before the migration
The project I was working on was a website using [Magnolia](https://www.magnolia-cms.com) as their CMS. It uses the [Freemarker](https://freemarker.apache.org/) templating engine under the hood. Essentially these are super-powered `HTML` files, which give you access to the CMS content. You can still use all of the `HTML` tags you want, including the `<script>` tag.

As it is with any website, this one relied on some basic javascript for the majority of .. things (don't ask me what specifically, I don't like to think about it anymore than I have to). As such, it had some dependencies. These _are_:

1. [jquery@3.5.1](https://www.npmjs.com/package/jquery/v/3.5.1) - Release three years ago.
2. [jquery-colorscroll](https://github.com/hendriklammers/jquery-colorscroll) - Last commit was on Dec 12, 2014 and the project has been archived
3. [jquery.ellipsis](https://github.com/jjenzz/jquery.ellipsis/commits/master) - Last commit was on Nov 2, 2018.

All of the JS code we wrote depended in some form or another on jquery. At the time it was saving is a lot of headache in dealing with IE11. We never got around to introducing a propper tool-chain for it. The way I inherited it all worked as following.

We have this folder structure:
```
js
├── application.js <- Entry Point
├── modules
│   ├── scroll-animations.js
│   ├── popovers.js
│   └── ...more custom js
└── libs
├── jquery.js
├── jquery-colorscroll.js
├── jquery.ellipsis.js
└── ...more dependencies
```
All the modules were stored on a dedicated object, which in turn was stored on the window and every module had its own space on this. We did this by wrapping every module in [IIFE](https://developer.mozilla.org/en-US/docs/Glossary/IIFE). Inside this we'd assign an `initialize` function, which would .. well.. initialize the module. It's a little hard to read, so I'll give you an example.


Here's an example for a module.
```javascript
(function () {
    $(document).ready(function () {
        function init() {
            console.info("Initialize scroller stuff")
            $('.some-class').click(
                function (event) { // Note the lack of an arrow function. We couldn't use those because IE11 does not support them and we had very little as a tool-chain
                    console.info("log this statement when I click on the element which contains the class 'some-class'")
                }
            )
        }

        window.MODULE_HOLDER.Scroller = {
            init: init
        }
    })

})()
```



Here's an example `application.js` file _in its entirety (note the lack of import statements)_
```javascript
(function (){
    window.MODULE_HOLDER = window.MODULE_HOLDER || {}; // create the object to hold all our modules

    $(document).ready(function () {
        window.MODULE_HOLDER = window.MODULE_HOLDER || {}; // create the object to hold all our modules
        if(window.MODULE_HOLDER.Scroller && window.MODULE_HOLDER.Scroller.init !== undefined){ // check if a module has loaded itself
            window.MODULE_HOLDER.Scroller.init(); // run the init function
        }  
        
    })
    
})()
```


Ok, so now we have two files with IIFEs in them. `application.js` initializes modules, and modules register themselves to `window.MODULE_HOLDER` and provide the `init` function. How does it tie all together?

_**Gulp enters the stage**_

We _had_ a gulp file which basically concatenated all the js files together into a giant `IIFE`, starting with the files in the `lib` directory. This would ensure that jquery would load itself into the `window` and be ready once `application.js`, or any of the modules requested it (it also did some other stuff like concatinating all css/scss files and moving them to the right directory, stripping all `console` statements out, minifying it and placing it where Magnolia expected it to be - don't worry, it wasn't complete anarchy, more like _anarchy adjacent_).

The output would be two files.
1. application.min.js - containing all libs (jquery, jquery plugins, etc.), and all of our js code.
2. theme.min.css - containing the base theme and our additions to it.

The real project had some 30-40 modules. Some were tiny. Unfortunately for me, not all of them were tiny. Some were very large, had `var`s all over the place, referenced other objects on the global window. Basically, I couldn't be sure that if I touched something that the functionality would remain in place.

In order to mitigate this, I decided that it would probably be a good idea to move the files over to Typescript so I'd at least have some _inkling_ of a warning that I broke something when I changed something. I looked into doing this with gulp first, but soon gave up because... reasons.... OK! OK! .. I took a webpack course a while back and wanted to give my new skills a test drive.

Boy was I gonna be in for a ride...

## 2. Webpack config to replace gulp

Right, so at this point you know what the project looked like and what the output was expected to look like. Next we install all the dependencies. These were

```
...
  "dependencies": {
    "jquery": "3.5.1",
  },
  "devDependencies": {
    "@types/jquery": "3.5.16",
    "autoprefixer": "10.4.7",
    "css-loader": "^6.7.3",
    "mini-css-extract-plugin": "2.6.1",
    "postcss": "8.4.14",
    "postcss-loader": "7.0.1",
    "rimraf": "^4.0.7",
    "sass": "^1.22.10",
    "sass-loader": "13.0.2",
    "style-loader": "3.3.1",
    "ts-loader": "^9.4.2",
    "typescript": "^4.9.4",
    "webpack": "5.74.0",
    "webpack-cli": "4.10.0"
  },
...
```

I had to install jquery again, despite it existing in the `libs` folder because webpack just down right refused to work with the one imported from the lib folder. Everything else remained the same.



Next I created the `tsconfig.json`. Initially I disabled all the checks because typescript wouldn't transpile my legacy JS code, but I later found a way around that (hint, we're using the `ts-loader` which as a `transpileOnly` option). This meant that my IDE would now show me everything that Typescript thought was wrong with my JS code. There were red lines _everywhere_. This is good because I can see what I can fix in the future (but admittedly, I feel like I'm standing at the foot of Mount Everest of Legacy Javascript when I open a file like that).

Anyway, here's the `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "node",
    "lib": ["dom", "dom.iterable", "esnext", "scripthost"],
    "newLine": "lf",
    "sourceMap": true,
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "noEmit": true
  }
}
```
The only real important thing about this config is the `noEmit` flag. We do not want `tsc` to always emit js files for everything, that should be handled by webpack.


Now we could move all our files from JS to TS. Since I'm a linux fanboy I can use the `rename` command for that. It looks something like this for me

```bash
rename "s/.js$/.ts"/ **/*.js
```
Ok, so now we've converted all our files. But there remains one other thing. Webpack has this really neat feature called [tree shaking](https://webpack.js.org/guides/tree-shaking/). Essentially this omits files form the output which haven't been reached via `import/export` calls. My goal was to keep all files as similar as possible while moving them over to typescript. Rewriting all the IIFEs would lead me down a rabbit hole I would not return from. I also didn't want to mess with any tree-shaking configs, since I 

  a) Didn't quite understand which config does what
  b) Didn't have the time to read all the possible configuration options
  c) Didn't want to accidentally remove code that I needed in some obscure part of the page

This meant that I would have to import all the modules in the `application.ts` file.

Recall, this is what it looked like before
```javascript
(function (){
    window.MODULE_HOLDER = window.MODULE_HOLDER || {}; // create the object to hold all our modules

    $(document).ready(function () {
        window.MODULE_HOLDER = window.MODULE_HOLDER || {}; // create the object to hold all our modules
        if(window.MODULE_HOLDER.Scroller && window.MODULE_HOLDER.Scroller.init !== undefined){ // check if a module has loaded itself
            window.MODULE_HOLDER.Scroller.init(); // run the init function
        }  
        
    })
    
})()
```

This is what it looks like with the imports

```javascript
import "./modules/scrollEffects"
import "./modules/hoverMagic"
import "./modules/fomrHideStuff"
import "./modules/aMillionOtherModules"

(function (){
    window.MODULE_HOLDER = window.MODULE_HOLDER || {}; // create the object to hold all our modules

    $(document).ready(function () {
        window.MODULE_HOLDER = window.MODULE_HOLDER || {}; // create the object to hold all our modules
        if(window.MODULE_HOLDER.Scroller && window.MODULE_HOLDER.Scroller.init !== undefined){ // check if a module has loaded itself
            window.MODULE_HOLDER.Scroller.init(); // run the init function
        }  
        
    })
    
})()
```

Because all of these imports are IIFEs, the imports would result in the final application.js being a concatinated version of the original, with all the init calls at the end. There was nothing to tree shake :(

Ok, but why do we need to import all of this when we previously didn't, you ask accusingly...

Well, Webpack expects a single entry point for the application. Yes, yes, I know, you can also use a glob pattern to match multiple files, but I spent half a day trying to get it to work with that I just kept on hitting a wall. If you know a better way, good for you! Also, please tell me. You can reach me on Twitter via [@polaroidkidd](https://twitter.com/PolaroidKidd).

Anyway, here's ~~wonderwall~~ the webpack config. 


<sub>Speaking of which... It's a lot to take in. Instead of giving you tiny snippits which you the copy&paste together in your project, I'll give you the entire config here with all the comments done inline. You're welcome you lazy git.</sub>


```javascript

const MiniCssExtractPlugin = require("mini-css-extract-plugin"); // This plugin is very aptly named because it extracts css and minifies it
const webpack = require("webpack"); // We need webpacks ProvidePlugin, which is on the Webpack object.
const path = require("path"); // We don't really need this, but it was there when I copied this config from somewhere else.

const outputDir = path.resolve(__dirname, "target/classes/theme/js"); // This denotes the final path where webpack will place the transpiled files. In my case this is inside the target folder, which gets created by the mvn install command. Magnolia expects all the JS files in this place (at leas that's how we've configured it)


/**
 * Webpack Config
 * @param env The env variable holds information about the current build status.
 */
module.exports = (env) => {

    // I call ther webpack cli with --env production/development. This gives me some control over when I want sourcemaps to be included or not.
    const sourceMaps = env.production ? {} : {devtool: "inline-source-map"}

    return {
        entry: "./src/main/resources/theme/js/application.ts", // Currently this fil
        ...sourceMaps,
        resolve: {
            extensions: [".tsx", ".ts", ".js"],
        },
        output: {
            filename: "application.min.js", // you don't need the "min" in there, I just like it because I'm a special boy
            path: outputDir,
        },
        module: {
            // Here's where some of the magic happens. Webpack gives us this rules array, which takes objects containing a test and a use parameter. 
            // "test" is a regex which matches files. For my typescript needs I make sure that this rule applies to all files ending in "ts"
            // "use" is either an array or an object which specifies a loader. A loader is some magic library which takes one type of file and converts it to another type
            // of file. In the case of typescript, we want to use the "ts-loader" to handle all typescript files, and we use the transpileOnly option to tell the
            // loader to ignore any typescript errors. This is especially useful if you're an idiot like me and volunteer to migrate 8 years old javascript to typescript
            // without modifying the 8 year old javascript.
             
            //Ok, I didn't volunteer. The JS code kept me up at night. This had to be done.
            

            rules: [
                {
                    // Regex
                    test: /\.ts?$/,
                    use: [
                        {
                            loader: "ts-loader",
                            options: {
                                transpileOnly: true, // magic flag to enable insanity mode
                            },
                        },
                    ],
                    exclude: /node_modules/, // super important. You don't want to start transpiling all your node_moodules.
                },
                    // Here we start handling our styles. There's a loader for css files and one for scss files
                {
                    test: /\.css$/i,
                    use: [
                        MiniCssExtractPlugin.loader,
                        {
                            loader: "css-loader",
                            options: {
                                url: false,
                            },
                        },
                        {
                            loader: "postcss-loader",
                            options: {
                                postcssOptions: {
                                  // This nifty little tool prefixes your css with more css to make sure it runs on the IE11.. 
                                  // whoops, I meant safari.
                                    plugins: () => [require("autoprefixer")],
                                },
                            },
                        },
                    ],
                },
                {
                    test: /\.(s(a|c)ss)$/,
                    use: [
                        MiniCssExtractPlugin.loader,
                        {
                            loader: "css-loader",
                            options: {
                                url: false,
                            },
                        },
                        {
                            loader: "postcss-loader",
                            options: {
                                postcssOptions: {
                                    plugins: () => [require("autoprefixer")],
                                },
                            },
                        },
                        {
                            loader: "sass-loader",
                            options: {
                                sassOptions: {
                                    outputStyle: "compressed",
                                },
                            },
                        },
                    ],
                },
            ],
        },
        plugins: [
            // Uncomment to get it to work. Find out why this works in Chapter 3.
            // new webpack.ProvidePlugin({
            //     jQuery: 'jquery',
            //     $: 'jquery',
            //     'window.jQuery': 'jquery',
            //     'window.$': 'jquery',
            // }),
            // Sometimes you'll want a separate css file. In this case you can use this directive to specify
            // the location where that should go. The path is RELATIVE TO THE OUTPUT FILE.    
            new MiniCssExtractPlugin({
                filename: "../css/theme.min.css", 
            }),
        ],
        performance: {
            // Here's some configs to start alerting you if your transpiled JS becomes bigger than
            // let me do the math here
            // carry the .. one...
            // open google...
            // "how to convert from byte to something I can understand"
            // AH! If it gets larger than 5.12 Megabyte
            maxEntrypointSize: 5120000,
            maxAssetSize: 5120000,
          // Note that this is massive. NextJS starts giving you trouble after ~200kb
        },
    };
};
```

That's it folks! A half-way decent way of moving 8 year old JS to typescript without changing the JS files (too much). Now the real grind can start and you can take care of all those pesky TS errors your IDE shows you! I know, you can barely wait! 

But there's one more thing... jquery and the plugins...

> Ok, no biggie, I have them in the `libs` folder, I can just import them at the top of the `application.ts`

Well... yes and no. This is where the fun part starts (and I mean "fun" in the sense of "not very fun at all").

If you're anything like me you google "webpack jquery" and come to the webpack documentation for the [ProvidePlugin](https://webpack.js.org/plugins/provide-plugin/), which gives you the following plugin config

```javascript
new webpack.ProvidePlugin({
  $: 'jquery',
  jQuery: 'jquery',
});
```

Props to the webpack guys, the documentation is usually enough to get you started, but since jquery ~~needs to burn in hell for all eternity~~ is a nifty little lib with lots of special use-cases, I feel like the documentation is a tad lacking here.

So here's the short version of how to get it fixed.

Update `application.ts`

```javascript
import $ from  "jquery";
// init jquery to be used by all libs
// @ts-ignore
global.$ = $

// load jquery plugins
import "./libs/owl.carousel";
import "jquery-ellipsis";

// all the other imports
```

Add this to your webpack config plugin section (or uncomment it if you were brave enough to simply copy and paste mine like that weird beautiful creature you are)

```javascript
new webpack.ProvidePlugin({
    jQuery: 'jquery',
    $: 'jquery',
    'window.jQuery': 'jquery',
    'window.$': 'jquery',
})
```

Now you can run it and everything should work!

> Hey.. uhm.. you haven't told us yet what command to use for running it

Wow... you kids really do need everything spoon-fed... let me just look up where I copied my run config from....

You can add these two scripts to your `package.json` file.

```
...
"dev": "webpack  watch --config ./webpack.config.js --mode development --env development",
"prod": "webpack build --config ./webpack.config.js --mode production --env production",
...
```
The `webpack watch` will update your final output file everytime it noticed as chang in the entry or any imported files. It'll also create a source-map for you so your browser can pinpoint for you exactly where sh*t hit the fan.

That's it! For all you other maniacs who want to know why and how the `ProvidePlugin` works with jquery ~~and why the webpack documentation left me banging my head against the wall~~, continue reading at your own risk.

## 3. `$` is not defined

So, if you're like me ~~and copy & paste every snippit that looks like it could work for you until you get frustrated that it doesn't work~~ always read the documentation, you'll notice that the snippit provided by the webpack documentation (and a gazillion stackoverflow posts) is lacking. Mainly because all your code transpiles but you get this error

```
$ is not defined
```

Well, when we use webpack, it makes sure that jQuery isn't global anymore. jQuery notices that it is being used in modules (ie. if its needed they'll be an import statement for it) and therefore it doesn't globallify itself, but returns the `jQuery` function which we set to the `$` and then use in _that specific module/file only_.

This is a good thing if you're living in 2023. But back in ~~1800~~ 2015, when ~~IE11~~ we didn't have fancy arrow functions and useful module magic which made sure that things inside one file _remained_ in that file, we (ab)used the `window` object.

So how do we force this? Well, we can be blunt and simply tell it like this

```javascript

import $ from "jquery";
window.$ = $;
```

Voilà! Now we've exposed all of our code to `jquery`. ~~Shame on you~~ Congratulations! I'm sure you'll [fix this later](https://twitter.com/changelog/status/1586882941963091969?lang=en))

But wait! There's more!

The `ProvidePlugin`, well, it's kind of getting in our way. What this plugin does is make sure that any Javascript which relies on `jQuery` or `$`, it'll rewrite that code to require it as a module. _It does not make `jQuery` or `$` globally available_ <sup>This _might_ be written down in the webpack config, but I never got that far</sup>.

The issue with this is that if you have any template code (such as magnolia's Freemarker template), which rely on `jQuery` or `$` being in the global scope, well, it won't be available to them.

Things get really tricky in legacy code. Sometimes a library will  reference `jQuery` via  `window.jQuery`, which is the same, but it's not. However, we can fix that too!

Adjust the `ProvidePlugin` as follows.

```javascript
module.exports = {
    plugins: [
        new webpack.ProvidePlugin({
            jQuery: 'jquery',
            $: 'jquery',
            'window.jQuery': 'jquery',
            'window.$': 'jquery',
        })
};
```

> You Lied! There's more errors!

Well.. uhmm.. yes. Remember when I told you that the `ProvidePlugin` ~~super secretly~~ really obviously rewrites your code? Well, it's also rewriting `window.$ = $`
Update `window.$` to `global.$`.

> But Daniel, the `global` keyword is from node!

Yes, and weback runs on *drumroll* node! But also, when it encouters something like this, it'll rewrite `global` to `window`, which is what we actually wanted in the first place!


And now, we're really done!