<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
    <div>
      <video ref="video" width="400" height="300" autoplay></video>
      <button @click="startCamera">Start Camera</button>
      <button @click="stopCamera">Stop Camera</button>
      <p>PEOPLE COUNT: {{ peopleCount }}</p>
    </div>
    <!-- Removendo a segunda seção de câmera -->
    <!-- <div>
      <canvas id="srcimg" ref="srcimg" class="imgcanvas"></canvas>
      <canvas id="dstimg" ref="dstimg" class="imgcanvas"></canvas>
    </div> -->
    <canvas id="canvas" ref="canvas" class="imgcanvas"></canvas>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'Main',
  props: {
    msg: String
  },
  data() {
    return {
      faceClass: null,
      eyeClass: null,
      files: [],
      videoStream: null,
      peopleCount: 0
    }
  },
  created() {
    let cascadeFile = 'haarcascade_frontalface_default.xml'
    this.createFileFromURL(cascadeFile, cascadeFile, (face) => { this.faceClass = face })
    cascadeFile = 'haarcascade_eye.xml'
    this.createFileFromURL(cascadeFile, cascadeFile, (eye) => { this.eyeClass = eye })
  },
  methods: {
    startCamera() {
      navigator.mediaDevices.getUserMedia({ video: true })
        .then(stream => {
          this.$refs.video.srcObject = stream;
          this.videoStream = stream;
          this.$refs.video.play();
          this.detectFace();
        })
        .catch(err => {
          console.error('Error accessing camera:', err);
        });
    },
    stopCamera() {
      if (this.videoStream) {
        this.videoStream.getTracks().forEach(track => {
          track.stop();
        });
      }
    },
    detectFace() {
      let video = this.$refs.video;
      let canvas = this.$refs.canvas;
      let ctx = canvas.getContext('2d');

      const drawFrame = () => {
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
        let srcMat = this.$cv.imread(canvas);
        let gray = new this.$cv.Mat();
        this.$cv.cvtColor(srcMat, gray, this.$cv.COLOR_RGBA2GRAY);
        let faces = new this.$cv.RectVector();
        let eyes = new this.$cv.RectVector();
        this.faceClass.detectMultiScale(gray, faces);
        this.peopleCount = faces.size();
        for (let i = 0; i < faces.size(); i++) {
          let roiGray = gray.roi(faces.get(i));
          let roiSrc = srcMat.roi(faces.get(i));
          let point1 = new this.$cv.Point(faces.get(i).x, faces.get(i).y);
          let point2 = new this.$cv.Point(faces.get(i).x + faces.get(i).width, faces.get(i).y + faces.get(i).height);
          this.$cv.rectangle(srcMat, point1, point2, [255, 0, 0, 255]);
          this.eyeClass.detectMultiScale(roiGray, eyes);
          for (let j = 0; j < eyes.size(); j++) {
            let point1 = new this.$cv.Point(eyes.get(j).x, eyes.get(j).y);
            let point2 = new this.$cv.Point(eyes.get(j).x + eyes.get(j).width, eyes.get(j).y + eyes.get(j).height);
            this.$cv.rectangle(roiSrc, point1, point2, [0, 255, 0, 255]);
          }
          roiGray.delete();
          roiSrc.delete();
        }
        this.$cv.imshow(canvas, srcMat);
        srcMat.delete();
        gray.delete();
        faces.delete();
        eyes.delete();
        requestAnimationFrame(drawFrame);
      };

      drawFrame();
    },
    createFileFromURL(file, url, cb) {
      axios.get('/models/haarcascades/' + url)
        .then((resp) => {
          let rtn = this.$cv.FS_createDataFile('/', file, resp.data, true, false, false)
          if (!rtn) return cb(null)
          let classifier = new this.$cv.CascadeClassifier()
          rtn = classifier.load(file)
          if (!rtn) return cb(null)

          cb(classifier)
          console.log('loaded', rtn, classifier.empty(), this.faceClass)

        })
        .catch((err) => {
          console.log('ERR', err);
        })
    }
  },
  beforeDestroy() {
    this.stopCamera();
  }
}
</script>

<style scoped>
.imgcanvas {
  width: 400px;
  height: 300px;
  border: 2px solid #000;
  margin: 10px;
}
</style>
