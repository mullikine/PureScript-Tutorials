## Description
### Canvas
We're producing this result with [HTML5 Canvas](https://www.w3schools.com/html/html5_canvas.asp).

![canvas01](https://user-images.githubusercontent.com/38290734/57141313-2eeadf80-6dba-11e9-8d24-ca4d3e78e55b.png)

*I'm keeping it simple because in the next tutorial we're animating it and control it with user input.*

The canvas is very stateful and imperative. It's important that we have the ability to work in this manner in a programming language and PureScript allows us to do it safely and segregate our harmful effects from our pure code.
## Instructions
### Setup
1. Install required packages
    ```
    npx psc-package install console
    npx psc-package install web-html
    npx psc-package install canvas
    ```
1. Set up npm scripts in `package.json` for ease of use
    ```json
    "scripts": {
        "postinstall": "psc-package install",
        "build": "pulp --psc-package --watch build --to bundle.js"
    }
    ```
### Usage
1. Download and install all project dependencies
    ```
    npm install
    ```
1. Start the PureScript build process
    ```
    npm run build
    ```