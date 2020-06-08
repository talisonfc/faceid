<template>
  <v-app>
    <v-content>
      <div id="app">
        <video ref="video" id="video" width="640" height="480" autoplay></video>
        <canvas ref="canvas" id="canvas" width="640" height="480"></canvas>
      </div>

      <v-form>
        <v-text-field outlined label="Label" v-model="currentDescriptor.label"></v-text-field>
      </v-form>

      <v-btn @click="capture()">Start</v-btn>
      <v-btn @click="enabled = false">Finish</v-btn>
      <v-btn @click="saveDescriptors()">Save</v-btn>
      <v-btn @click="recoginize()">Recoginize</v-btn>
    </v-content>
  </v-app>
</template>

<script>
import * as faceapi from "face-api.js";

export default {
  name: "App",
  data() {
    return {
      enabled: false,
      selectedFaceDetector: "TINY_FACE_DETECTOR",
      minConfidence: 0.5,
      inputSize: 512,
      scoreThreshold: 0.5,
      video: {},
      canvas: {},
      captures: [],
      options: undefined,
      labeledDescriptors: [],
      currentDescriptor: {
        label: undefined,
        descriptors: []
      }
    };
  },
  methods: {
    onPlay(camera) {
      console.log(camera);
    },
    getCurrentFaceDetectionNet() {
      if (this.selectedFaceDetector === "SSD_MOBILENETV1") {
        return faceapi.nets.ssdMobilenetv1;
      }
      if (this.selectedFaceDetector === "TINY_FACE_DETECTOR") {
        return faceapi.nets.tinyFaceDetector;
      }
      if (this.selectedFaceDetector === "MTCNN") {
        return faceapi.nets.mtcnn;
      }
      if (this.selectedFaceDetector == "FACELANDMARK68TINYNET") {
        return faceapi.nets.faceLandmark68TinyNet;
      }
    },
    getFaceDetectorOptions() {
      return this.selectedFaceDetector === "SSD_MOBILENETV1"
        ? new faceapi.SsdMobilenetv1Options({
            minConfidence: this.minConfidence
          })
        : this.selectedFaceDetector === "TINY_FACE_DETECTOR"
        ? new faceapi.TinyFaceDetectorOptions({
            inputSize: this.inputSize,
            scoreThreshold: this.scoreThreshold
          })
        : new faceapi.MtcnnOptions({ minFaceSize: this.minFaceSize });
    },
    isFaceDetectionModelLoaded() {
      return !!this.getCurrentFaceDetectionNet().params;
    },
    async initNetworkNeural() {
      if (
        this.video.paused ||
        this.video.ended ||
        !this.isFaceDetectionModelLoaded()
      ) {
        const net = this.getCurrentFaceDetectionNet();
        await faceapi.loadFaceLandmarkModel("/models");
        await faceapi.loadFaceRecognitionModel("/models");
        await net.load("/models");
      }
    },
    captureDescritor(detection) {
      if (detection !== undefined) {
        const { descriptor } = detection;
        if (descriptor !== undefined) {
          this.currentDescriptor.descriptors.push(descriptor);
        }
      }
    },
    saveDescriptors() {
      this.labeledDescriptors.push(
        new faceapi.LabeledFaceDescriptors(this.currentDescriptor.label, [
          ...this.currentDescriptor.descriptors
        ])
      );

      this.currentDescriptor = {
        label: undefined,
        descriptors: []
      };
    },
    async recoginize() {
      const result = await this.detect();
      if (result !== undefined && result.descriptor !== undefined) {
        const faceMatcher = new faceapi.FaceMatcher(this.labeledDescriptors);
        const bestMatch = faceMatcher.findBestMatch(result.descriptor);
        console.log(bestMatch);
        const distancia = bestMatch.distance.toFixed(2)

        const dims = faceapi.matchDimensions(this.canvas, this.video, true);
        const resizedResult = faceapi.resizeResults(result, dims);

        const _box = resizedResult.alignedRect._box
        const box = { x: _box._x, y: _box._y, width: _box._width, height: _box._height };
        // see DrawBoxOptions below
        const drawOptions = {
          label: `${bestMatch.label} - ${distancia}`,
          lineWidth: 2
        };
        const drawBox = new faceapi.draw.DrawBox(box, drawOptions);
        drawBox.draw(this.canvas);
      }
    },
    async detect() {
      const result = await faceapi
        .detectSingleFace(this.video, this.options)
        .withFaceLandmarks()
        .withFaceDescriptor();

      const dims = faceapi.matchDimensions(this.canvas, this.video, true);
      const resizedResult = faceapi.resizeResults(result, dims);

      faceapi.draw.drawDetections(this.canvas, resizedResult);
      faceapi.draw.drawFaceLandmarks(this.canvas, resizedResult);

      return result;
    },
    async capture() {
      try {
        const result = await this.detect();
        this.captureDescritor(result);
      } catch (e) {
        console.error(e);
      }
    }
  },
  mounted() {
    this.video = this.$refs.video;
    this.canvas = this.$refs.canvas;
    if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
      navigator.mediaDevices.getUserMedia({ video: true }).then(stream => {
        this.video.srcObject = stream;
      });
    }

    this.initNetworkNeural();
    this.options = this.getFaceDetectorOptions();
  }
};
</script>

<style scope>
body {
  background-color: #f0f0f0;
}
#app {
  color: #2c3e50;
}
#video {
  background-color: #000000;
}
#canvas {
  position: absolute;
  top: 0;
  left: 0;
  border: 1px solid red;
}
li {
  display: inline;
  padding: 5px;
}
</style>

