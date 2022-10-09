<template>
  <div class="upload-wrap">
    <el-upload
      :action="upload_qiniu_area"
      :auto-upload="true"
      :limit="1"
      accept="image/jpg, image/png, image/jpeg"
      :file-list="fileList"
      list-type="picture-card"
      :on-preview="picCardPreview"
      :before-upload="beforePicUpload"
      :on-exceed="handleExceed"
      :on-remove="removePic"
      :http-request="uploadQiniu"
      :multiple="false"
    >
      <el-button size="small" type="primary">点击上传</el-button>
      <div slot="tip" class="el-upload__tip">只能上传jpg/png文件，且不超过10MB</div>
    </el-upload>

    <el-dialog :visible.sync="dialogVisible">
      <img width="100%" :src="dialogImageUrl" />
    </el-dialog>
  </div>
</template>

<script>
import { getUploadToken } from './upload'
import axios from 'axios'

export default {
  name: 'UploadPicture',
  data () {
    return {
      upload_qiniu_address: 'http://rjgwa5zhf.bkt.gdipper.com/',
      dialogImageUrl: '',
      dialogVisible: false,
      fileList: [],
      upload_qiniu_area: 'https://up-na0.qiniup.com', // 七牛云上传储存区域的上传域名
      upToken: ''
    }
  },
  created () {
    getUploadToken().then(result => {
      // 获取token
      this.upToken = result.data.upToken
      console.log(this.upToken)
      console.log('----')
    })
  },
  methods: {
    emitInput (val) {
      this.$emit('input', val)
    },
    picCardPreview (file) {
      // 上传图预览
      this.dialogImageUrl = file.url
      this.dialogVisible = true
    },
    beforePicUpload (file) {
      // 图片校验
      const limitPic = file.type === 'image/png' || file.type === 'image/jpg' || file.type === 'image/jpeg'
      if (!limitPic) {
        this.$notify.warning({
          title: '警告',
          message: '请上传格式为image/png,image/jpg,image/jpeg的图片'
        })
      }
      const limitSize = file.size / 1024 / 1024 / 2 < 2
      if (!limitSize) {
        this.$notify.warning({
          title: '警告',
          message: '图片大小必须小于2M'
        })
      }
      return limitPic && limitSize
    },
    removePic (file, fileList) {
      // 移除图片
      this.fileList = fileList
    },
    handleExceed () {
      // 文件超出个数限制
      this.$notify.warning({
        title: '警告',
        message: '一次只能上传三张图片',
        duration: 2000
      })
    },
    uploadQiniu (request) {
      // 上传七牛
      this.handleUpload(request)
        .then(result => {
          if (!result.data.key) {
            this.$message.error({ message: '图片上传失败,请重新上传', duration: 2000 })
          } else {
            this.fileList.push({ url: this.upload_qiniu_address + result.data.key })
            this.$emit('uploadSuccess', this.fileList)
          }
        })
        .catch(err => {
          this.$message.error({ message: `图片上传失败${err}`, duration: 2000 })
        })
    },
    handleUpload (request) {
      const promise = new Promise((resolve, reject) => {
        const config = {
          headers: { 'Content-Type': 'multipart/form-data' }
        }
        let fileType = ''
        if (request.file.type === 'image/jpg') {
          fileType = 'jpg'
        } else if (request.file.type === 'image/png') {
          fileType = 'png'
        } else if (request.file.type === 'image/jpeg') {
          fileType = 'jpeg'
        }
        const key = `front_${new Date().getTime()}${Math.floor(Math.random() * 100)}.${fileType}` // 自定义图片名

        const fd = new FormData()
        fd.append('file', request.file)
        fd.append('token', this.upToken)
        fd.append('key', key)
        console.log(this.upToken)
        axios
          .post(this.upload_qiniu_area, fd, config)
          .then(res => {
            if (res.status === 200 && res.data) {
              resolve(res)
            } else {
              reject(res)
            }
          })
          .then(() => {
            this.emitInput(this.fileList[0].url)
          })
          .catch(err => {
            this.$message.error({ message: `上传失败[${err.status}]`, duration: 2000 })
          })
      })

      return promise
    }
  }
}
</script>

<style scoped>
.upload-wrap {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 50px;
}
</style>