<template>  
  <div>
    <el-upload v-if="autoUpload" ref="upload" :action="uploadUrl + action" :headers="headers" :before-upload="beforeUpload" :on-remove="handleRemove" :data="extraData" :on-success="success" :on-error="handleError" :show-file-list="showFile" :accept="accept" :on-exceed="uploadExceed" :limit="1" :auto-upload="autoUpload">
      <slot></slot>
    </el-upload>
    <el-upload v-else ref="upload" :action="action" :headers="headers" :before-upload="beforeUploadfile" :on-remove="handleRemove" :on-change="handleChange" :http-request="upload" :data="extraData" :on-success="success" :show-file-list="showFile" :accept="accept" :on-exceed="uploadExceed" :limit="1" :auto-upload="false">
      <slot></slot>
    </el-upload>
  </div>
</template>
<script>
import BenzAMRRecorder from 'benz-amr-recorder'
import { getToken } from '@/base/cookie'
import axios from 'axios'
const uploadUrl = process.env.VUE_APP_WECHAT
export default {
  data() {
    return {
      amr: new BenzAMRRecorder(),
      uploadUrl,
      fileList: [],
      loading: null,
      headers: {
        'X-Token': getToken()
      }
    }
  },
  props: {
    allowSize: {
      type: Number, // 允许文件大小kb
      default: 10
    },
    action: {
      default: '/wechat/material/addImage',
      type: String
    },
    accept: {
      default: '',
      type: String
    },
    checkType: {
      default: () => ([]),
      type: Array
    },
    autoUpload: {
      default: false,
      type: Boolean
    },
    showFile: {
      default: false,
      type: Boolean
    },
    btnText: {
      default: '上传图片',
      type: String
    },
    title: {
      default: '',
      type: String
    },
    groupId: {
      default: '',
      type: String
    },
    type: {
      default: 'img',
      type: String
    }
  },
  computed: {
    extraData() {
      return {
        title: this.title,
        groupId: this.groupId
      }
    }
  },
  methods: {
    beforeUploadfile(file) {
      if (!file) {
        this.$message.warning('请先上传文件!')
        return false
      }
      const isLtM = file.size / 1024 / 1024 < this.allowSize
      if (!isLtM) {
        this.$message.warning(`${this.type === 'audio' ? '音频' : '视频'}大小不能超过 ${this.allowSize}MB!`)
        return false
      }
      return true
    },
    beforeUpload(file) {
      if (!file) {
        this.$message.warning('请先上传文件!')
        return false
      }
      const isLtM = file.size / 1024 / 1024 < this.allowSize
      if (!isLtM) {
        this.$message.warning(`文件大小不能超过 ${this.allowSize}MB!`)
        return false
      }
      this.handleImgChange()
      return true
    },
    handleChange(file, fileList) {
      this.fileList = fileList
      if (!this.checkType) return
      const fileExt = (file.name.replace(/.+\./, '') || '').toLocaleLowerCase()
      if (!this.checkType.includes(fileExt)) {
        this.$message.warning('请选择正确的文件类型!')
        this.$refs.upload.clearFiles()
        return false
      }
    },
    handleImgChange() {
      this.loading = this.$loading({
        lock: true,
        text: '素材上传中',
        spinner: 'el-icon-loading',
        background: 'hsla(0,0%,100%,.9)'
      })
      return true
    },
    async upload(param) {
      const formData = new FormData()
      formData.append('file', param.file)
      formData.append('title', this.title)
      console.log(param.file, 'param.file')
      if (this.type === 'audio' && !['video/x-ms-wma', 'audio/x-ms-wma'].includes(param.file.type)) {
        try {
          await this.amr.initWithBlob(param.file)
          const replyLength = this.amr.getDuration()
          if (replyLength > 60) {
            this.$message.warning('音频时长不能超过60秒!')
            this.amr = new BenzAMRRecorder()
            this.$refs.upload.clearFiles()
            return false
          }
          formData.append('replyLength', replyLength)
        } catch (err) {
          this.$message.error('语音解析失败!')
          this.amr = new BenzAMRRecorder()
          return false
        }
      }
      setTimeout(() => {
        this.loading = this.$loading({
          lock: true,
          text: '素材上传中',
          spinner: 'el-icon-loading',
          background: 'hsla(0,0%,100%,.9)'
        })
      }, 0)
      axios({
        url: `${uploadUrl}${this.action}`,
        method: 'post',
        data: formData,
        headers: { 'X-Token': getToken() },
        processData: false, // 告诉axios不要去处理发送的数据(重要参数)
        contentType: false
      }).then(res => {
        if (res && String(res?.data?.code) === '200') {
          this.$message.success('文件上传成功!')
          this.$emit('uploadSuccess')
        } else {
          this.$message.error(res?.data?.msg || '文件上传失败!')
        }
      }).catch(message => {
        this.$message.error(message?.data?.msg || '文件上传失败!')
      }).finally(_ => {
        this.loading.close()
      })
    },
    submit() {
      if (!this.fileList.length) {
        this.$message.warning('请先上传文件')
        return
      }
      this.$refs.upload.submit()
    },
    clearFiles() {
      this.$refs.upload.clearFiles()
    },
    async success(res, file) {
      if (~~res?.code !== 200 || res === false) {
        await this.$nextTick()
        this.loading.close()
        this.$refs.upload.clearFiles()
        return
      }
      if (this.type === 'img') {
        this.$message.success('上传成功!')
        this.$emit('uploadSuccess')
      }
      this.$refs.upload.clearFiles()
      await this.$nextTick()
      this.loading && this.loading.close()
    },
    async handleError() {
      this.$refs.upload.clearFiles()
    },
    uploadExceed(files, fileList) {
      this.$set(fileList[0], 'raw', files[0])
      this.$set(fileList[0], 'name', files[0].name)
      this.$refs.upload.clearFiles()
      this.$refs.upload.handleStart(files[0])
    },
    handleRemove(file, fileList) {
      this.$refs.upload.clearFiles()
    }
  }
}
</script>
<style lang="scss" scoped>
</style>
