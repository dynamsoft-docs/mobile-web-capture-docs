---
layout: default-layout
needAutoGenerateSidebar: true
needGenerateH3Content: true
noTitleIndex: true
title: Mobile Web Capture - Creating HelloWorld
keywords: Documentation, Mobile Web Capture, Creating HelloWorld
breadcrumbText: Creating HelloWorld
description: Mobile Web Capture Documentation Creating HelloWorld
permalink: /gettingstarted/helloworld.html
---

# Creating HelloWorld

In this section, we’ll break down and show all the steps needed to build the HelloWorld solution in a web page.

- Check out [HelloWorld](https://dynamsoft.github.io/mobile-web-capture/samples/hello-world/hello-world/) online

We’ll build on this skeleton page:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Mobile Web Capture - HelloWorld</title>
</head>
<body>
</body>
<script type="module">
// Write your code here
</script>
</html>
```

## Adding the dependency

This sample is using a CDN to include the SDKs. Please refer to [Adding the dependency - Use a CDN]({{ site.gettingstarted }}add_dependency.html#use-a-cdn).

If you would like to host the resources files on your own server. Please refer to [Adding the dependency - Host yourself]({{ site.gettingstarted }}add_dependency.html#host-yourself).

## Define necessary HTML elements

For HelloWorld, we define below elements.

- Container to hold the viewer

```html
<div id="container"></div>
```

- Restore button and `img` element for displaying the result image

```html
<div id="imageContainer">
    <div id="restore">Restore</div>
    <span>Original Image:</span>
    <img id="original">
    <span>Normalized Image:</span>
    <img id="normalized">
</div>
```

## Link CSS to HTML

`ddv.css` is the necessary css file which defines the viewer style of Dynamsoft Document Viewer.
`index.css` defines the style of elements which is in Helloworld.

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/dynamsoft-document-viewer@1.1.0/dist/ddv.css">
<link rel="stylesheet" href="./index.css">
```

`index.css` content:

```css
html,body {
    width: 100%;
    height: 100%;
    margin:0;
    padding:0;
    overscroll-behavior-y: none;
    overflow: hidden;
}

#container {
    width: 100%;
    height: 100%;
}

#imageContainer {
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: space-around;
    box-sizing: border-box;
    align-items: center;
    flex-direction: column;
    padding: 10px 0px;
}

#imageContainer img {
    width: 80%;
    height: 40%;
    object-fit: contain;
    border:none;
}

#restore {
    display: flex;
    width: 80px;
    height: 40px;
    align-items: center;
    background: #fe8e14;
    justify-content: center;
    color: white;
    user-select: none;
    cursor: pointer;
}
```

## Related SDK initialization

```javascript
//Preloads the Document Normalizer module.
Dynamsoft.Core.CoreModule.loadWasm(["DDN"]);
//Preloads the Document Viewer module.
Dynamsoft.DDV.Core.loadWasm();

// Initialize DDN
await Dynamsoft.License.LicenseManager.initLicense(
    "DLS2eyJoYW5kc2hha2VDb2RlIjoiMjAwMDAxLTEwMjQ5NjE5NyJ9",
    true
);
// Initialize DDV
await Dynamsoft.DDV.Core.init();
```

## Configure document boundaries function

- Step one: The related configuration code is packaged in `utils.js`, so it should be imported.

    ```javascript
    import { isMobile, initDocDetectModule } from "./utils.js";
    ```

- Step two: Call the following function.

    ```javascript
    await initDocDetectModule(Dynamsoft.DDV, Dynamsoft.CVR);
    ```

`utils.js` content:

```javascript
export function isMobile(){
    return "ontouchstart" in document.documentElement;
}

export async function initDocDetectModule(DDV, CVR) {
    const router = await CVR.CaptureVisionRouter.createInstance();

    class DDNNormalizeHandler extends DDV.DocumentDetect {
        async detect(image, config) {
            if (!router) {
                return Promise.resolve({
                    success: false
                });
            };
    
            let width = image.width;
            let height = image.height;
            let ratio = 1;
            let data;
    
            if (height > 720) {
                ratio = height / 720;
                height = 720;
                width = Math.floor(width / ratio);
                data = compress(image.data, image.width, image.height, width, height);
            } else {
                data = image.data.slice(0);
            }
    
    
            // Define DSImage according to the usage of DDN
            const DSImage = {
                bytes: new Uint8Array(data),
                width,
                height,
                stride: width * 4, //RGBA
                format: 10 // IPF_ABGR_8888
            };
    
            // Use DDN normalized module
            const results = await router.capture(DSImage, 'detect-document-boundaries');
    
            // Filter the results and generate corresponding return values
            if (results.items.length <= 0) {
                return Promise.resolve({
                    success: false
                });
            };
    
            const quad = [];
            results.items[0].location.points.forEach((p) => {
                quad.push([p.x * ratio, p.y * ratio]);
            });
    
            const detectResult = this.processDetectResult({
                location: quad,
                width: image.width,
                height: image.height,
                config
            });
    
            return Promise.resolve(detectResult);
        }
    }
  
    DDV.setProcessingHandler('documentBoundariesDetect', new DDNNormalizeHandler())
}

function compress(
    imageData,
    imageWidth,
    imageHeight,
    newWidth,
    newHeight,
) {
    let source = null;
    try {
        source = new Uint8ClampedArray(imageData);
    } catch (error) {
        source = new Uint8Array(imageData);
    }
  
    const scaleW = newWidth / imageWidth;
    const scaleH = newHeight / imageHeight;
    const targetSize = newWidth * newHeight * 4;
    const targetMemory = new ArrayBuffer(targetSize);
    let distData = null;
  
    try {
        distData = new Uint8ClampedArray(targetMemory, 0, targetSize);
    } catch (error) {
        distData = new Uint8Array(targetMemory, 0, targetSize);
    }
  
    const filter = (distCol, distRow) => {
        const srcCol = Math.min(imageWidth - 1, distCol / scaleW);
        const srcRow = Math.min(imageHeight - 1, distRow / scaleH);
        const intCol = Math.floor(srcCol);
        const intRow = Math.floor(srcRow);
  
        let distI = (distRow * newWidth) + distCol;
        let srcI = (intRow * imageWidth) + intCol;
  
        distI *= 4;
        srcI *= 4;
  
        for (let j = 0; j <= 3; j += 1) {
            distData[distI + j] = source[srcI + j];
        }
    };
  
    for (let col = 0; col < newWidth; col += 1) {
        for (let row = 0; row < newHeight; row += 1) {
            filter(col, row);
        }
    }
  
    return distData;
}
```

## Create a capture viewer

```javascript
const captureViewer = new Dynamsoft.DDV.CaptureViewer({
    container: "container",
    viewerConfig: {
        acceptedPolygonConfidence: 60,
        enableAutoCapture: true,
        enableAutoDetect: true
    }
});
// Play video stream in 1080P
captureViewer.play({ 
    resolution: [1920,1080],
});
```

## Display the result image

Use the capture event to obtain the result image.

```javascript
captureViewer.on("captured", async (e) => {
    const pageData =  await captureViewer.currentDocument.getPageData(e.pageUid);
    //Original image
    document.getElementById("original").src = URL.createObjectURL(pageData.raw.data); 
    // Normalized image
    document.getElementById("normalized").src = URL.createObjectURL(pageData.display.data); 
    // Stop video stream and hide capture viewer's container
    captureViewer.stop();
    document.getElementById("container").style.display = "none";
});
```

For now, we finish the main workflow for HelloWorld, can add the restore function to capture new image additionally.

```javascript
document.getElementById("restore").onclick = () => {
    captureViewer.currentDocument.deleteAllPages();
    captureViewer.play();
    document.getElementById("container").style.display = "";
};
```

## Review the complete code

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Mobile Web Capture - HelloWorld</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/dynamsoft-document-viewer@1.1.0/dist/ddv.css">
    <link rel="stylesheet" href="./index.css">
</head>
<body>
    <div id="container"></div>
    <div id="imageContainer">
        <div id="restore">Restore</div>
        <span>Original Image:</span>
        <img id="original">
        <span>Normalized Image:</span>
        <img id="normalized">
    </div>
</body>
<script src="https://cdn.jsdelivr.net/npm/dynamsoft-document-viewer@1.1.0/dist/ddv.js"></script>
<script src="https://cdn.jsdelivr.net/npm/dynamsoft-core@3.0.30/dist/core.js"></script>
<script src="https://cdn.jsdelivr.net/npm/dynamsoft-license@3.0.20/dist/license.js"></script>
<script src="https://cdn.jsdelivr.net/npm/dynamsoft-document-normalizer@2.0.20/dist/ddn.js"></script>
<script src="https://cdn.jsdelivr.net/npm/dynamsoft-capture-vision-router@2.0.30/dist/cvr.js"></script>
<script type="module">
    import { isMobile, initDocDetectModule } from "./utils.js";

    (async () => {
        //Preloads the Document Normalizer module.
        Dynamsoft.Core.CoreModule.loadWasm(["DDN"]);
        //Preloads the Document Viewer module.
        Dynamsoft.DDV.Core.loadWasm();

        // Initialize DDN
        await Dynamsoft.License.LicenseManager.initLicense(
            "DLS2eyJoYW5kc2hha2VDb2RlIjoiMjAwMDAxLTEwMjQ5NjE5NyJ9",
            true
        );
        // Initialize DDV
        await Dynamsoft.DDV.Core.init();

        // Configure document boundaries function
        await initDocDetectModule(Dynamsoft.DDV, Dynamsoft.CVR);

        //Create a capture viewer
        const captureViewer = new Dynamsoft.DDV.CaptureViewer({
            container: "container",
            viewerConfig: {
                acceptedPolygonConfidence: 60,
                enableAutoCapture: true,
                enableAutoDetect: true
            }
        });
        // Play video stream in 1080P
        captureViewer.play({ 
            resolution: [1920,1080],
        });

        // Display the result image
        captureViewer.on("captured", async (e) => {
            // Stop video stream and hide capture viewer's container
            captureViewer.stop();
            document.getElementById("container").style.display = "none";

            const pageData =  await captureViewer.currentDocument.getPageData(e.pageUid);
            // Original image
            document.getElementById("original").src = URL.createObjectURL(pageData.raw.data); 
            // Normalized image
            document.getElementById("normalized").src = URL.createObjectURL(pageData.display.data); 
        });

        // Restore Button function
        document.getElementById("restore").onclick = () => {
            captureViewer.currentDocument.deleteAllPages();
            captureViewer.play();
            document.getElementById("container").style.display = "";
        };
    })();
</script>
</html>
```

## Download the whole project

- Hello World - [Github](https://github.com/Dynamsoft/mobile-web-capture/tree/master/samples/hello-world/hello-world) \| [Run](https://dynamsoft.github.io/mobile-web-capture/samples/hello-world/hello-world/)
  - Angular App - [Github](https://github.com/Dynamsoft/mobile-web-capture/tree/master/samples/hello-world/hello-world-angular)
  - React App - [Github](https://github.com/Dynamsoft/mobile-web-capture/tree/master/samples/hello-world/hello-world-react)
  - Vue3 App - [Github](https://github.com/Dynamsoft/mobile-web-capture/tree/master/samples/hello-world/hello-world-vue3)

## More use cases

We provide some samples which demonstrate the popular use cases, for example, review and adjust the boundaries, edit the result images, export the result images in PDF format and so on.

Please refer to the [Use Case]({{ site.codegallery }}usecases/index.html) section.

## [Demo](https://demo.dynamsoft.com/mobile-web-capture/)