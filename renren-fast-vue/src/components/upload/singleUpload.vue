<template>
  <div>
    <el-upload
      :action="upload_qiniu_area"
      :data="dataObj"
      list-type="picture"
      :multiple="false" :show-file-list="showFileList"
      :file-list="fileList"
      :before-upload="beforeUpload"
      :on-remove="handleRemove"
      :on-success="handleUploadSuccess"
      :on-preview="handlePreview">
      <el-button size="small" type="primary">点击上传</el-button>
      <div slot="tip" class="el-upload__tip">只能上传jpg/png文件，且不超过10MB</div>
    </el-upload>
    <el-dialog :visible.sync="dialogVisible">
      <img width="100%" :src="fileList[0].url" alt="">
    </el-dialog>
  </div>
</template>
<script>
   import {policy} from './policy'

export default {
     name: 'singleUpload',
     props: {
       value: String
     },
     computed: {
       imageUrl () {
         return this.value
       },
       imageName () {
         if (this.value != null && this.value !== '') {
           return this.value.substr(this.value.lastIndexOf('/') + 1)
         } else {
           return null
         }
       },
       fileList () {
         return [{
           name: this.imageName,
           url: this.imageUrl
         }]
       },
       showFileList: {
         get: function () {
           return this.value !== null && this.value !== '' && this.value !== undefined
         },
         set: function (newValue) {
         }
       }
     },
     data () {
       return {
         dataObj: {
           token: '',
           key: ''
          // callback:'',
         },
         dialogVisible: false,
         upload_qiniu_address: 'http://rjgwa5zhf.bkt.gdipper.com/',
         upload_qiniu_area: 'https://up-na0.qiniup.com', // 七牛云上传储存区域的上传域名
         upToken: ''
       }
     },
     methods: {
       emitInput (val) {
         this.$emit('input', val)
       },
       handleRemove (file, fileList) {
         this.emitInput('')
       },
       handlePreview (file) {
         this.dialogVisible = true
       },
       beforeUpload (file) {
         let _self = this
         return new Promise((resolve, reject) => {
           policy().then(response => {
             console.log('响应的数据', response)
             let fileType = ''
             if (file.type === 'image/jpg') {
               fileType = 'jpg'
             } else if (file.type === 'image/png') {
               fileType = 'png'
             } else if (file.type === 'image/jpeg') {
               fileType = 'jpeg'
             }
             _self.dataObj.token = response.data.upToken
             _self.dataObj.key = `front_${new Date().getTime()}${Math.floor(Math.random() * 100)}.${fileType}`
             console.log('响应的数据222。。。', _self.dataObj)
             resolve(true)
           // eslint-disable-next-line handle-callback-err
           }).catch(err => {
             // eslint-disable-next-line prefer-promise-reject-errors
             reject(false)
           })
         })
       },
       handleUploadSuccess (res, file) {
         console.log('上传成功...')
         console.log(res)
         this.showFileList = true
         this.fileList.pop()
         this.fileList.push({ url: this.upload_qiniu_address + res.key })
         this.emitInput(this.fileList[0].url)
       }
     }
}
</script>
<style>

</style>


