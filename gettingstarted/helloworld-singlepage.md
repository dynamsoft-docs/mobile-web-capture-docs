---
layout: default-layout
needAutoGenerateSidebar: true
needGenerateH3Content: true
noTitleIndex: true
title: Mobile Web Capture - Creating HelloWorld - Single Page
keywords: Documentation, Mobile Web Capture, Creating HelloWorld - Single Page
breadcrumbText: Creating HelloWorld - Single Page
description: Mobile Web Capture Documentation Creating HelloWorld - Single Page
permalink: /gettingstarted/helloworld-singlepage.html
---

# Creating HelloWorld - Single Page

In this section, we’ll break down and show all the steps needed to build the HelloWorld - Single Page solution in a web page.

- Check out [HelloWorld - Single Page](https://dynamsoft.github.io/mobile-web-capture/samples/hello-world-singlepage/hello-world/hello-world.html) online

We’ll build on this skeleton page:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Mobile Web Capture - HelloWorld - Single Page</title>
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

  ```html
  <script src="https://cdn.jsdelivr.net/npm/dynamsoft-core@3.2.10/dist/core.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/dynamsoft-license@3.2.10/dist/license.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/dynamsoft-document-normalizer@2.2.10/dist/ddn.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/dynamsoft-capture-vision-router@2.2.10/dist/cvr.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/dynamsoft-camera-enhancer@4.0.2/dist/dce.js"></script>
  ```

## Define necessary HTML elements

For HelloWorld, we define below elements.

- Container for displaying the video stream, captured image, and normalized result image.

```html
<div id="div-ui-container" style="margin-top: 10px; height: 450px"></div>
<div id="div-image-container" style="display: none; width: 100%; height: 70vh"></div>
<div id="normalized-result"></div>
```

- Start/Restart Detection, Confirm Boundary, Normalize button to get the normalized result

```html
<button id="start-detecting">Start Detecting</button>
<button id="restart-detecting" style="display: none">Restart Detecting</button>
<button id="confirm-quad-for-normalization">Confirm the Boundary</button>
<button id="normalize-with-confirmed-quad" disabled>Normalize</button><br />
<input type="checkbox" style="vertical-align: middle" id="auto-normalize" /><label style="vertical-align: middle" for="auto-normalize">Normalize Automatically</label>

```

## Define variables

```javascript
let quads = [];
let cameraEnhancer = null;
let router = null;
let items;
let layer;
let originalImage;
let imageEditorView;
let promiseCVRReady;
let frameCount = 0;

const btnStart = document.querySelector("#start-detecting");
const btnRestart = document.querySelector("#restart-detecting");
const cameraViewContainer = document.querySelector("#div-ui-container");
const normalizedImageContainer = document.querySelector("#normalized-result");
const btnEdit = document.querySelector("#confirm-quad-for-normalization");
const btnNormalize = document.querySelector("#normalize-with-confirmed-quad");
const imageEditorViewContainer = document.querySelector("#div-image-container");
const autoNormalize = document.querySelector("#auto-normalize");
```

## Related SDK initialization

```javascript
/** LICENSE ALERT - README
 * To use the library, you need to first call the method initLicense() to initialize the license using a license key string.
 */
Dynamsoft.License.LicenseManager.initLicense("DLS2eyJoYW5kc2hha2VDb2RlIjoiMjAwMDAxLTEwMjQ5NjE5NyJ9");

//Preloads the Document Normalizer module.
Dynamsoft.Core.CoreModule.loadWasm(["DDN"]);

```

## Creates a CameraEnhancer instance and prepares an ImageEditorView instance

```javascript
async function initDCE() {
      const view = await Dynamsoft.DCE.CameraView.createInstance();
      cameraEnhancer = await Dynamsoft.DCE.CameraEnhancer.createInstance(view);
      imageEditorView = await Dynamsoft.DCE.ImageEditorView.createInstance(imageEditorViewContainer);
      /* Creates an image editing layer for drawing found document boundaries. */
      layer = imageEditorView.createDrawingLayer();
      cameraViewContainer.append(view.getUIElement());
    }
```

## Creates a CaptureVisionRouter instance

```javascript
/**
 * Creates a CaptureVisionRouter instance and configure the task to detect document boundaries.
 * Also, make sure the original image is returned after it has been processed.
 */
let cvrReady = (async function initCVR() {
await initDCE();
router = await Dynamsoft.CVR.CaptureVisionRouter.createInstance();
router.setInput(cameraEnhancer);
/**
 * Sets the result types to be returned.
 * Because we need to normalize the original image later, here we set the return result type to
 * include both the quadrilateral and original image data.
 */
let newSettings = await router.getSimplifiedSettings("DetectDocumentBoundaries_Default");
newSettings.capturedResultItemTypes |= Dynamsoft.Core.EnumCapturedResultItemType.CRIT_ORIGINAL_IMAGE;
await router.updateSettings("DetectDocumentBoundaries_Default", newSettings);

/* Defines the result receiver for the task.*/
const resultReceiver = new Dynamsoft.CVR.CapturedResultReceiver();
resultReceiver.onCapturedResultReceived = handleCapturedResult;
router.addResultReceiver(resultReceiver);
})();
```

## Defines the callback function

```javascript
/**
 * Defines the callback function that is executed after each image has been processed.
 */
async function handleCapturedResult(result) {
    /* Do something with the result */
    /* Saves the image data of the current frame for subsequent image editing. */
    const originalImage = result.items.filter((item) => item.type === 1);
    originalImageData = originalImage.length && originalImage[0].imageData;
    if (!autoNormalize.checked) {
    /* why > 1? Because the "result items" always contain a result of the original image. */
    if (result.items.length > 1) {
        items = result.items;
    }
    } else if (originalImageData) {
    /** If "Normalize Automatically" is checked, the library uses the document boundaries found in consecutive
        * image frames to decide whether conditions are suitable for automatic normalization.
        */
    if (result.items.length <= 1) {
        frameCount = 0;
        return;
    }
    frameCount++;
    /**
        * In our case, we determine a good condition for "automatic normalization" to be
        * "getting document boundary detected for 30 consecutive frames".
        *
        * NOTE that this condition will not be valid should you add a CapturedResultFilter
        * with ResultDeduplication enabled.
        */
    if (frameCount === 30) {
        frameCount = 0;
        normalizedImageContainer.innerHTML = "";
        /**
        * When the condition is met, we use the document boundary found in this image frame
        * to normalize the document by setting the coordinates of the ROI (region of interest)
        * in the built-in template "NormalizeDocument_Default".
        */
        let settings = await router.getSimplifiedSettings("NormalizeDocument_Default");
        settings.roiMeasuredInPercentage = 0;
        settings.roi.points = result.items[1].location.points;
        await router.updateSettings("NormalizeDocument_Default", settings);
        /**
        * After that, executes the normalization and shows the result on the page.
        */
        let normalizeResult = await router.capture(originalImageData, "NormalizeDocument_Default");
        normalizedImageContainer.append(normalizeResult.items[0].toCanvas());
        cameraViewContainer.style.display = "none";
        btnStart.style.display = "none";
        btnRestart.style.display = "inline";
        btnEdit.disabled = true;
        await router.stopCapturing();
    }
    }
}
```
## Normalize Automatically Checkbox

```javascript
autoNormalize.addEventListener("change", () => {
    btnEdit.style.display = autoNormalize.checked ? "none" : "inline";
    btnNormalize.style.display = autoNormalize.checked ? "none" : "inline";
});
```

## Start Detecting Button

```javascript
btnStart.addEventListener("click", async () => {
    try {
    await (promiseCVRReady = promiseCVRReady || (async () => {
        await cvrReady;
        /* Starts streaming the video. */
        await cameraEnhancer.open();
        /* Uses the built-in template "DetectDocumentBoundaries_Default" to start a continuous boundary detection task. */
        await router.startCapturing("DetectDocumentBoundaries_Default");
    })());
    } catch (ex) {
    let errMsg = ex.message || ex;
    console.error(errMsg);
    alert(errMsg);
    }
})
```

## Confirm the Boundary Button

```javascript
btnEdit.addEventListener("click", async () => {
    if (!cameraEnhancer.isOpen() || items.length <= 1) return;
    /* Stops the detection task since we assume we have found a good boundary. */
    router.stopCapturing();
    /* Hides the cameraView and shows the imageEditorView. */
    cameraViewContainer.style.display = "none";
    imageEditorViewContainer.style.display = "block";
    /* Draws the image on the imageEditorView first. */
    imageEditorView.setOriginalImage(originalImageData);
    quads = [];
    /* Draws the document boundary (quad) over the image. */
    for (let i = 0; i < items.length; i++) {
    if (items[i].type === Dynamsoft.Core.EnumCapturedResultItemType.CRIT_ORIGINAL_IMAGE) continue;
    const points = items[i].location.points;
    const quad = new Dynamsoft.DCE.QuadDrawingItem({ points });
    quads.push(quad);
    layer.addDrawingItems(quads);
    }
    btnStart.style.display = "none";
    btnRestart.style.display = "inline";
    btnNormalize.disabled = false;
    btnEdit.disabled = true;
});
```

## Normalize Button

```javascript
btnNormalize.addEventListener("click", async () => {
    /* Gets the selected quadrilateral. */
    let seletedItems = imageEditorView.getSelectedDrawingItems();
    let quad;
    if (seletedItems.length) {
    quad = seletedItems[0].getQuad();
    } else {
    quad = items[1].location;
    }
    const isPointOverBoundary = (point) => {
    if (point.x < 0 ||
        point.x > originalImageData.width ||
        point.y < 0 ||
        point.y > originalImageData.height) {
        return true;
    } else {
        return false;
    }
    };
    /* Check if the points beyond the boundaries of the image. */
    if (quad.points.some(point => isPointOverBoundary(point))) {
    alert("The document boundaries extend beyond the boundaries of the image and cannot be used to normalize the document.");
    return;
    }

    /* Hides the imageEditorView. */
    imageEditorViewContainer.style.display = "none";
    /* Removes the old normalized image if any. */
    normalizedImageContainer.innerHTML = "";
    /**
    * Sets the coordinates of the ROI (region of interest)
    * in the built-in template "NormalizeDocument_Default".
    */
    let newSettings = await router.getSimplifiedSettings("NormalizeDocument_Default");
    newSettings.roiMeasuredInPercentage = 0;
    newSettings.roi.points = quad.points;
    await router.updateSettings("NormalizeDocument_Default", newSettings);
    /* Executes the normalization and shows the result on the page. */
    let normalizeResult = await router.capture(originalImageData, "NormalizeDocument_Default");
    if (normalizeResult.items[0]) {
    normalizedImageContainer.append(normalizeResult.items[0].toCanvas());
    }
    layer.clearDrawingItems();
    btnNormalize.disabled = true;
    btnEdit.disabled = true;
});
```

For now, we finish the main workflow for HelloWorld, can add the restart function to capture new image additionally.

## Restart Detecting Button

```javascript
btnRestart.addEventListener("click", async () => {
    /* Reset the UI elements and restart the detection task. */
    imageEditorViewContainer.style.display = "none";
    normalizedImageContainer.innerHTML = "";
    cameraViewContainer.style.display = "block";
    btnStart.style.display = "inline";
    btnRestart.style.display = "none";
    btnNormalize.disabled = true;
    btnEdit.disabled = false;
    layer.clearDrawingItems();

    await router.startCapturing("DetectDocumentBoundaries_Default");
})
```

## Review the complete code

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Mobile Web Capture - HelloWorld - Single Page</title>
    <script src="https://cdn.jsdelivr.net/npm/dynamsoft-core@3.2.10/dist/core.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/dynamsoft-license@3.2.10/dist/license.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/dynamsoft-document-normalizer@2.2.10/dist/ddn.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/dynamsoft-capture-vision-router@2.2.10/dist/cvr.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/dynamsoft-camera-enhancer@4.0.2/dist/dce.js"></script>
</head>
<body>
   <h1 style="font-size: 1.5em">
    Detect the boundary of a document and normalize it
  </h1>
  <button id="start-detecting">Start Detecting</button>
  <button id="restart-detecting" style="display: none">Restart Detecting</button>
  <button id="confirm-quad-for-normalization">Confirm the Boundary</button>
  <button id="normalize-with-confirmed-quad" disabled>Normalize</button><br />
  <input type="checkbox" style="vertical-align: middle" id="auto-normalize" /><label style="vertical-align: middle" for="auto-normalize">Normalize Automatically</label>
  <div id="div-ui-container" style="margin-top: 10px; height: 450px"></div>
  <div id="div-image-container" style="display: none; width: 100%; height: 70vh"></div>
  <div id="normalized-result"></div>
   <script>
    let quads = [];
    let cameraEnhancer = null;
    let router = null;
    let items;
    let layer;
    let originalImage;
    let imageEditorView;
    let promiseCVRReady;
    let frameCount = 0;

    const btnStart = document.querySelector("#start-detecting");
    const btnRestart = document.querySelector("#restart-detecting");
    const cameraViewContainer = document.querySelector("#div-ui-container");
    const normalizedImageContainer = document.querySelector("#normalized-result");
    const btnEdit = document.querySelector("#confirm-quad-for-normalization");
    const btnNormalize = document.querySelector("#normalize-with-confirmed-quad");
    const imageEditorViewContainer = document.querySelector("#div-image-container");
    const autoNormalize = document.querySelector("#auto-normalize");

    /** LICENSE ALERT - README
     * To use the library, you need to first call the method initLicense() to initialize the license using a license key string.
     */
    Dynamsoft.License.LicenseManager.initLicense("DLS2eyJoYW5kc2hha2VDb2RlIjoiMjAwMDAxLTEwMjQ5NjE5NyJ9");
    /**
     * The license "DLS2eyJoYW5kc2hha2VDb2RlIjoiMjAwMDAxLTEwMjQ5NjE5NyJ9" is a temporary license for testing good for 24 hours.
     * You can visit https://www.dynamsoft.com/customer/license/trialLicense?utm_source=github&architecture=dcv&product=ddn&package=js to get your own trial license good for 30 days.
     * LICENSE ALERT - THE END
     */

    /**
     * Preloads the `DocumentNormalizer` module, saving time in preparing for document border detection and image normalization.
     */
    Dynamsoft.Core.CoreModule.loadWasm(["DDN"]);

    /**
     * Creates a CameraEnhancer instance and prepares an ImageEditorView instance for later use.
     */
    async function initDCE() {
      const view = await Dynamsoft.DCE.CameraView.createInstance();
      cameraEnhancer = await Dynamsoft.DCE.CameraEnhancer.createInstance(view);
      imageEditorView = await Dynamsoft.DCE.ImageEditorView.createInstance(imageEditorViewContainer);
      /* Creates an image editing layer for drawing found document boundaries. */
      layer = imageEditorView.createDrawingLayer();
      cameraViewContainer.append(view.getUIElement());
    }

    /**
     * Creates a CaptureVisionRouter instance and configure the task to detect document boundaries.
     * Also, make sure the original image is returned after it has been processed.
     */
    let cvrReady = (async function initCVR() {
      await initDCE();
      router = await Dynamsoft.CVR.CaptureVisionRouter.createInstance();
      router.setInput(cameraEnhancer);
      /**
       * Sets the result types to be returned.
       * Because we need to normalize the original image later, here we set the return result type to
       * include both the quadrilateral and original image data.
       */
      let newSettings = await router.getSimplifiedSettings("DetectDocumentBoundaries_Default");
      newSettings.capturedResultItemTypes |= Dynamsoft.Core.EnumCapturedResultItemType.CRIT_ORIGINAL_IMAGE;
      await router.updateSettings("DetectDocumentBoundaries_Default", newSettings);

      /* Defines the result receiver for the task.*/
      const resultReceiver = new Dynamsoft.CVR.CapturedResultReceiver();
      resultReceiver.onCapturedResultReceived = handleCapturedResult;
      router.addResultReceiver(resultReceiver);
    })();

    /**
     * Defines the callback function that is executed after each image has been processed.
     */
    async function handleCapturedResult(result) {
      /* Do something with the result */
      /* Saves the image data of the current frame for subsequent image editing. */
      const originalImage = result.items.filter((item) => item.type === 1);
      originalImageData = originalImage.length && originalImage[0].imageData;
      if (!autoNormalize.checked) {
        /* why > 1? Because the "result items" always contain a result of the original image. */
        if (result.items.length > 1) {
          items = result.items;
        }
      } else if (originalImageData) {
        /** If "Normalize Automatically" is checked, the library uses the document boundaries found in consecutive
         * image frames to decide whether conditions are suitable for automatic normalization.
         */
        if (result.items.length <= 1) {
          frameCount = 0;
          return;
        }
        frameCount++;
        /**
         * In our case, we determine a good condition for "automatic normalization" to be
         * "getting document boundary detected for 30 consecutive frames".
         *
         * NOTE that this condition will not be valid should you add a CapturedResultFilter
         * with ResultDeduplication enabled.
         */
        if (frameCount === 30) {
          frameCount = 0;
          normalizedImageContainer.innerHTML = "";
          /**
           * When the condition is met, we use the document boundary found in this image frame
           * to normalize the document by setting the coordinates of the ROI (region of interest)
           * in the built-in template "NormalizeDocument_Default".
           */
          let settings = await router.getSimplifiedSettings("NormalizeDocument_Default");
          settings.roiMeasuredInPercentage = 0;
          settings.roi.points = result.items[1].location.points;
          await router.updateSettings("NormalizeDocument_Default", settings);
          /**
           * After that, executes the normalization and shows the result on the page.
           */
          let normalizeResult = await router.capture(originalImageData, "NormalizeDocument_Default");
          normalizedImageContainer.append(normalizeResult.items[0].toCanvas());
          cameraViewContainer.style.display = "none";
          btnStart.style.display = "none";
          btnRestart.style.display = "inline";
          btnEdit.disabled = true;
          await router.stopCapturing();
        }
      }
    }

    btnStart.addEventListener("click", async () => {
      try {
        await (promiseCVRReady = promiseCVRReady || (async () => {
          await cvrReady;
          /* Starts streaming the video. */
          await cameraEnhancer.open();
          /* Uses the built-in template "DetectDocumentBoundaries_Default" to start a continuous boundary detection task. */
          await router.startCapturing("DetectDocumentBoundaries_Default");
        })());
      } catch (ex) {
        let errMsg = ex.message || ex;
        console.error(errMsg);
        alert(errMsg);
      }
    })

    btnRestart.addEventListener("click", async () => {
      /* Reset the UI elements and restart the detection task. */
      imageEditorViewContainer.style.display = "none";
      normalizedImageContainer.innerHTML = "";
      cameraViewContainer.style.display = "block";
      btnStart.style.display = "inline";
      btnRestart.style.display = "none";
      btnNormalize.disabled = true;
      btnEdit.disabled = false;
      layer.clearDrawingItems();

      await router.startCapturing("DetectDocumentBoundaries_Default");
    })

    autoNormalize.addEventListener("change", () => {
      btnEdit.style.display = autoNormalize.checked ? "none" : "inline";
      btnNormalize.style.display = autoNormalize.checked ? "none" : "inline";
    });

    btnEdit.addEventListener("click", async () => {
      if (!cameraEnhancer.isOpen() || items.length <= 1) return;
      /* Stops the detection task since we assume we have found a good boundary. */
      router.stopCapturing();
      /* Hides the cameraView and shows the imageEditorView. */
      cameraViewContainer.style.display = "none";
      imageEditorViewContainer.style.display = "block";
      /* Draws the image on the imageEditorView first. */
      imageEditorView.setOriginalImage(originalImageData);
      quads = [];
      /* Draws the document boundary (quad) over the image. */
      for (let i = 0; i < items.length; i++) {
        if (items[i].type === Dynamsoft.Core.EnumCapturedResultItemType.CRIT_ORIGINAL_IMAGE) continue;
        const points = items[i].location.points;
        const quad = new Dynamsoft.DCE.QuadDrawingItem({ points });
        quads.push(quad);
        layer.addDrawingItems(quads);
      }
      btnStart.style.display = "none";
      btnRestart.style.display = "inline";
      btnNormalize.disabled = false;
      btnEdit.disabled = true;
    });

    btnNormalize.addEventListener("click", async () => {
      /* Gets the selected quadrilateral. */
      let seletedItems = imageEditorView.getSelectedDrawingItems();
      let quad;
      if (seletedItems.length) {
        quad = seletedItems[0].getQuad();
      } else {
        quad = items[1].location;
      }
      const isPointOverBoundary = (point) => {
        if (point.x < 0 ||
          point.x > originalImageData.width ||
          point.y < 0 ||
          point.y > originalImageData.height) {
          return true;
        } else {
          return false;
        }
      };
      /* Check if the points beyond the boundaries of the image. */
      if (quad.points.some(point => isPointOverBoundary(point))) {
        alert("The document boundaries extend beyond the boundaries of the image and cannot be used to normalize the document.");
        return;
      }

      /* Hides the imageEditorView. */
      imageEditorViewContainer.style.display = "none";
      /* Removes the old normalized image if any. */
      normalizedImageContainer.innerHTML = "";
      /**
       * Sets the coordinates of the ROI (region of interest)
       * in the built-in template "NormalizeDocument_Default".
       */
      let newSettings = await router.getSimplifiedSettings("NormalizeDocument_Default");
      newSettings.roiMeasuredInPercentage = 0;
      newSettings.roi.points = quad.points;
      await router.updateSettings("NormalizeDocument_Default", newSettings);
      /* Executes the normalization and shows the result on the page. */
      let normalizeResult = await router.capture(originalImageData, "NormalizeDocument_Default");
      if (normalizeResult.items[0]) {
        normalizedImageContainer.append(normalizeResult.items[0].toCanvas());
      }
      layer.clearDrawingItems();
      btnNormalize.disabled = true;
      btnEdit.disabled = true;
    });
  </script>
</body>
</html>
```

## Download the whole project

- Hello World - [Github](https://github.com/Dynamsoft/mobile-web-capture/tree/master/samples/hello-world-singlepage/hello-world) \| [Run](https://dynamsoft.github.io/mobile-web-capture/samples/hello-world-singlepage/hello-world/hello-world.html)
  - Angular App - [Github](https://github.com/Dynamsoft/mobile-web-capture/tree/master/samples/hello-world-singlepage/hello-world-angular)
  - React App - [Github](https://github.com/Dynamsoft/mobile-web-capture/tree/master/samples/hello-world-singlepage/hello-world-react)
  - Vue3 App - [Github](https://github.com/Dynamsoft/mobile-web-capture/tree/master/samples/hello-world-singlepage/hello-world-vue3)

## More use cases

We provide some samples which demonstrate the popular use cases, for example, review and adjust the boundaries, edit the result images, export the result images in PDF format and so on.

Please refer to the [Use Case]({{ site.codegallery }}usecases/index.html) section.

## [Demo](https://demo.dynamsoft.com/mobile-web-capture/)
