<template>
  <div id="app">
    <div v-show="showContainer" class="face-capture" id="face-capture">
      <video ref="refVideo" id="video" autoplay preload loop muted playsinline webkit-playsinline></video>
      <!--<img src="../../assets/images/video-cover.png" alt="cover" class="img-cover"/>-->
      <div class="control-container face-capture">
        <div class="text-desc">{{scanTip}}</div>
        <img src="../../assets/images/close.png" class="close" alt="" @click="close"/>
        <canvas ref="refCanvas" :width="screenSize.width" :height="screenSize.height" :style="{opacity: 0}"></canvas>
      </div>
      <div class="rect" v-for="item in profile"
           :style="{ width: item.width + 'px', height: item.height + 'px', left: item.left + 'px', top: item.top + 'px'}"></div>
    </div>
    <img v-show="!showContainer" :src="imgUrl"/>
  </div>
</template>

<script>
  import tracking from '@/assets/js/tracking-min.js'
  import '@/assets/js/face-min.js'
  export default {
    name: "faceTracking",
    props:{
    },
    data() {
      return {
        screenSize: {width: window.screen.width, height: window.screen.height},
        URL: null,
        streamIns: null,        // The video stream
        showContainer: true,
        tracker: null,
        tipFlag: false,         // Indicates that the user has detected
        flag: false,            // Determine if a photo has been taken
        context: null,          // Canvas Context
        profile: [],            // Face contour
        removePhotoID: null,    // Stop converting pictures
        scanTip: 'In face recognition...',// text tip
        imgUrl: ''              // base64 format pic
      }
    },
    mounted() {
      this.playVideo()
    },
    methods: {
      /**
       * Video playback
       * @desc The front camera is switched on by default
       * */
      playVideo() {
        this.getUserMedia(
          {
            video: {
              width: 600,
              height: 600,
              facingMode: "user" // Camera front priority
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
          //webkit browser
          navigator.webkitGetUserMedia(constrains).then(success).catch(error);
        } else if (navigator.mozGetUserMedia) {
          //Firefox browser
          navagator.mozGetUserMedia(constrains).then(success).catch(error);
        } else if (navigator.getUserMedia) {
          // Old API
          navigator.getUserMedia(constrains).then(success).catch(error);
        } else {
          this.scanTip = "Your browser does not support access to user media devices"
        }
      },
      success(stream) {
        this.streamIns = stream
        // webkit browser
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
        this.scanTip = "Failed to access user media. Procedure" + e.name + "," + e.message
      },
      /**
       * Face to capture
       */
      initTracker() {
        this.context = this.$refs.refCanvas.getContext("2d")          // Canvas
        this.tracker = new window.tracking.ObjectTracker('face')      // tracker
        this.tracker.setStepSize(1.7)
        this.tracker.on('track', this.handleTracked)                  // listener
        try {
          window.tracking.track('#video', this.tracker)               // Began to track
        } catch (e) {
          // this.scanTip = "Failed to access user media. Please try again"
          this.scanTip = e
        }
      },
      /**
       * Track Event - 【Face tracking】
       * */
      handleTracked(e) {
        if (e.data.length === 0) {
          this.scanTip = 'No human face detected'
        } else {
          if (!this.tipFlag) {
            this.scanTip = 'Detection succeeded, taking photos, please hold for 2 seconds'
          }
          if (!this.flag) {
            // this.scanTip = 'Is taking picture...'
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
        this.imgUrl = this.saveAsPNG(this.$refs.refCanvas)
        /**
         * After you get the Base64 image,
         * you can call the this.pare method to compare the backend interface,
         * or you can call the getBlobBydataURI method to convert it to a file and compare it

         In our project, there is a place to set personal profile picture. First,
         save the user's picture, and then compare the address of the picture with the current photo to the back-end interface.
         * */
        // this.compare(imgUrl)
        this.scanTip = this.imgUrl
        this.$emit('base64GenerateEvent', this.imgUrl)
        this.close()
      },
      /**
       * Base64 convert to file
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
       * Pic compare
       * */
      // compare(url) {
      //     let blob = this.getBlobBydataURI(url, 'image/png')
      //     let formData = new FormData()
      //     formData.append("file", blob, "file_" + Date.parse(new Date()) + ".png")
      //     // TODO Face recognition is performed after obtaining the file
      // },
      /**
       * Save as PNG,base64 image
       * @desc canvas dom convert to base64 pic
       * */
      saveAsPNG(c) {
        return c.toDataURL('image/png', 0.3) // Parameter 2: picture quality; About 30K in size
      },
      /**
       * Close and clean up resources
       */
      close() {
        // alert('Exit the scan')
        this.flag = false
        this.tipFlag = false
        this.showContainer = false
        this.tracker && this.tracker.removeListener('track', this.handleTracked)
        this.tracker = null
        this.context = null
        this.profile = []
        this.scanTip = 'In face recognition...'
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
    /*position: fixed;*/
    /*top: 0;*/
    /*bottom: 0;*/
    /*left: 0;*/
    /*right: 0;*/
    /*width: 100%;*/
    /*height: 100%;*/
    /*object-fit: cover;*/
    /*z-index: 2;*/
    /*background-repeat: no-repeat;*/
    /*background-size: 100% 100%;*/
    position: fixed;
    top: 4rem;
    width: 18rem;
    height: 18rem;
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
    margin-bottom: 1rem;
  }

  .face-capture .control-container {
    /*margin-top: 10rem;*/
    /*position: relative;*/
    /*width: 100%;*/
    /*height: 100%;*/
    /*object-fit: cover;*/
    /*z-index: 4;*/
    /*background-repeat: no-repeat;*/
    /*background-size: 100% 100%;*/
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
    width: 50px;
    height: 50px;
    z-index: 10;
  }
</style>
