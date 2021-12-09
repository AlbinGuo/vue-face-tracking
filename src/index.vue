<template>
  <div id="app">
    <div v-show="showContainer" class="face-capture" id="face-capture">
      <video ref="refVideo" id="video" autoplay preload loop muted playsinline webkit-playsinline></video>
      <!--<img src="../../assets/images/video-cover.png" alt="cover" class="img-cover"/>-->
      <div class="control-container face-capture">
        <canvas ref="refCanvas" :width="screenSize.width" :height="screenSize.height" :style="{opacity: 0}"></canvas>
      </div>
      <div class="rect" v-for="item in profile"
           :style="{ width: item.width + 'px', height: item.height + 'px', left: item.left + 'px', top: item.top + 'px'}"></div>
      <div class="text-desc">AAAAAAAAAAAAAAA</div>
    </div>
    <img v-show="!showContainer" :src="imgUrl"/>
  </div>
</template>

<script>
  // 引入trackingjs所需文件
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
        streamIns: null,        // 视频流
        showContainer: true,    // 显示
        tracker: null,
        tipFlag: false,         // 提示用户已经检测到
        flag: false,            // 判断是否已经拍照
        context: null,          // canvas上下文
        profile: [],            // 轮廓
        removePhotoID: null,    // 停止转换图片
        scanTip: '人脸识别中...',// 提示文字
        imgUrl: ''              // base64格式图片
      }
    },
    mounted() {
      this.playVideo()
    },
    methods: {
      /**
       * 重新扫描
       * */
      reScan(){
        this.playVideo()
      },
      /**
       * 视频播放 - 默认调起前置摄像头
       * */
      playVideo() {
        alert(11)
        this.getUserMedia(
          {
            video: {
              width: 600,
              height: 600,
              facingMode: "user" /* 摄像头前置优先 */
            }
          },
          this.success,
          this.error)
      },
      /**
       * 访问用户媒体设备
       * @param constrains
       * @param success
       * @param error
       */
      getUserMedia(constrains, success, error) {
        if (navigator.mediaDevices.getUserMedia) {
          //最新标准API
          navigator.mediaDevices.getUserMedia(constrains).then(success).catch(error);
        } else if (navigator.webkitGetUserMedia) {
          //webkit内核浏览器
          navigator.webkitGetUserMedia(constrains).then(success).catch(error);
        } else if (navigator.mozGetUserMedia) {
          //Firefox浏览器
          navagator.mozGetUserMedia(constrains).then(success).catch(error);
        } else if (navigator.getUserMedia) {
          //旧版API
          navigator.getUserMedia(constrains).then(success).catch(error);
        } else {
          this.scanTip = "你的浏览器不支持访问用户媒体设备"
        }
      },
      success(stream) {
        this.streamIns = stream
        // webkit内核浏览器
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
        // this.scanTip = "访问用户媒体失败" + e.name + "," + e.message
        this.scanTip = "访问用户媒体失败"
      },
      /**
       * 人脸捕捉
       */
      initTracker() {
        this.context = this.$refs.refCanvas.getContext("2d")          // 画布
        this.tracker = new window.tracking.ObjectTracker('face')      // tracker实例
        this.tracker.setStepSize(1.7)
        this.tracker.on('track', this.handleTracked)                  // 绑定监听方法
        try {
          window.tracking.track('#video', this.tracker)               // 开始追踪
        } catch (e) {
          // this.scanTip = "访问用户媒体失败，请重试"
          this.scanTip = e
        }
      },
      /**
       * 追踪事件 - 【脸部追踪】
       * */
      handleTracked(e) {
        if (e.data.length === 0) {
          this.scanTip = '未检测到人脸'
        } else {
          if (!this.tipFlag) {
            this.scanTip = '检测成功，正在拍照，请保持不动2秒'
          }
          // 1秒后拍照，仅拍一次
          if (!this.flag) {
            // this.scanTip = '拍照中...'
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
       * 绘制跟踪框
       * */
      plot({x, y, width: w, height: h}) {
        // 创建框对象
        this.profile.push({ width: w, height: h, left: x, top: y })
      },
      /**
       * 识别成功后拍照
       */
      tackPhoto() {
        this.context.drawImage(this.$refs.refVideo, 0, 0, this.screenSize.width, 400)
        // 保存为base64格式
        this.imgUrl = this.saveAsPNG(this.$refs.refCanvas)
        /** 拿到base64格式图片之后就可以在this.compare方法中去调用后端接口比较了，也可以调用getBlobBydataURI方法转化成文件再去比较
         * 我们项目里有一个设置个人头像的地方，先保存一下用户的图片，然后去拿这个图片的地址和当前拍照图片给后端接口去比较。
         * */
        // this.compare(imgUrl)
        this.scanTip = this.imgUrl
        this.$emit('base64GenerateEvent', this.imgUrl)
        this.close()
      },
      /**
       * Base64转文件
       * */
      // getBlobBydataURI(dataURI, type) {
      //   // base64解码
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
       * 照片比对
       * */
      // compare(url) {
      //     let blob = this.getBlobBydataURI(url, 'image/png')
      //     let formData = new FormData()
      //     formData.append("file", blob, "file_" + Date.parse(new Date()) + ".png")
      //     // TODO 得到文件后进行人脸识别
      // },
      /**
       * 保存为png,base64格式图片
       * @desc canvas的dom转化为图片base64
       * */
      saveAsPNG(c) {
        return c.toDataURL('image/png', 0.3) // 参数二：图片质量；大小30K左右
      },
      /**
       * 关闭并清理资源
       */
      close() {
        this.flag = false
        this.tipFlag = false
        this.showContainer = false
        this.tracker && this.tracker.removeListener('track', this.handleTracked)
        this.tracker = null
        this.context = null
        this.profile = []
        this.scanTip = '人脸识别中...'
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
