<template>
  <div>
    <el-upload
      :action="upload_qiniu_area"
      :data="dataObj"
      :list-type="listType"
      :file-list="fileList"
      :before-upload="beforeUpload"
      :on-remove="handleRemove"
      :on-success="handleUploadSuccess"
      :on-preview="handlePreview"
      :limit="maxCount"
      :on-exceed="handleExceed"
      :show-file-list="showFile"
    >
      <i class="el-icon-plus"></i>
    </el-upload>
    <el-dialog :visible.sync="dialogVisible">
      <img width="100%" :src="dialogImageUrl" alt />
    </el-dialog>
  </div>
</template>
<script>
import { policy } from './policy'
export default {
  name: 'multiUpload',
  props: {
    // 图片属性数组
    value: Array,
    // 最大上传图片数量
    maxCount: {
      type: Number,
      default: 30
    },
    listType: {
      type: String,
      default: 'picture-card'
    },
    showFile: {
      type: Boolean,
      default: true
    }

  },
  data () {
    return {
      dataObj: {
        token: '',
        key: ''
      },
      dialogVisible: false,
      dialogImageUrl: null,
      upload_qiniu_address: 'http://rjgwa5zhf.bkt.gdipper.com/',
      upload_qiniu_area: 'https://up-na0.qiniup.com'
    }
  },
  computed: {
    fileList () {
      let fileList = []
      for (let i = 0; i < this.value.length; i++) {
        fileList.push({ url: this.value[i] })
      }

      return fileList
    }
  },
  mounted () {},
  methods: {
    emitInput (fileList) {
      let value = []
      for (let i = 0; i < fileList.length; i++) {
        value.push(fileList[i].url)
      }
      this.$emit('input', value)
    },
    handleRemove (file, fileList) {
      this.emitInput(fileList)
    },
    handlePreview (file) {
      this.dialogVisible = true
      this.dialogImageUrl = file.url
    },
    beforeUpload (file) {
      let _self = this
      return new Promise((resolve, reject) => {
        policy()
          .then(response => {
            // eslint-disable-next-line no-template-curly-in-string
            console.log('这是什么${filename}')
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
            resolve(true)
          })
          .catch(err => {
            console.log('出错了...', err)
            // eslint-disable-next-line prefer-promise-reject-errors
            reject(false)
          })
      })
    },
    handleUploadSuccess (res, file) {
      this.fileList.push({
        name: file.name,
        // url: this.dataObj.host + "/" + this.dataObj.dir + "/" + file.name； 替换${filename}为真正的文件名
        // eslint-disable-next-line no-template-curly-in-string
        url: this.upload_qiniu_address + res.key
      })
      this.emitInput(this.fileList)
    },
    handleExceed (files, fileList) {
      this.$message({
        message: '最多只能上传' + this.maxCount + '张图片',
        type: 'warning',
        duration: 1000
      })
    }
  }
}
</script>
<style>
</style>


