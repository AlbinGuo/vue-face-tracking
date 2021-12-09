<template>
  <div id="app">
    <div v-show="showContainer" class="face-capture" id="face-capture">
      <video ref="refVideo" id="video" autoplay preload loop muted playsinline webkit-playsinline></video>
      <div class="control-container face-capture">
        <canvas ref="refCanvas" :width="screenSize.width" :height="screenSize.height" :style="{opacity: 0}"></canvas>
      </div>
      <div class="rect" v-for="item in profile"
           :style="{ width: item.width + 'px', height: item.height + 'px', left: item.left + 'px', top: item.top + 'px'}"></div>
      <div class="text-desc">{{scanTip}}</div>
    </div>
    <img v-show="!showContainer" :src="imgUrl"/>
  </div>
</template>

<script>
  import tracking from './js/tracking-min.js'
  import './js/face-min.js'
  export default {
    name: "faceTracking",
    props:{
    },
    data() {
      return {
        screenSize: {width: window.screen.width, height: window.screen.height},
        URL: null,
        streamIns: null,        // Video Stream
        showContainer: true,
        tracker: null,
        tipFlag: false,         // tips
        flag: false,            // Determine if a photo has been taken
        context: null,          // Canvas Context
        profile: [],            // outline
        removePhotoID: null,    // Stop converting pictures
        scanTip: 'tracking...',// tip text
        imgUrl: ''              // base64 pic
      }
    },
    mounted() {
      this.playVideo()
    },
    methods: {
      /**
       * rescan
       * */
      reScan(){
        this.playVideo()
      },
      /**
       * Play Video - Camera front priority
       * */
      playVideo() {
        this.getUserMedia(
          {
            video: {
              width: 600,
              height: 600,
              facingMode: "user" /* Camera front priority */
            }
          },
          this.success,
          this.error)
      },
      /**
       * Access user media devices
       * @param constrains
       * @param success
       * @param error
       */
      getUserMedia(constrains, success, error) {
        if (navigator.mediaDevices.getUserMedia) {
          //The latest standards API
          navigator.mediaDevices.getUserMedia(constrains).then(success).catch(error);
        } else if (navigator.webkitGetUserMedia) {
          //webkit Kernel browser
          navigator.webkitGetUserMedia(constrains).then(success).catch(error);
        } else if (navigator.mozGetUserMedia) {
          //Firefox browser
          navagator.mozGetUserMedia(constrains).then(success).catch(error);
        } else if (navigator.getUserMedia) {
          //Old API
          navigator.getUserMedia(constrains).then(success).catch(error);
        } else {
          this.scanTip = "Your browser does not support access to user media devices"
        }
      },
      success(stream) {
        this.streamIns = stream
        this.URL = window.URL || window.webkitURL
        if ("srcObject" in this.$refs.refVideo) {
          this.$refs.refVideo.srcObject = stream
        } else {
          this.$refs.refVideo.src = this.URL.createObjectURL(stream)
        }
        this.$refs.refVideo.onloadedmetadata = e => {
          this.$refs.refVideo.play()
          this.initTracker()
        }
      },
      error(e) {
        // this.scanTip = "Failed to access user media. Procedure" + e.name + "," + e.message
        this.scanTip = "Failed to access user media. Procedure"
      },
      /**
       * Face to capture
       */
      initTracker() {
        this.context = this.$refs.refCanvas.getContext("2d")
        this.tracker = new window.tracking.ObjectTracker('face')
        this.tracker.setStepSize(1.7)
        this.tracker.on('track', this.handleTracked)
        try {
          window.tracking.track('#video', this.tracker)               // tracking
        } catch (e) {
          // this.scanTip = "Failed to access user media. Please try again"
          this.scanTip = e
        }
      },
      /**
       * Tracking events - [Face tracking]
       * */
      handleTracked(e) {
        if (e.data.length === 0) {
          this.scanTip = 'No human face detected'
        } else {
          if (!this.tipFlag) {
            this.scanTip = 'Detection succeeded, taking photos, please hold for 2 seconds'
          }
          if (!this.flag) {
            this.flag = true
            this.removePhotoID = setTimeout(() => {
              this.tackPhoto()
              this.tipFlag = true
            }, 2000)
          }
          e.data.forEach(this.plot)
        }
      },
      /**
       * Draw trace box
       * */
      plot({x, y, width: w, height: h}) {
        this.profile.push({ width: w, height: h, left: x, top: y })
      },
      /**
       * Take photos after successful identification
       */
      tackPhoto() {
        this.context.drawImage(this.$refs.refVideo, 0, 0, this.screenSize.width, 400)
        // pic base64
        this.imgUrl = this.saveAsPNG(this.$refs.refCanvas)
        /** After you get the Base64 image, you can call the this.pare method to compare the backend interface,
         * or you can call the getBlobBydataURI method to convert it to a file and compare it
         * In our project, there is a place to set personal profile picture. First, save the user's picture,
         * and then compare the address of the picture with the current photo to the back-end interface.
         * */
        // this.compare(imgUrl)
        this.scanTip = this.imgUrl
        this.$emit('base64GenerateEvent', this.imgUrl)
        this.close()
      },
      /**
       * Base64 to File
       * */
      // getBlobBydataURI(dataURI, type) {
      //   var binary = window.atob(dataURI.split(',')[1]);
      //   var array = [];
      //   for(var i = 0; i < binary.length; i++) {
      //     array.push(binary.charCodeAt(i));
      //   }
      //   return new Blob([new Uint8Array(array)], {
      //     type: type
      //   });
      // },
      /**
       * pic compare
       * */
      // compare(url) {
      //     let blob = this.getBlobBydataURI(url, 'image/png')
      //     let formData = new FormData()
      //     formData.append("file", blob, "file_" + Date.parse(new Date()) + ".png")
      //     // TODO Face recognition is performed after obtaining the file
      // },
      /**
       * Save as PNG,base64 image
       * */
      saveAsPNG(c) {
        return c.toDataURL('image/png', 0.3) // pic size 30K.
      },
      /**
       * Close and clean up resources
       */
      close() {
        this.flag = false
        this.tipFlag = false
        this.showContainer = false
        this.tracker && this.tracker.removeListener('track', this.handleTracked)
        this.tracker = null
        this.context = null
        this.profile = []
        this.scanTip = 'Tracking...'
        clearTimeout(this.removePhotoID)
        if (this.streamIns) {
          this.streamIns.enabled = false
          this.streamIns.getTracks()[0].stop()
          this.streamIns.getVideoTracks()[0].stop()
        }
        this.streamIns = null
      },
    }
  }
</script>

<style>
  .face-capture {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }

  .face-capture video, .face-capture canvas {
    position: fixed;
    top: 2rem;
    width: 6rem;
    height: 6rem;
    -o-object-fit: cover;
    object-fit: cover;
    z-index: 2;
    background-repeat: no-repeat;
    background-size: 100% 100%;
    background-color: green;
    border-radius: 50%;
  }

  .face-capture canvas {
    z-index: 2;
  }

  .face-capture .img-cover {
    position: fixed;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    z-index: 2;
    background-repeat: no-repeat;
    background-size: 100% 100%;
  }

  .face-capture .rect {
    border: 2px solid #0aeb08;
    position: fixed;
    z-index: 3;
  }

  .text-desc{
    font-size: .4rem;
    position: absolute;
    bottom: 2rem;
  }

  .face-capture .control-container {
    position: absolute;
    bottom: 5rem;
  }

  .face-capture .title {
    text-align: center;
    color: white;
    margin: 1.6rem auto;
    font-size: 18px;
  }

  .face-capture .close {
    width: .6rem;
    height: .6rem;
    z-index: 10;
    position: fixed;
    bottom: 2rem;
  }
</style>
