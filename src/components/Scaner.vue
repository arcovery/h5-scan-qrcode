<template>
  <div class="scaner" ref="scaner">
    <div class="banner" v-if="showBanner">
      <i class="close_icon" @click="() => (showBanner = false)"></i>
      <p class="text">提示</p>
    </div>
    <div class="cover">
      <p class="line"></p>
      <span class="square top left"></span>
      <span class="square top right"></span>
      <span class="square bottom right"></span>
      <span class="square bottom left"></span>
      <p class="tips">将二维码放入框内，即可自动扫描</p>
    </div>
    <div class="btn" @click="setup">
      <span class="text" >你好啊</span>
      <svg
        t="1696869280858"
        class="icon"
        viewBox="0 0 1024 1024"
        version="1.1"
        xmlns="http://www.w3.org/2000/svg"
        p-id="2462"
        width="24"
        height="24">
        <path
          d="M305.587883 813.744345c-14.09502 12.824073-15.126512 34.658358-2.303462 48.752354 12.843516 14.09502 34.678824 15.126512 48.752354 2.302439l364.835266-332.676845c14.114462-12.844539 15.127536-34.659381 2.302439-48.753377-0.733711-0.815575-1.486864-1.588171-2.302439-2.302439l-0.078795-0.080841-0.13917-0.13917L352.037798 148.369166c-14.074553-12.824073-35.909861-11.791557-48.752354 2.302439-12.824073 14.09502-11.792581 35.930327 2.303462 48.753377l336.845795 307.168891L305.588907 813.743322 305.587883 813.744345 305.587883 813.744345z"
          fill="#272536"
          p-id="2463"></path>
      </svg>
    </div>
    <video
      v-show="showPlay"
      class="source"
      ref="video"
      :width="videoWH.width"
      :height="videoWH.height"
      controls></video>
    <canvas v-show="!showPlay" ref="canvas" />
    <button v-show="showPlay" @click="setup">开始</button>
  </div>
</template>

<script>
// eslint-disable-next-line no-unused-vars
import adapter from 'webrtc-adapter'
import jsQR from 'jsqr'

export default {
  name: 'Scaner',
  props: {
    // 使用后置相机
    useBackCamera: {
      type: Boolean,
      default: true,
    },
    // 扫描识别后停止
    stopOnScaned: {
      type: Boolean,
      default: true,
    },
    drawOnfound: {
      type: Boolean,
      default: true,
    },
    // 线条颜色
    lineColor: {
      type: String,
      default: '#03C03C',
    },
    // 线条宽度
    lineWidth: {
      type: Number,
      default: 2,
    },
    // 视频宽度
    videoWidth: {
      type: Number,
      default:
        document.documentElement.clientWidth || document.body.clientWidth,
    },
    // 视频高度
    videoHeight: {
      type: Number,
      default:
        document.documentElement.clientHeight || document.body.clientHeight,
    },
    responsive: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      showPlay: false,
      showBanner: true,
      containerWidth: null,
      active: false,
    }
  },
  computed: {
    videoWH() {
      if (this.containerWidth) {
        const width = this.containerWidth
        const height = width * 1
        return { width, height }
      }
      return { width: this.videoWidth, height: this.videoHeight }
    },
  },
  watch: {
    active: {
      immediate: true,
      handler(active) {
        if (!active) {
          this.fullStop()
        }
      },
    },
  },
  methods: {
    // 画线
    drawLine(begin, end) {
      this.canvas.beginPath()
      this.canvas.moveTo(begin.x, begin.y)
      this.canvas.lineTo(end.x, end.y)
      this.canvas.lineWidth = this.lineWidth
      this.canvas.strokeStyle = this.lineColor
      this.canvas.stroke()
    },
    // 画框
    drawBox(location) {
      if (this.drawOnfound) {
        this.drawLine(location.topLeftCorner, location.topRightCorner)
        this.drawLine(location.topRightCorner, location.bottomRightCorner)
        this.drawLine(location.bottomRightCorner, location.bottomLeftCorner)
        this.drawLine(location.bottomLeftCorner, location.topLeftCorner)
      }
    },
    tick() {
      if (
        this.$refs.video &&
        this.$refs.video.readyState === this.$refs.video.HAVE_ENOUGH_DATA
      ) {
        this.$refs.canvas.height = this.videoWH.height
        this.$refs.canvas.width = this.videoWH.width
        this.canvas.drawImage(
          this.$refs.video,
          0,
          0,
          this.$refs.canvas.width,
          this.$refs.canvas.height
        )
        const imageData = this.canvas.getImageData(
          0,
          0,
          this.$refs.canvas.width,
          this.$refs.canvas.height
        )
        let code = false
        try {
          code = jsQR(imageData.data, imageData.width, imageData.height)
        } catch (e) {
          console.error(e)
        }
        if (code) {
          this.drawBox(code.location)
          this.found(code.data)
        }
      }
      this.run()
    },
    // 初始化
    setup() {
      if (this.responsive) {
        this.$nextTick(() => {
          this.containerWidth = this.$refs.scaner.clientWidth
        })
      }
      if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        this.previousCode = null
        this.parity = 0
        this.active = true
        this.canvas = this.$refs.canvas.getContext('2d')
        const facingMode = this.useBackCamera
          ? { exact: 'environment' }
          : 'user'
        const handleSuccess = stream => {
          if (this.$refs.video.srcObject !== undefined) {
            this.$refs.video.srcObject = stream
          } else if (window.videoEl.mozSrcObject !== undefined) {
            this.$refs.video.mozSrcObject = stream
          } else if (window.URL.createObjectURL) {
            this.$refs.video.src = window.URL.createObjectURL(stream)
          } else if (window.webkitURL) {
            this.$refs.video.src = window.webkitURL.createObjectURL(stream)
          } else {
            this.$refs.video.src = stream
          }
          this.$refs.video.playsInline = true
          const playPromise = this.$refs.video.play()
          playPromise.catch(() => (this.showPlay = true))
          playPromise.then(this.run)
        }
        navigator.mediaDevices
          .getUserMedia({ video: { facingMode } })
          .then(handleSuccess)
          .catch(() => {
            navigator.mediaDevices
              .getUserMedia({ video: true })
              .then(handleSuccess)
              .catch(error => {
                this.$emit('error-captured', error)
              })
          })
      } else {
        window.alert('获取数据失败')
      }
    },
    run() {
      if (this.active) {
        requestAnimationFrame(this.tick)
      }
    },
    found(code) {
      if (this.previousCode !== code) {
        this.previousCode = code
      } else if (this.previousCode === code) {
        this.parity += 1
      }
      if (this.parity > 2) {
        this.active = this.stopOnScanned ? false : true
        this.parity = 0
        this.$emit('code-scanned', code)
      }
    },
    // 完全停止
    fullStop() {
      if (this.$refs.video && this.$refs.video.srcObject) {
        this.$refs.video.srcObject.getTracks().forEach(t => t.stop())
      }
    },
  },
  mounted() {
    this.setup()
  },
  beforeDestroy() {
    this.fullStop()
  },
}
</script>

<style lang="css" scoped>
.btn {
  height: 50px;
  position: absolute;
  top: 85%;
  left: 50%;
  width: 240px;
  transform: translateX(-50%);
  background: #ffffff;
  border-radius: 8px;
  box-sizing: border-box;
  padding: 12px;
  opacity: 0.9;
  box-shadow: 1px 1px 10px rgba(0, 0, 0, 0.2);
  cursor: pointer;
  display: flex;
  gap: 4px;
  align-items: center;
  justify-content: space-between;
  padding: 0 20px 0 20px;
}
.btn .text {
  color: #000000;
  font-size: 16px;
  text-align: center;
  font-weight: bold;
}
.scaner {
  background: #000000;
  position: fixed;
  top: 0px;
  left: 0;
  width: 100%;
  height: 100%;
  height: -webkit-calc(100%);
  height: -moz-calc(100%);
  height: -ms-calc(100%);
  height: -o-calc(100%);
  height: calc(100%);
}
.scaner .banner {
  width: 240px;
  position: absolute;
  top: 16px;
  left: 50%;
  transform: translateX(-50%);
  background: #098907;
  border-radius: 8px;
  box-sizing: border-box;
  padding: 12px;
  opacity: 0.9;
  box-shadow: 1px 1px 10px rgba(0, 0, 0, 0.2);
}
.scaner .banner .text {
  padding: 0;
  margin: 0;
  color: #ffffff;
  font-size: 12px;
  text-align: justify;
  text-align-last: left;
}
.scaner .banner .close_icon {
  display: inline-block;
  height: 24px;
  width: 24px;
  background: url('../assets/close.png') no-repeat center;
  background-size: auto 100%;
  position: absolute;
  right: 8px;
  top: 8px;
}
.scaner .cover {
  height: 220px;
  width: 220px;
  position: absolute;
  top: 50%;
  left: 50%;
  -webkit-transform: translate(-50%, -50%);
  -moz-transform: translate(-50%, -50%);
  -ms-transform: translate(-50%, -50%);
  -o-transform: translate(-50%, -50%);
  transform: translate(-50%, -50%);
  border: 0.5px solid #999999;
  z-index: 1111;
}
.scaner .cover .line {
  width: 200px;
  height: 1px;
  margin-left: 10px;
  background: #5f68e8;
  background: linear-gradient(
    to right,
    transparent,
    #5f68e8,
    #0165ff,
    #5f68e8,
    transparent
  );
  position: absolute;
  -webkit-animation: scan 1.75s infinite linear;
  -moz-animation: scan 1.75s infinite linear;
  -ms-animation: scan 1.75s infinite linear;
  -o-animation: scan 1.75s infinite linear;
  animation: scan 1.75s infinite linear;
  -webkit-animation-fill-mode: both;
  -moz-animation-fill-mode: both;
  -ms-animation-fill-mode: both;
  -o-animation-fill-mode: both;
  animation-fill-mode: both;
  border-radius: 1px;
}
.scaner .cover .square {
  display: inline-block;
  height: 20px;
  width: 20px;
  position: absolute;
}
.scaner .cover .square.top {
  top: 0;
  border-top: 1px solid #5f68e8;
}
.scaner .cover .square.left {
  left: 0;
  border-left: 1px solid #5f68e8;
}
.scaner .cover .square.bottom {
  bottom: 0;
  border-bottom: 1px solid #5f68e8;
}
.scaner .cover .square.right {
  right: 0;
  border-right: 1px solid #5f68e8;
}
.scaner .cover .tips {
  position: absolute;
  bottom: 0px;
  width: 100%;
  font-size: 14px;
  color: #ffffff;
  opacity: 0.8;
}
@-webkit-keyframes scan {
  0% {
    top: 0;
  }
  25% {
    top: 50px;
  }
  50% {
    top: 100px;
  }
  75% {
    top: 150px;
  }
  100% {
    top: 200px;
  }
}
@-moz-keyframes scan {
  0% {
    top: 0;
  }
  25% {
    top: 50px;
  }
  50% {
    top: 100px;
  }
  75% {
    top: 150px;
  }
  100% {
    top: 200px;
  }
}
@-o-keyframes scan {
  0% {
    top: 0;
  }
  25% {
    top: 50px;
  }
  50% {
    top: 100px;
  }
  75% {
    top: 150px;
  }
  100% {
    top: 200px;
  }
}
@keyframes scan {
  0% {
    top: 0;
  }
  25% {
    top: 50px;
  }
  50% {
    top: 100px;
  }
  75% {
    top: 150px;
  }
  100% {
    top: 200px;
  }
}
</style>
