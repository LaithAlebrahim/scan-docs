<template>
  <div style="padding: 30px">
    <h2 style="margin-top: 10px">
      A tool for upload docs and crop it to keep just the content
    </h2>

    <h3 style="margin-bottom: 10px">Upload your images:</h3>
    <input
      type="file"
      multiple
      @change="handleFileChange"
      accept="image/png, image/gif, image/jpeg"
    />

    <div v-if="results.length" style="margin-top: 50px">
      <h2 style="margin-bottom: 0; margin-left: 20px">Results</h2>
      <div style="display: flex; flex-wrap: wrap">
        <div
          v-for="(result, index) in results"
          :key="index"
          style="margin: 20px"
        >
          <div>
            <h3>Original image</h3>
            <img
              :src="result.orig"
              style="max-height: 400px; max-width: 400px"
            />
          </div>
          <div :id="`extracted${index}`">
            <h3>Extracted Paper</h3>
          </div>
          <div>
            <h3>Corner Points</h3>
            <pre style="font-family: monospace">{{ result.cornerPoints }}</pre>
          </div>
          <button @click="saveCroppedImage(result.croppedImage, index)">
            Save Cropped Image
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted } from 'vue';

const results = ref([]);

const handleFileChange = async (event) => {
  const files = event.target.files;

  for (const file of files) {
    const origSrc = URL.createObjectURL(file);
    const img = new Image();
    img.src = origSrc;
    const enhanceImage = (canvas) => {
      const src = window.cv.imread(canvas);
      let dst = new window.cv.Mat();
      let m = window.cv.Mat.ones(3, 3, window.cv.CV_8U);
      let anchor = new window.cv.Point(-1, -1);

      // Removing shadows: Dilate to increase object area
      window.cv.dilate(
        src,
        dst,
        m,
        anchor,
        1,
        window.cv.BORDER_CONSTANT,
        window.cv.morphologyDefaultBorderValue()
      );

      // Enhance contrast with reduced brightness
      dst.convertTo(dst, -1, 1.1, 15); // Reduced contrast and brightness slightly

      // Enhance resolution: Scale the image
      let resized = new window.cv.Mat();
      window.cv.resize(
        dst,
        resized,
        new window.cv.Size(),
        1.5,
        1.5,
        window.cv.INTER_CUBIC
      );

      // Convert back to canvas to display in the browser
      window.cv.imshow(canvas, resized);
      src.delete();
      dst.delete();
      m.delete();
      resized.delete();
    };

    img.onload = async () => {
      const scanner = new window.jscanify();
      const contour = scanner.findPaperContour(window.cv.imread(img));
      const cornerPoints = scanner.getCornerPoints(contour);

      const croppedCanvas = document.createElement('canvas');
      const ctx = croppedCanvas.getContext('2d');

      const {
        topLeftCorner,
        topRightCorner,
        bottomLeftCorner,
        bottomRightCorner,
      } = cornerPoints;

      const width = topRightCorner.x - topLeftCorner.x;
      const height = bottomLeftCorner.y - topLeftCorner.y;

      croppedCanvas.width = width;
      croppedCanvas.height = height;

      ctx.drawImage(
        img,
        topLeftCorner.x,
        topLeftCorner.y,
        width,
        height,
        0,
        0,
        width,
        height
      );

      // Apply enhancements
      enhanceImage(croppedCanvas);

      const croppedImage = croppedCanvas.toDataURL('image/png');

      results.value.push({
        orig: origSrc,
        croppedImage,
        cornerPoints: JSON.stringify(cornerPoints, null, 4),
      });

      await nextTick();

      const extractedDiv = document.getElementById(
        `extracted${results.value.length - 1}`
      );
      const imgElement = new Image();
      imgElement.src = croppedImage;
      imgElement.style.maxHeight = '400px';
      imgElement.style.maxWidth = '400px';
      extractedDiv.appendChild(imgElement);
    };
  }
};

const saveCroppedImage = (croppedImage, index) => {
  const link = document.createElement('a');
  link.href = croppedImage;
  link.download = `cropped_image_${index}.png`;
  link.click();
};

onMounted(() => {
  const script = document.createElement('script');
  script.src = 'https://docs.opencv.org/4.7.0/opencv.js';
  script.async = true;
  document.body.appendChild(script);

  const jscanifyScript = document.createElement('script');
  jscanifyScript.src =
    'https://cdn.jsdelivr.net/gh/ColonelParrot/jscanify@master/src/jscanify.min.js';
  jscanifyScript.async = true;
  document.body.appendChild(jscanifyScript);
});
</script>

<style scoped>
h1,
h2,
h3 {
  font-weight: 400;
}

img,
canvas {
  max-height: 400px;
  max-width: 400px;
}
</style>
