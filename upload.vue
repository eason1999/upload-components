<template>
  <div class="upload-componet" >
    <canvas ref="canvas" class="compress-canvas"></canvas>
    <div class="img-wrap clearfix">
      <div class="fl imgs" v-for="(img,i) in viewImgs" :key="i">
        <img :src="img" alt="" preview="0" preview-text="描述文字">
        <i class="el-icon-error del-btn" @click="delImg(i)" v-if="!disabled"></i>
        <div class="set-main-btn">
          <el-button type="text" size="mini" @click="setMain(i)" v-show="!disabled">设为主图</el-button>
        </div>
      </div>
      <div class="fl file-wrap center-parent" v-show="ableFile && !disabled">
        <input type="file" name="file" @change="upload">
        <i class="el-icon-plus upload-icon center-child"></i>
      </div>
    </div>
    
    <!-- <p v-show="Processing!=0">正在上传{{Processing}}%</p> -->
    
    <el-dialog
      title="裁剪图片"
      :visible.sync="dialogVisible"
      width="440px"
      v-if="iscut"
      @close="clearFile"
    >
      <div class="cut-img-wrap" >
        <vueCropper
          ref="cropper"
          :img="option.img"
          :outputSize="option.size"
          :outputType="option.outputType"
          :info="true"
          @realTime="realTime"
          :full="option.full"
          :autoCrop="option.autoCrop"
          :autoCropWidth="option.autoCropWidth"
          :autoCropHeight="option.autoCropHeight"
          :fixedBox="option.fixedBox"
        ></vueCropper>
        <div class="opt-btns">
          <i class="el-icon-error" @click="clearFile"></i>
          <i class="el-icon-success" @click="sureCut"></i>
        </div>
      </div>
    </el-dialog>
    
  </div>
</template>
<script>
import vueCropper from 'vue-cropper'

/* eslint-disable */
// let srcs = [
//   {
//     uploadFileServerPath: '',
//     uploadFilePath: ''
//   }
// ]
export default {
  name: "uploadImg",
  model:{
    prop:'srcs'
  },
  props:{
    // 修改顺序 （设置主图功能）
    changeSort: {
      type:Boolean,
      default:true
    },
    // 裁切功能
    iscut:{
      type:Boolean,
      default:false
    },
    // 图片最多张数
    max:{
      type:Number,
      default:1
    },
    // 图片links
    srcs:{
      // type:String || Array,
      default: function() {
        return []
      }
    },
    // 图片裁切宽
    width: {
      type: Number,
      default: 200
    },
    // 图片裁切高
    height: {
      type: Number,
      default: 200
    },
    // 图片可操作？
    disabled: {
      type: Boolean,
      default: false
    }
    // min:{
    //   type:Number,
    //   default:1
    // },
  },
  components:{
    vueCropper
  },
  data () {
    return {
      receiveType: 'string',
      ableServe:true,
      dialogVisible:false,
      defaultImg: '',
      fileEvent:null,
      url:'common/image?dir=1',
      maxSize:4*1024*1024, //kb为单位
      maxSizes: 4,
      Processing:0,
      serviceImg:'',
      viewImgs: [],
      // serviceImgs:[],
      option: {
				img: '',
				size: 1,
        outputType: 'jpeg',
        full:true,
        autoCrop:true,
        autoCropWidth: 200,
        autoCropHeight: 200,
        fixedBox: true,
        previews:{}
      },
    };
  },
  computed:{
    ableFile(){
      return this.serviceImgs.length < this.max && this.serviceImgs.length > -1
    },
    serviceImgs: {
      get() {
        return this.srcs
      },
      set(val) {
        this.viewImgs = val.map(ele => ele.uploadFileServerPath)
        this.$emit('input',val)
      }
    }
  },
  watch:{
    // serviceImgs:function(val){
    //   console.log('图片组件传递值：',val)
    //   this.$emit('input',val)
    // }, 
    srcs:function(val) {
      if (typeof val === 'string') {
        console.log('---请传递数组结构---')
      } else {
        this.serviceImgs = val
      }
    }
  },
  mounted(){
    this.option.autoCropWidth = this.width
    this.option.autoCropHeight = this.height
    this.setSrcs(this.srcs)
  },
  methods:{
    cloneSrcs() {
      return JSON.parse(JSON.stringify(this.srcs))
    },
    emitVal(val) {
      this.$emit('input', val)
    },
    setSrcs(val) {
      // 设置serviceImgs 新值  会触发 serviceImgs watcher
      // 会v-model 触发 srcs更新的
      this.serviceImgs = val
    },
    upload(e){
      let vm = this
      var e=window.event || e;
      vm.fileEvent = e
      var file = e.target.files[0]
      if (!/\.(gif|jpg|jpeg|png|bmp|GIF|JPG|PNG)$/.test(e.target.value)) {
        this.$message({
          message: '图片类型必须是.gif,jpeg,jpg,png,bmp中的一种',
          type: 'warning'
        });
        vm.fileEvent.target.value = ''
				 //alert('图片类型必须是.gif,jpeg,jpg,png,bmp中的一种')
				return false
      }
      var reader = new FileReader()
      reader.readAsDataURL(file)
      vm.compressImg(reader)
    },
    realTime(data){// 实时预览函数
      this.option.previews = data
    },
    compressImg(reader){ //压缩图片
      var vm = this
      reader.onload = function(){
        var img = new Image,
            width = 640, //image resize
            quality = 0.8, //image quality
            canvas = vm.$refs.canvas,
            drawer = canvas.getContext("2d");
        img.src = this.result;
        img.onload = function() {
            let src
            canvas.width = width;
            canvas.height = width * (img.height / img.width);
            drawer.drawImage(img, 0, 0, canvas.width, canvas.height);
            src = canvas.toDataURL("image/jpeg", quality);
            var _blob = vm.dataURItoBlob(src)
            if(vm.iscut){ //裁剪
              vm.dialogVisible = true
              vm.option.img = window.URL.createObjectURL(_blob)
            }else{ // 不裁剪
              vm.postImg(_blob)
            }
        }
      }
    },
    postImg(fileVal){
      // 查询时返回 两个字段
      // 编辑提交过去 相对路径
      let vm = this
      //判断已上传的长度
      if(vm.serviceImgs.length == vm.max){
        return
      }
      console.log(typeof fileVal,fileVal.size/1024/1024)
      if(fileVal.size>vm.maxSize){
        vm.$message({
          message: `请上传${vm.maxSizes}M以内的图片`,
          type: 'warning'
        });
        return
      }
      let formData = new FormData()
      var fileName = typeof fileVal=='object' ? new Date().valueOf()+'.png': fileVal.name
      formData.append('files', fileVal, fileName)
      formData.append('fileType', 1, fileName)
      vm.ableServe = false
      console.log('----开始上传----')
      vm.$store.dispatch('uploadImgs',  formData).then(res=> {
        console.log('---上传成功---')
        this.clearFile()
        console.log(this.srcs)
        this.serviceImg = res.uploadFileServerPath

        // if (this.receiveType === 'string') {
        //   vm.serviceImgs.push(res.uploadFileServerPath)
        // } else {
        //   this.serviceImgs.push({
        //     uploadFileServerPath: res.uploadFileServerPath,
        //     uploadFilePath: res.uploadFilePath
        //   })
        // }
        this.serviceImgs.push({
          uploadFileServerPath: res.uploadFileServerPath,
          uploadFilePath: res.uploadFilePath
        })
        this.ableServe = true
        // console.log(vm.serviceImgs)
        // vm.$emit('input',vm.serviceImgs.join(','))
      }).catch(() => {
        vm.clearFile()
        vm.ableServe = true
      })
  
    },
    dataURItoBlob(dataURI){
      var byteString = atob(dataURI.split(',')[1]);
      var mimeString = dataURI.split(',')[0].split(':')[1].split(';')[0];
      var ab = new ArrayBuffer(byteString.length);
      var ia = new Uint8Array(ab);
      for (var i = 0; i < byteString.length; i++) {
          ia[i] = byteString.charCodeAt(i);
      }
      return new Blob([ab], {type: mimeString});
    },
    sureCut(){
      if(!this.ableServe){
        return
      }
      var vm = this
      this.$refs.cropper.getCropData(dataURI=>{
        vm.postImg( vm.dataURItoBlob(dataURI) )
      })
    },
    delImg(i){
      console.log(i)
      this.serviceImgs.splice(i,1)
      // 删除原数组
    },
    clearFile(){
      this.fileEvent.target.value = ''
      this.dialogVisible = false

    },
    setMain(i) {
      // 颠倒排序
      const src = this.serviceImgs[i]
      this.serviceImgs.splice(i, 1)
      this.serviceImgs.unshift(src)
    }
  },
}
</script>
<style lang="scss" scoped>
$width:120px;
.upload-componet{
  .compress-canvas{
    display: none;
  }
  .img-wrap{
     //width: 850px;
    // height: 150px;
    .set-main-btn {
      position: absolute;
      bottom: -40px;
      text-align: center;
      width: 100%;
    }
    .fl{
      width: $width;
      height: $width;
      border:1px solid #e9eaed;
      position: relative;
      margin-right: 10px;
      margin-bottom: 40px;
      i.del-btn{
        position: absolute;
        right: -10px;
        top:-10px;
        font-size: 20px;
        cursor: pointer;
      }
      img{
        width:100%;
        height: 100%;
      }
    }

    .file-wrap{
      color: #999;
      border:1px solid ;
      width: $width;
      height: $width;
      //position: relative;
      text-align: center;
      input{
        cursor: pointer;
        position: absolute;
        top:0;
        left:0;
        width:100%;
        height: 100%;
        opacity: 0;
        z-index: 2;
      }
      i.upload-icon{
        color: #999;
        font-size: 100px;
        margin-top: 10px;
      }
    }
    
  }
  .cut-img-wrap{
    width:400px;
    height: 400px;
    margin-bottom: 42px;
    border:1px solid #333;
    position: relative;
    div.opt-btns{
      position: absolute;
      top: 100%;
      text-align: center;
      width:100%;
      padding: 10px 0;
      i{
        font-size: 36px; 
        cursor: pointer; 
        &.el-icon-error{
          color:#fa5555;
          margin-right: 40px;
        }
        &.el-icon-success{
          margin-left: 40px;
          color: #ff962f;
        }
      }
    }
  }
  
}
</style>