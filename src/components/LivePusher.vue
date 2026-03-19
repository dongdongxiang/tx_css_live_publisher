<template>
  <div class="live-pusher">

    <!-- ── 左侧配置面板 ── -->
    <div class="config-panel">

      <!-- 采集源选择 -->
      <div class="section">
        <div class="source-tabs">
          <button
            v-for="src in sourceOptions"
            :key="src.value"
            class="source-btn"
            :class="{ active: sourceType === src.value }"
            @click="switchSource(src.value)"
          >{{ src.label }}</button>
        </div>
        <p class="source-desc">{{ currentSourceDesc }}</p>
      </div>

      <!-- 设备选择：摄像头模式 -->
      <div v-if="sourceType === 'camera'" class="section">
        <div class="section-title">设备选择</div>
        <div class="device-row">
          <span class="device-label">摄像头设备</span>
          <select v-model="selectedCamera" @change="onCameraChange" class="device-select">
            <option value="">-- 请选择 --</option>
            <option v-for="d in cameraDevices" :key="d.deviceId" :value="d.deviceId">
              {{ d.label || '摄像头 ' + d.deviceId.slice(0, 8) }}
            </option>
          </select>
        </div>
        <div class="device-row">
          <span class="device-label">麦克风设备</span>
          <select v-model="selectedMic" @change="onMicChange" class="device-select">
            <option value="">-- 请选择 --</option>
            <option v-for="d in micDevices" :key="d.deviceId" :value="d.deviceId">
              {{ d.label || '麦克风 ' + d.deviceId.slice(0, 8) }}
            </option>
          </select>
        </div>
        <div class="device-actions">
          <button class="btn-sm" @click="openCameraAndMic" :disabled="cameraOpen">开启摄像头+麦克风</button>
          <button class="btn-sm btn-danger" @click="closeCamera" :disabled="!cameraOpen">关闭摄像头</button>
          <button class="btn-sm btn-danger" @click="closeMic" :disabled="!micOpen">关闭麦克风</button>
        </div>
      </div>

      <!-- 设备选择：屏幕共享模式 -->
      <div v-if="sourceType === 'screen'" class="section">
        <div class="section-title">屏幕采集</div>
        <div class="device-row">
          <span class="device-label">麦克风设备</span>
          <select v-model="selectedMic" @change="onMicChange" class="device-select">
            <option value="">-- 请选择 --</option>
            <option v-for="d in micDevices" :key="d.deviceId" :value="d.deviceId">
              {{ d.label || '麦克风 ' + d.deviceId.slice(0, 8) }}
            </option>
          </select>
        </div>

        <!-- 画中画选项 -->
        <div class="pip-option">
          <label class="checkbox-label">
            <input type="checkbox" v-model="pipEnabled" :disabled="screenOpen" />
            <span>右下角显示摄像头画面（画中画）</span>
          </label>
        </div>
        <template v-if="pipEnabled">
          <div class="device-row">
            <span class="device-label">摄像头设备</span>
            <select v-model="selectedCamera" class="device-select" :disabled="screenOpen">
              <option value="">-- 请选择 --</option>
              <option v-for="d in cameraDevices" :key="d.deviceId" :value="d.deviceId">
                {{ d.label || '摄像头 ' + d.deviceId.slice(0, 8) }}
              </option>
            </select>
          </div>
          <div class="pip-config">
            <span class="config-key">小窗尺寸</span>
            <select v-model="pipSize" class="device-select" style="width:120px">
              <option value="small">小（1/5 宽）</option>
              <option value="medium">中（1/4 宽）</option>
              <option value="large">大（1/3 宽）</option>
            </select>
          </div>
        </template>

        <div class="device-actions">
          <button class="btn-sm" @click="openScreen" :disabled="screenOpen">开启屏幕共享</button>
          <button class="btn-sm" @click="openMic" :disabled="micOpen">开启麦克风</button>
          <button class="btn-sm btn-danger" @click="stopScreen" :disabled="!screenOpen">停止共享</button>
        </div>
      </div>

      <!-- 采集配置 -->
      <div class="section">
        <div class="section-header">
          <span class="section-title">采集配置</span>
          <span class="link" @click="showCaptureEdit = !showCaptureEdit">编辑</span>
        </div>
        <template v-if="!showCaptureEdit">
          <div class="config-row">
            <span class="config-key">采集质量</span>
            <span class="config-val">推荐配置 - {{ capturePreset }}</span>
          </div>
          <div class="config-row">
            <span class="config-key">视频质量</span>
            <span class="config-val">{{ videoQualityLabel }}</span>
          </div>
          <div class="config-row">
            <span class="config-key">音频质量</span>
            <span class="config-val">{{ audioQualityLabel }}</span>
          </div>
        </template>
        <template v-else>
          <div class="config-row">
            <span class="config-key">视频质量</span>
            <select v-model="videoQuality" class="device-select">
              <option value="120p">120P（低清）</option>
              <option value="180p">180P</option>
              <option value="240p">240P</option>
              <option value="360p">360P（标清）</option>
              <option value="480p">480P</option>
              <option value="720p">720P（高清）</option>
              <option value="1080p">1080P（超清）</option>
            </select>
          </div>
          <div class="config-row">
            <span class="config-key">音频质量</span>
            <select v-model="audioQuality" class="device-select">
              <option value="standard">标准（48kHz 单声道）</option>
              <option value="high">高清（48kHz 立体声）</option>
            </select>
          </div>
          <div class="config-row">
            <span class="config-key">自定义帧率</span>
            <input type="number" v-model.number="customFPS" class="num-input" min="1" max="60" />
            <span style="font-size:12px;color:#999">fps（0=不设置）</span>
          </div>
        </template>
      </div>

      <!-- 推流统计 -->
      <div v-if="isPushing && stats" class="section stats-section">
        <div class="section-title">推流统计</div>
        <div class="config-row">
          <span class="config-key">视频帧率</span>
          <span class="config-val">{{ stats.video && stats.video.framesPerSecond }} fps</span>
        </div>
        <div class="config-row">
          <span class="config-key">视频码率</span>
          <span class="config-val">{{ stats.video && stats.video.bitrate }} kbps</span>
        </div>
        <div class="config-row">
          <span class="config-key">音频码率</span>
          <span class="config-val">{{ stats.audio && stats.audio.bitrate }} kbps</span>
        </div>
        <div class="config-row">
          <span class="config-key">推流状态</span>
          <span class="config-val" :class="pushStatusClass">{{ pushStatusText }}</span>
        </div>
      </div>

      <!-- 错误信息 -->
      <div v-if="errorMsg" class="error-bar">{{ errorMsg }}</div>
    </div>

    <!-- ── 右侧预览 + 控制 ── -->
    <div class="preview-panel">
      <!-- SDK 渲染容器，TXLivePusher 会在此 div 内注入 video 元素 -->
      <div
        id="local_video"
        class="preview-box"
      >
        <div v-if="!cameraOpen && !screenOpen" class="preview-placeholder">预览画面</div>
        <div v-if="isPushing" class="live-badge">LIVE</div>
      </div>

      <!-- 推流地址 -->
      <div class="url-row">
        <input
          v-model="pushUrl"
          class="url-input"
          placeholder="请输入 WebRTC 推流地址  webrtc://domain/AppName/StreamName?txSecret=xxx&txTime=xxx"
          :disabled="isPushing"
        />
        <span class="link" @click="showUrlHelper = true">快速生成</span>
      </div>

      <!-- 控制按钮 -->
      <div class="control-row">
        <button
          class="btn-push"
          :class="{ pushing: isPushing }"
          :disabled="!canPush"
          @click="togglePush"
        >{{ isPushing ? '停止推流' : '开始推流' }}</button>

        <button class="btn-ctrl" @click="toggleVideo" :disabled="!isPushing">
          {{ videoEnabled ? '🖥 关闭画面' : '🖥 开启画面' }}
        </button>

        <button class="btn-ctrl" @click="toggleAudio" :disabled="!isPushing">
          {{ audioEnabled ? '🎤 关闭声音' : '🎤 开启声音' }}
        </button>
      </div>
    </div>

    <!-- ── 快速生成推流地址弹窗 ── -->
    <div v-if="showUrlHelper" class="modal-mask" @click.self="showUrlHelper = false">
      <div class="modal">
        <div class="modal-title">快速生成 WebRTC 推流地址</div>
        <div class="form-row">
          <label>推流域名</label>
          <input v-model="urlHelper.domain" placeholder="push.example.com" />
        </div>
        <div class="form-row">
          <label>AppName</label>
          <input v-model="urlHelper.appName" placeholder="live" />
        </div>
        <div class="form-row">
          <label>StreamName</label>
          <input v-model="urlHelper.streamName" placeholder="stream001" />
        </div>
        <div class="form-row">
          <label>鉴权Key（可选）</label>
          <input v-model="urlHelper.authKey" placeholder="推流鉴权 Key" />
        </div>
        <div class="form-row">
          <label>有效期(秒)</label>
          <input type="number" v-model.number="urlHelper.expire" placeholder="3600" />
        </div>
        <div class="modal-actions">
          <button class="btn-primary" @click="generateUrl">生成地址</button>
          <button class="btn-cancel" @click="showUrlHelper = false">取消</button>
        </div>
        <div v-if="generatedUrl" class="generated-url">
          <span class="generated-url-text">{{ generatedUrl }}</span>
          <span class="link" @click="applyGeneratedUrl">复制并填入</span>
        </div>
      </div>
    </div>

  </div>
</template>

<script>
export default {
  name: 'LivePusher',

  props: {
    defaultPushUrl: { type: String, default: '' }
  },

  data() {
    return {
      // SDK 实例
      livePusher: null,
      sdkReady: false,

      // 采集源
      sourceType: 'camera',

      // 设备列表
      cameraDevices: [],
      micDevices: [],
      selectedCamera: '',
      selectedMic: '',

      // 采集状态
      cameraOpen: false,
      micOpen: false,
      screenOpen: false,

      // 画中画
      pipEnabled: false,
      pipSize: 'small',       // small | medium | large
      pipCameraStreamId: '',  // SDK 返回的摄像头 streamId
      pipScreenStreamId: '',  // SDK 返回的屏幕 streamId

      // 采集配置
      showCaptureEdit: false,
      capturePreset: '720P',
      videoQuality: '720p',
      audioQuality: 'standard',
      customFPS: 0,

      // 推流状态
      isPushing: false,
      videoEnabled: true,
      audioEnabled: true,
      pushStatus: '',   // SDK 回调的状态码
      stats: null,

      // 推流地址
      pushUrl: '',

      // 快速生成弹窗
      showUrlHelper: false,
      urlHelper: { domain: '', appName: 'live', streamName: '', authKey: '', expire: 3600 },
      generatedUrl: '',

      errorMsg: ''
    }
  },

  computed: {
    sourceOptions() {
      return [
        { value: 'camera', label: '摄像头采集' },
        { value: 'screen', label: '屏幕分享采集' }
      ]
    },
    currentSourceDesc() {
      return this.sourceType === 'camera'
        ? '通过摄像头/麦克风（支持外接）进行视频/声音的采集。'
        : '采集屏幕画面，可同时混入麦克风声音。'
    },
    videoQualityLabel() {
      const map = {
        '120p': '120P（低清）', '180p': '180P', '240p': '240P',
        '360p': '360P（标清）', '480p': '480P',
        '720p': '720P（高清）', '1080p': '1080P（超清）'
      }
      return map[this.videoQuality] || this.videoQuality
    },
    audioQualityLabel() {
      return this.audioQuality === 'high' ? '高清（48kHz 立体声）' : '标准（48kHz 单声道）'
    },
    // 只要有推流地址就可以点"开始推流"（采集已在开启摄像头/屏幕时建立）
    canPush() {
      return !!this.pushUrl && this.sdkReady && (this.cameraOpen || this.screenOpen)
    },
    pushStatusText() {
      const map = {
        'live':         '推流中',
        'interrupted':  '连接中断',
        'stopped':      '已停止',
        'failed':       '推流失败'
      }
      return map[this.pushStatus] || this.pushStatus || '推流中'
    },
    pushStatusClass() {
      return {
        'status-live':   this.pushStatus === 'live',
        'status-error':  ['interrupted', 'failed'].includes(this.pushStatus)
      }
    }
  },

  mounted() {
    if (this.defaultPushUrl) this.pushUrl = this.defaultPushUrl
    this.initSDK()
  },

  beforeUnmount() {
    this.destroySDK()
  },

  methods: {
    // ── SDK 初始化 ──────────────────────────────────────────────
    async initSDK() {
      if (typeof TXLivePusher === 'undefined') {
        this.setError('TXLivePusher SDK 未加载，请检查网络或 index.html 中的 script 引入')
        return
      }

      // 检查浏览器兼容性
      TXLivePusher.checkSupport().then(data => {
        if (!data.isWebRTCSupported) {
          this.setError('当前浏览器不支持 WebRTC，请使用 Chrome / Edge / Firefox / Safari')
          return
        }
        if (!data.isH264EncodeSupported) {
          this.setError('当前浏览器不支持 H.264 编码')
          return
        }

        this.livePusher = new TXLivePusher()

        // 指定预览容器
        this.livePusher.setRenderView('local_video')
        // 预览静音（避免回声，不影响推流音频）
        const videoEl = document.querySelector('#local_video video')
        if (videoEl) videoEl.muted = true

        // 注册回调
        this.livePusher.setObserver({
          onPushStatusUpdate: (status, message) => {
            this.pushStatus = status
            if (['interrupted', 'failed'].includes(status)) {
              this.isPushing = false
              this.setError(`推流${status === 'failed' ? '失败' : '中断'}：${message}`)
            }
            if (status === 'stopped') {
              this.isPushing = false
            }
          },
          onStatisticsUpdate: (data) => {
            this.stats = data
          },
          onError: (errCode, errMsg) => {
            this.setError(`错误 [${errCode}]：${errMsg}`)
          }
        })

        this.sdkReady = true
        this.enumerateDevices()
      })
    },

    destroySDK() {
      if (this.livePusher) {
        try {
          if (this.isPushing) this.livePusher.stopPush()
          this.stopPip()
          this.livePusher.stopCamera()
          this.livePusher.stopMicrophone()
          this.livePusher.stopScreenCapture && this.livePusher.stopScreenCapture()
        } catch (_) {}
        this.livePusher = null
      }
    },

    // ── 设备枚举 ────────────────────────────────────────────────
    async enumerateDevices() {
      try {
        // 先请求权限以获取设备名称
        const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true })
        stream.getTracks().forEach(t => t.stop())
      } catch (_) {}
      try {
        const devices = await navigator.mediaDevices.enumerateDevices()
        this.cameraDevices = devices.filter(d => d.kind === 'videoinput')
        this.micDevices    = devices.filter(d => d.kind === 'audioinput')
        if (this.cameraDevices.length) this.selectedCamera = this.cameraDevices[0].deviceId
        if (this.micDevices.length)    this.selectedMic    = this.micDevices[0].deviceId
      } catch (_) {}
    },

    // ── 采集源切换 ──────────────────────────────────────────────
    async switchSource(type) {
      if (this.isPushing) {
        this.setError('请先停止推流再切换采集源')
        return
      }
      // 停止当前采集
      this.closeCamera()
      this.closeMic()
      this.stopScreen()
      this.sourceType = type
    },

    // ── 摄像头 + 麦克风 ─────────────────────────────────────────
    async openCameraAndMic() {
      if (!this.sdkReady) return
      this.clearError()
      this.applyQualitySettings()

      try {
        // 同时开启摄像头和麦克风，都成功后才算就绪
        await Promise.all([
          this.livePusher.startCamera(),
          this.livePusher.startMicrophone()
        ])
        this.cameraOpen = true
        this.micOpen    = true

        // 如指定了具体设备则切换
        if (this.selectedCamera) {
          const dm = this.livePusher.getDeviceManager()
          dm.switchCamera(this.selectedCamera)
        }
        if (this.selectedMic) {
          const dm = this.livePusher.getDeviceManager()
          dm.switchMicrophone && dm.switchMicrophone(this.selectedMic)
        }
      } catch (e) {
        this.setError('开启摄像头/麦克风失败：' + e.toString())
      }
    },

    async openMic() {
      if (!this.sdkReady) return
      this.clearError()
      try {
        await this.livePusher.startMicrophone()
        this.micOpen = true
      } catch (e) {
        this.setError('开启麦克风失败：' + e.toString())
      }
    },

    closeCamera() {
      if (this.livePusher && this.cameraOpen) {
        this.livePusher.stopCamera()
        this.cameraOpen = false
      }
    },

    closeMic() {
      if (this.livePusher && this.micOpen) {
        this.livePusher.stopMicrophone()
        this.micOpen = false
      }
    },

    onCameraChange() {
      if (!this.cameraOpen || !this.livePusher) return
      const dm = this.livePusher.getDeviceManager()
      dm.switchCamera(this.selectedCamera)
    },

    onMicChange() {
      if (!this.micOpen || !this.livePusher) return
      const dm = this.livePusher.getDeviceManager()
      dm.switchMicrophone && dm.switchMicrophone(this.selectedMic)
    },

    // ── 屏幕共享 ────────────────────────────────────────────────
    async openScreen() {
      if (!this.sdkReady) return
      this.clearError()
      this.applyQualitySettings()
      try {
        if (this.pipEnabled) {
          const vem = this.livePusher.getVideoEffectManager()
          vem.enableMixing(true)

          const cameraStreamId = await this.livePusher.startCamera(
            this.selectedCamera || undefined
          )
          this.cameraOpen = true

          const screenStreamId = await this.livePusher.startScreenCapture()
          this.screenOpen = true

          // 用 videoQuality 决定混流输出分辨率，与 setVideoQuality 保持一致
          const qualitySizeMap = {
            '120p':  { w: 160,  h: 120  },
            '180p':  { w: 320,  h: 180  },
            '240p':  { w: 320,  h: 240  },
            '360p':  { w: 640,  h: 360  },
            '480p':  { w: 640,  h: 480  },
            '720p':  { w: 1280, h: 720  },
            '1080p': { w: 1920, h: 1080 }
          }
          const qs   = qualitySizeMap[this.videoQuality] || { w: 1280, h: 720 }
          const outW = qs.w
          const outH = qs.h

          console.log('[PiP] output size:', outW, 'x', outH, '| cameraId:', cameraStreamId, '| screenId:', screenStreamId)

          this.applyPipMixing(vem, cameraStreamId, screenStreamId, outW, outH)
        } else {
          const screenStreamId = await this.livePusher.startScreenCapture()
          this.screenOpen = true
          console.log('[no-pip] screenStreamId:', screenStreamId)
        }
      } catch (e) {
        if (!e.toString().includes('NotAllowed')) {
          this.setError('屏幕共享失败：' + e.toString())
        }
      }
    },

    stopScreen() {
      if (!this.livePusher || !this.screenOpen) return
      this.stopPip()
      this.livePusher.stopScreenCapture && this.livePusher.stopScreenCapture()
      this.screenOpen = false
    },

    // ── 画中画混流 ───────────────────────────────────────────────
    applyPipMixing(vem, cameraStreamId, screenStreamId, outW = 1280, outH = 720) {
      this.pipCameraStreamId = cameraStreamId
      this.pipScreenStreamId = screenStreamId

      const ratioMap = { small: 0.2, medium: 0.25, large: 0.33 }
      const ratio    = ratioMap[this.pipSize] || 0.2
      const margin   = 16

      const pipW = Math.round(outW * ratio)
      const pipH = Math.round(pipW * 9 / 16)

      // x、y 是中心点坐标
      // 屏幕流：中心 = (outW/2, outH/2)
      // 摄像头小窗：中心 = (outW - margin - pipW/2, outH - margin - pipH/2)
      const pipCX = outW - margin - pipW / 2
      const pipCY = outH - margin - pipH / 2

      vem.setLayout([
        {
          streamId: screenStreamId,
          x:        outW / 2,
          y:        outH / 2,
          width:    outW,
          height:   outH,
          zOrder:   0
        },
        {
          streamId: cameraStreamId,
          x:        pipCX,
          y:        pipCY,
          width:    pipW,
          height:   pipH,
          zOrder:   1
        }
      ])
    },

    stopPip() {
      if (!this.livePusher) return
      try {
        const vem = this.livePusher.getVideoEffectManager()
        vem.enableMixing(false)
      } catch (_) {}
      // 画中画开启的摄像头单独关闭
      if (this.cameraOpen) {
        try { this.livePusher.stopCamera() } catch (_) {}
        this.cameraOpen = false
      }
    },

    // ── 质量设置（推流前应用） ───────────────────────────────────
    applyQualitySettings() {
      if (!this.livePusher) return
      this.livePusher.setVideoQuality(this.videoQuality)
      this.livePusher.setAudioQuality(this.audioQuality)
      if (this.customFPS > 0) {
        this.livePusher.setProperty('setVideoFPS', this.customFPS)
      }
    },

    // ── 推流控制 ────────────────────────────────────────────────
    async togglePush() {
      if (this.isPushing) {
        this.stopPush()
      } else {
        await this.startPush()
      }
    },

    async startPush() {
      if (!this.pushUrl || !this.livePusher) return
      this.clearError()
      this.applyQualitySettings()
      try {
        await this.livePusher.startPush(this.pushUrl)
        this.isPushing    = true
        this.videoEnabled = true
        this.audioEnabled = true
        this.pushStatus   = 'live'
      } catch (e) {
        this.setError('推流失败：' + e.toString())
      }
    },

    stopPush() {
      if (this.livePusher) {
        this.livePusher.stopPush()
      }
      this.isPushing  = false
      this.pushStatus = 'stopped'
      this.stats      = null
    },

    // ── 推流中控制 ───────────────────────────────────────────────
    toggleVideo() {
      if (!this.livePusher || !this.isPushing) return
      this.videoEnabled = !this.videoEnabled
      // SDK 暂无直接 muteVideo，通过 track enabled 实现
      const videoTracks = this.livePusher.getVideoTrack
        ? [this.livePusher.getVideoTrack()]
        : []
      videoTracks.forEach(t => t && (t.enabled = this.videoEnabled))
    },

    toggleAudio() {
      if (!this.livePusher || !this.isPushing) return
      this.audioEnabled = !this.audioEnabled
      const audioTracks = this.livePusher.getAudioTrack
        ? [this.livePusher.getAudioTrack()]
        : []
      audioTracks.forEach(t => t && (t.enabled = this.audioEnabled))
    },

    // ── 快速生成推流地址 ─────────────────────────────────────────
    generateUrl() {
      const { domain, appName, streamName, authKey, expire } = this.urlHelper
      if (!domain || !streamName) {
        this.setError('请填写推流域名和流名称')
        return
      }
      let url = `webrtc://${domain}/${appName}/${streamName}`
      if (authKey) {
        const txTime   = Math.floor(Date.now() / 1000 + expire).toString(16).toUpperCase()
        const raw      = authKey + streamName + txTime
        const txSecret = this.md5(raw)
        url += `?txSecret=${txSecret}&txTime=${txTime}`
      }
      this.generatedUrl = url
    },

    applyGeneratedUrl() {
      this.pushUrl = this.generatedUrl
      navigator.clipboard?.writeText(this.generatedUrl)
      this.showUrlHelper = false
    },

    // 简易 MD5（用于鉴权签名，仅在客户端演示用）
    md5(str) {
      // 若引入了外部 MD5 库可替换此处；此处返回原始拼接仅作占位，实际应使用 MD5 库
      // 推荐: npm install md5  然后 import md5 from 'md5'
      if (typeof window.md5 === 'function') return window.md5(str)
      console.warn('未找到 MD5 函数，txSecret 未正确计算，请引入 md5 库')
      return str
    },

    // ── 错误处理 ────────────────────────────────────────────────
    setError(msg) {
      this.errorMsg = msg
      setTimeout(() => { this.errorMsg = '' }, 6000)
    },
    clearError() {
      this.errorMsg = ''
    }
  }
}
</script>

<style scoped>
.live-pusher {
  display: flex;
  gap: 24px;
  padding: 24px;
  background: #f5f6fa;
  min-height: 100vh;
  font-size: 14px;
  color: #333;
  box-sizing: border-box;
}

/* ── 左侧 ── */
.config-panel {
  width: 380px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.section {
  background: #fff;
  border-radius: 8px;
  padding: 16px;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.section-title {
  font-weight: 600;
  font-size: 15px;
  margin-bottom: 12px;
  display: block;
}

.section-header .section-title { margin-bottom: 0; }

.source-tabs {
  display: flex;
  gap: 8px;
  margin-bottom: 10px;
}

.source-btn {
  padding: 6px 16px;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  background: #fff;
  cursor: pointer;
  font-size: 13px;
  transition: all .2s;
}

.source-btn.active {
  border-color: #0052d9;
  color: #0052d9;
  background: #e8f0fe;
}

.source-btn:hover:not(.active) { border-color: #0052d9; color: #0052d9; }

.source-desc { margin: 0; color: #666; font-size: 13px; }

.device-row {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
}

.device-label {
  width: 80px;
  color: #888;
  flex-shrink: 0;
  font-size: 13px;
}

.device-select {
  flex: 1;
  padding: 5px 8px;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  font-size: 13px;
}

.device-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  margin-top: 4px;
}

.btn-sm {
  padding: 5px 12px;
  font-size: 12px;
  border: 1px solid #0052d9;
  color: #0052d9;
  background: #fff;
  border-radius: 4px;
  cursor: pointer;
  transition: all .2s;
}

.btn-sm:hover:not(:disabled) { background: #0052d9; color: #fff; }
.btn-sm:disabled { border-color: #ccc; color: #ccc; cursor: not-allowed; }
.btn-sm.btn-danger { border-color: #e34d59; color: #e34d59; }
.btn-sm.btn-danger:hover:not(:disabled) { background: #e34d59; color: #fff; }

.config-row {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-bottom: 8px;
  font-size: 13px;
}

.config-key { width: 90px; color: #888; flex-shrink: 0; }
.config-val { color: #333; }

.num-input {
  width: 70px;
  padding: 4px 8px;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  font-size: 13px;
}

.link { color: #0052d9; cursor: pointer; font-size: 13px; }
.link:hover { text-decoration: underline; }

.stats-section { border-left: 3px solid #0052d9; }

.status-live  { color: #52c41a; font-weight: 600; }
.status-error { color: #e34d59; font-weight: 600; }

.error-bar {
  background: #fff3f3;
  border: 1px solid #e34d59;
  border-radius: 6px;
  padding: 10px 14px;
  color: #e34d59;
  font-size: 13px;
  line-height: 1.5;
}

/* ── 右侧 ── */
.preview-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

/* TXLivePusher 的预览容器，SDK 会向内注入 <video> */
.preview-box {
  position: relative;
  background: #1a1a2e;
  border-radius: 8px;
  overflow: hidden;
  aspect-ratio: 16 / 9;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* SDK 注入的 video 元素绝对定位铺满容器 */
:deep(#local_video video) {
  position: absolute;
  inset: 0;
  width: 100% !important;
  height: 100% !important;
  object-fit: contain;
  display: block;
}

.preview-placeholder {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: #555;
  font-size: 15px;
  pointer-events: none;
}

.live-badge {
  position: absolute;
  top: 12px;
  left: 12px;
  background: #e34d59;
  color: #fff;
  font-size: 12px;
  font-weight: 700;
  padding: 2px 8px;
  border-radius: 4px;
  letter-spacing: 1px;
}

.url-row {
  display: flex;
  align-items: center;
  gap: 12px;
  background: #fff;
  border-radius: 8px;
  padding: 10px 16px;
  border: 1px solid #e8e8e8;
}

.url-input {
  flex: 1;
  border: none;
  outline: none;
  font-size: 13px;
  color: #333;
  background: transparent;
}

.url-input::placeholder { color: #bbb; }

.control-row {
  display: flex;
  align-items: center;
  gap: 12px;
}

.btn-push {
  padding: 8px 28px;
  background: #0052d9;
  color: #fff;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
  transition: background .2s;
}

.btn-push.pushing { background: #e34d59; }
.btn-push:disabled { background: #c0c4cc; cursor: not-allowed; }

.btn-ctrl {
  padding: 7px 16px;
  background: #fff;
  border: 1px solid #d9d9d9;
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
  transition: all .2s;
}

.btn-ctrl:hover:not(:disabled) { border-color: #0052d9; color: #0052d9; }
.btn-ctrl:disabled { color: #c0c4cc; cursor: not-allowed; }

/* ── 弹窗 ── */
.modal-mask {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,.45);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  background: #fff;
  border-radius: 10px;
  padding: 28px 32px;
  width: 440px;
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.modal-title { font-size: 16px; font-weight: 600; }

.form-row { display: flex; align-items: center; gap: 12px; }

.form-row label {
  width: 100px;
  color: #666;
  flex-shrink: 0;
  font-size: 13px;
}

.form-row input {
  flex: 1;
  padding: 6px 10px;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  font-size: 13px;
}

.modal-actions { display: flex; gap: 12px; margin-top: 4px; }

.btn-primary {
  padding: 7px 22px;
  background: #0052d9;
  color: #fff;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
}

.btn-cancel {
  padding: 7px 22px;
  background: #f5f5f5;
  border: 1px solid #d9d9d9;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
}

.generated-url {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  background: #f5f6fa;
  border-radius: 6px;
  padding: 10px 12px;
}

.generated-url-text {
  flex: 1;
  font-size: 12px;
  color: #555;
  word-break: break-all;
  line-height: 1.6;
}

/* ── 画中画 ── */
.pip-option {
  margin: 8px 0 10px;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  font-size: 13px;
  color: #333;
  user-select: none;
}

.checkbox-label input[type="checkbox"] {
  width: 15px;
  height: 15px;
  cursor: pointer;
  accent-color: #0052d9;
}

.checkbox-label input[type="checkbox"]:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}

.pip-config {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
  font-size: 13px;
}

.pip-config .config-key {
  width: 60px;
  color: #888;
  flex-shrink: 0;
}
</style>
