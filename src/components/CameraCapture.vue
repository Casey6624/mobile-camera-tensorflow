<template>
  <div>
    <h3 v-if="!isVideoStreamReady && !initFailMessage">Initializing webcam stream ...</h3>
    <h3 v-if="!isModelReady && !initFailMessage">loading model ...</h3>
    <h3 v-if="initFailMessage">Failed to init stream and/or model - {{ initFailMessage }}</h3>

    <div class="resultFrame">
      <video ref="video" autoplay></video>
      <canvas ref="canvas" :width="resultWidth" :height="resultHeight"></canvas>
    </div>
    <button @click="startMatching = !startMatching">{{ startMatching ? "Stop" : "Start" }}</button>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Watch } from "vue-property-decorator";
import * as cocoSsd from "@tensorflow-models/coco-ssd";
import * as blazeFace from "@tensorflow-models/blazeface";
//import * as tf from "@tensorflow/tfjs";
import "@tensorflow/tfjs-backend-cpu";
import "@tensorflow/tfjs-backend-webgl";
//import "@tensorflow/tfjs-converter";
@Component({})
export default class CameraCapture extends Vue {
  public streamPromise: any;
  public modelPromise: any;
  public isVideoStreamReady = false;
  public isModelReady = false;
  public initFailMessage = "";
  public model: any;
  public baseModel: any = "lite_mobilenet_v2";
  public videoRatio = 1;
  public resultWidth = 0;
  public resultHeight = 0;
  public startMatching = false;
  public timer: any = null;

  public detectionType: "objects" | "faces" = "faces";

  @Watch("startMatching")
  startMatch(value: boolean) {
    clearTimeout(this.timer);
    if (value) {
      this.timer = setInterval(
        function() {
          requestAnimationFrame(
            function() {
              (this as any).detectObjects();
            }.bind(this)
          );
        }.bind(this),
        500
      );
    }
  }

  public initWebcamStream() {
    // if the browser supports mediaDevices.getUserMedia API
    if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
      return navigator.mediaDevices
        .getUserMedia({
          audio: false, // don't capture audio
          video: { facingMode: "environment" } // use the rear camera if there is
        })
        .then(stream => {
          // set <video> source as the webcam input
          const video: any = this.$refs.video;
          try {
            video.srcObject = stream;
          } catch (error) {
            // support older browsers
            video.src = URL.createObjectURL(stream);
          }
          /*
              model.detect uses tf.fromPixels to create tensors.
              tf.fromPixels api will get the <video> size from the width and height attributes,
                which means <video> width and height attributes needs to be set before called model.detect
              To make the <video> responsive, I get the initial video ratio when it's loaded (onloadedmetadata)
              Then addEventListener on resize, which will adjust the size but remain the ratio
              At last, resolve the Promise.
            */
          this.isVideoStreamReady = true;
          this.setResultSize();
          return new Promise((resolve: any, reject) => {
            // when video is loaded
            video.onloadedmetadata = () => {
              // calculate the video ratio
              this.videoRatio = video.offsetHeight / video.offsetWidth;
              // add event listener on resize to reset the <video> and <canvas> sizes

              // set the initial size
              //this.setResultSize();
              //this.isVideoStreamReady = true;
              console.log("webcam stream initialized");
              resolve();
            };
          });
        })
        .catch(error => {
          console.log("failed to initialize webcam stream", error);
          window.removeEventListener("resize", this.setResultSize);
          throw error;
        });
    } else {
      return Promise.reject(
        new Error("Your browser does not support mediaDevices.getUserMedia API")
      );
    }
  }

  public setResultSize() {
    // get the current browser window size
    const clientWidth = document.documentElement.clientWidth;
    // set max width as 600
    this.resultWidth = Math.min(600, clientWidth);
    // set the height according to the video ratio
    this.resultHeight = this.resultWidth * this.videoRatio;
    // set <video> width and height
    /*
        Doesn't use vue binding :width and :height,
          because the initial value of resultWidth and resultHeight
          will affect the ratio got from the initWebcamStream()
      */
    const video: any = this.$refs.video;
    video.width = this.resultWidth;
    video.height = this.resultHeight;
  }
  public loadModel() {
    this.isModelReady = false;
    // if model already exists => dispose it and load a new one
    if (this.model) this.model.dispose();
    // load model with the baseModel
    if (this.detectionType === "objects") {
      return cocoSsd
        .load(this.baseModel)
        .then(model => {
          this.model = model;
          this.isModelReady = true;
          console.log("model loaded");
        })
        .catch(error => {
          console.log("failed to load the model", error);
          throw error;
        });
    } else if (this.detectionType === "faces") {
      return blazeFace
        .load()
        .then(model => {
          this.model = model;
          this.isModelReady = true;
          console.log("model loaded");
        })
        .catch(error => {
          console.log("failed to load the model", error);
          throw error;
        });
    }
  }
  async detectObjects() {
    if (!this.isModelReady) return;
    let predictions;

    if (this.detectionType === "objects") {
      predictions = await this.model.detect(this.$refs.video);
    } else if (this.detectionType === "faces") {
      predictions = await this.model.estimateFaces(this.$refs.video, false);
    }
    console.log(predictions);
    this.renderPredictions(predictions);
  }

  async loadModelAndStartDetecting() {
    // wait for both stream and model promise finished
    // => start detecting objects
    Promise.all([this.streamPromise, this.loadModel()])
      .then(() => {
        this.detectObjects();
      })
      .catch(error => {
        console.log(error);
        console.log("Failed to init stream and/or model");
        this.initFailMessage = error;
      });
  }
  renderPredictions(predictions: any) {
    // get the context of canvas
    const ctx: any = (this.$refs.canvas as any).getContext("2d");
    // clear the canvas
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
    if (this.detectionType === "objects") {
      predictions.forEach((prediction: any) => {
        ctx.lineWidth = 3;
        ctx.strokeStyle = "#34bebd";
        ctx.fillStyle = "#34bebd";
        ctx.stroke();
        ctx.shadowBlur = 10;
        ctx.font = "24px Arial bold";

        ctx.beginPath();
        ctx.rect(...prediction.bbox);
        ctx.fillText(
          `${(prediction.score * 100).toFixed(1)}% ${prediction.class}`,
          prediction.bbox[0],
          prediction.bbox[1] > 10 ? prediction.bbox[1] - 5 : 10
        );
      });
    } else if (this.detectionType === "faces") {
      predictions.forEach((prediction: any) => {
        ctx.beginPath();
        ctx.rect(...prediction.topLeft, ...prediction.bottomRight);
        ctx.fillText(
          `${(prediction.length > 0 ? prediction.probability[0] : 0 * 100).toFixed(1)}% ${
            prediction.length > 0 ? prediction.probability[0] : 0
          }`,
          prediction.topLeft,
          prediction.bottomRight > 10 ? prediction.bbox[1] - 5 : 10
        );
      });
    }
  }
  mounted() {
    this.streamPromise = this.initWebcamStream();
    this.loadModelAndStartDetecting();
    window.addEventListener("resize", this.setResultSize);
  }

  destoryed() {
    window.removeEventListener("resize", this.setResultSize);
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

body {
  margin: 0;
}
.resultFrame {
  display: grid;
}

video {
  grid-area: 1 / 1 / 2 / 2;
}
canvas {
  grid-area: 1 / 1 / 2 / 2;
}
</style>
