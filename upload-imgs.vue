<template>
<div class="uploadimgs">
  <div v-for="(item, i) in srcs" :key="i" :class="[promptText === '' ? 'img-item' : 'img-item no-border']">
    <img :src="item.base64 || item.uploadFileServerPath" @click="preview(i)">
    <span class="del" @click="del(i)" v-if="!disabled">
      <i class="iconfont-cha"></i>
    </span>

    <div class="prompt" v-if="promptText">
      <i class="nav-icon iconfont-sousuo"></i>{{promptText}}
    </div>
  </div>
  <!-- 上传功能 -->
  <div class="upload" v-show="moreUpload && !disabled">
    <input type="file" name="files" @change="upload" accept="image/*">
    <div>
      <i class="upload-icon iconfont-plus"></i>
      <p>{{title}}</p>
    </div>
  </div>
  <!-- 大图预览 -->
  <div v-transfer-dom>
    <previewer :list="previewerList" ref="previewer" ></previewer>
  </div>
</div>
</template>
<script>
import { Previewer, TransferDom } from 'vux'

// 上传时 前端预览
// 直接展示时 前端取后端地址
export default {
  components: {
    Previewer,
  },
  directives: {
    TransferDom
  },
  model: {
    prop: 'srcs',
    event: 'change',
  },
  props: {
    srcs: {
      type: Array,
      default: () => [],
    },
    title: {
      type: String,
      default: '上传截图'
    },
    maxFileSize: {
      type: Number,
      default: 1024 * 1024 * 10
    },
    // 最多上传
    max: {
      type: Number,
      default: 1,
    },
    // 最少上传
    min: {
      type: Number,
      default: 1,
    },
    // 展示型 true
    disabled: {
      type: Boolean,
      default: false,
    },
    promptText: {
      type: String,
      default: ''
    },
  },
  data () {
    return {
      innerAccepts: ['image/png', 'image/jpg', 'image/gif', 'image/jpeg', 'image/bmp', 'image/svg', 'image/raw'],
      fileEvent: null,
    }
  },
  computed: {
    previewerList() {
      return this.serviceImgs.map(ele => {
        const src = ele.base64 || ele.uploadFileServerPath
        return {
          src,
        }
      })
    },
    calcAccepts() {
      return (this.accepts || this.innerAccepts).join(',')
    },
    serviceImgs: {
      get() {
        return this.srcs
      },
      set(val) {
        // console.log('set serviceImgs', val)
        // this.$emit('change', val)
      },
    },
    moreUpload() {
      return this.serviceImgs.length < this.max
    },
  },
  mounted () {},
  methods: {
    upload(e) {
      this.fileEvent = e
      const file = e.target.files[0]
      // console.log(file)
      if (file.size > this.maxFileSize) {
        this.fileEvent.target.value = ''
        const maxSize = parseInt(this.maxFileSize / 1024 / 1024, 10)
        this.$vux.toast.show({
          text: `文件大小不能超过${maxSize}M`,
          type: 'warn'
        })
        return
      }
      this.fePreviewer(file)
    },
    preview(i) {
      this.$refs.previewer.show(i)
    },
    fePreviewer(file) {
      // 前端预览实现
      const that = this
      const reader = new FileReader()
      reader.readAsDataURL(file)
      reader.onloadend = function(e) {
        // console.log('base64生成', e)
        const item = {
          base64: reader.result,
          uploadFilePath: '',
          uploadFileServerPath: '',
        }
        that.serviceImgs.push(item)
        // console.log(that.serviceImgs)
        that.postImg(file)
      }
    },
    postImg(file) {
      const body = new FormData()
      body.append('files', file)
      body.append('fileType', 1)
      const L = this.serviceImgs.length > 0 ? this.serviceImgs.length - 1 : 0
      this.$vux.loading.show({
        text: '正在上传...'
      })
      this.$axios.post('/api/ue/uploadFile', body, {
        headers: { 'Content-Type': 'multipart/form-data' },
        onUploadProgress(e) {
          console.log(e.loaded, e.total)
        },
      }).then(res => {
        this.$vux.loading.hide()
        this.fileEvent.target.value = ''
        if (res.data.code === 1 && res.status === 200) {
          this.$vux.toast.show({
            text: '上传成功',
            type: 'success'
          })
          const { uploadFilePath, uploadFileServerPath } = res.data.data
          this.serviceImgs[L].uploadFilePath = uploadFilePath
          this.serviceImgs[L].uploadFileServerPath = uploadFileServerPath
        } else {
          this.fileEvent.target.value = ''
          this.serviceImgs.splice(L, 1)
          this.$vux.toast.show({
            text: res.data.msg || '上传失败',
            type: 'warn'
          })
        }
      }).catch(() => {
        this.serviceImgs.splice(L, 1)
        this.$vux.loading.hide()
        this.fileEvent.target.value = ''
        this.$vux.toast.show({
          text: '上传失败',
          type: 'warn'
        })
      })
    },
    del(i) {
      this.serviceImgs.splice(i, 1)
    },
  },
}
</script>
<style lang="less" scoped>
.uploadimgs {
  display: inline-block;
  .img-item {
    display: inline-block;
    width: 180px/2;
    height: 180px/2;
    border: 2px/2 dashed #d8d8d8;
    border-radius: 10px/2;
    margin-right: 20px/2;
    margin-bottom: 20px/2;
    vertical-align: top;
    position: relative;
    &.no-border {
      border: none;
    }
    &:last-child {
      margin-right: 0;
    }
    img {
      width: 100%;
      height: 100%;
    }
    span.del {
      position: absolute;
      right: -15px/2;
      top: -15px/2;
      width: 30px/2;
      height: 30px/2;
      border-radius: 50%;
      background-color: #000;
      text-align: center;
      line-height: 26px/2;
      i {
        font-size: 12px/2;
        color: #fff;
      }
    }
    .prompt {
      position: absolute;
      bottom: 0;
      left: 0;
      right: 0;
      height: 40px/2;
      line-height: 40px/2;
      color: #fff;
      text-align: center;
      background-color: #FB625B;
      // border-radius: 10px/2;
      border-bottom-left-radius: 10px/2;
      border-bottom-right-radius: 10px/2;
      font-size: 20px/2;
      i {
        font-size: 20px/2;
      }
    }
  }
  .upload {
    vertical-align: top;
    display: inline-block;
    width: 180px/2;
    height: 180px/2;
    border: 2px/2 dashed #d8d8d8;
    border-radius: 10px/2;
    text-align: center;
    position: relative;
    input {
      position: absolute;
      top:0;
      left: 0;
      width: 100%;
      height: 100%;
      opacity: 0;
    }
    i {
      font-size: 100px/2;
      color: #d8d8d8;
    }
  }
}
</style>