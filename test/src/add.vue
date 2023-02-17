<template>  
  <div class="massMessageManage_add auto_h_block">
    <div class="add-title">新建群发</div>
    <tip :textList="textList" :has-title="true" style="width: 712px"></tip>
    <div class="flex">
      <div class="sendNumber" @click="previewSendNumber"><span> 预计发送人数</span><span style="font-size: 20px">{{ sendNumber }}</span></div>
      <el-form class="form mt-16" :model="addForm" ref="addForm" label-width="82px" style="width: 50%">
        <el-form-item label="群发方式" style="width: 500px">
          <el-radio-group v-model="rangType" @change="changeRangType">
            <el-radio label="1">全部粉丝</el-radio>
            <el-radio label="2">筛选标签</el-radio>
          </el-radio-group>
          <div v-if="~~rangType === 2"><a class="tag-btn mr-5" href="javascript:;" style="white-space: nowrap;" v-for="(itemLabel, index) in addForm.labelList"><span class="mr-5">{{ itemLabel.labelName }}</span><i class="el-icon-error" @click="handleDelLabel(index)"></i></a><a class="add-label tag-btn" href="javascript:;" style="white-space: nowrap;" @click="handleAddLabel()"><i class="el-icon-circle-plus-outline mr-5"></i><span>添加标签</span></a></div>
        </el-form-item>
        <el-form-item label="性别">
          <el-checkbox-group v-model="addForm.sex">
            <el-checkbox :label="0">未知</el-checkbox>
            <el-checkbox :label="1">男</el-checkbox>
            <el-checkbox :label="2">女</el-checkbox>
          </el-checkbox-group>
        </el-form-item>
        <el-form-item label="地区:" @keyup.delete.native="watchDel">
          <el-cascader v-model="addForm.city" clearable size="large" placeholder="可不选,只能选择中国地区" style="width: 350px" ref="cascader" popper-class="mass_message_manage_cascader_pop" :props="{ value: 'name',label: 'name',children: 'childList',checkStrictly: true, expandTrigger: 'hover' }" :options="areaData" @change="handleChange"></el-cascader>
        </el-form-item>
        <el-form-item label="关注时间">
          <el-date-picker size="large" v-model="time" type="daterange" align="right" unlink-panels range-separator="至" start-placeholder="开始日期" end-placeholder="结束日期" value-format="yyyy-MM-dd" :picker-options="pickerOptions"></el-date-picker>
        </el-form-item>
        <el-form-item class="is-required message-content" label="发送消息">
          <materialTabs @getContent="getContent" :publishStatus="7" :isSingle="false" :hiddenInsertLink="false"></materialTabs>
        </el-form-item>
        <el-form-item class="is-required" label="发送时间">
          <el-select v-model="addForm.sendType" style="width: 136px" size="large">
            <el-option :value="1" label="立即发送"></el-option>
            <el-option :value="2" label="定时发送"></el-option>
          </el-select>
          <el-select class="ml-10" v-if="~~addForm.sendType === 2" v-model="sendTimeType" style="width: 136px" size="large" @change="sendTime=''">
            <el-option :value="1" label="今天"></el-option>
            <el-option :value="2" label="明天"></el-option>
          </el-select>
          <el-time-picker class="ml-10" v-if="~~addForm.sendType === 2" format="HH:mm" style="width: 136px" size="large" placeholder="发送时间" value-format="HH:mm" v-model="sendTime" :picker-options="timeOptions" @change="timeChange"></el-time-picker>
        </el-form-item>
        <el-form-item>
          <el-button class="fl" type="primary" size="large" :loading="isloading" @click.stop="handleSave">发送</el-button>
        </el-form-item>
      </el-form>
    </div>
    <addTag :visibleDialog="labelVisible" @submit="submitTag" :chooseList="JSON.parse(JSON.stringify(addForm.labelList))" @close="labelVisible = false" :limit="5"></addTag>
    <el-dialog width="800px" :title="'预发送人数（共' + wechatUserInfoResult.total + '人）'" :visible.sync="visible">
      <el-table class="mt-10" :data="wechatUserInfoResult.list" :height="400" :header-cell-style="{background: 'rgb(242, 244, 247)'}">
        <el-table-column label="头像">
          <template slot-scope="{ row }">
            <div class="flex_center"><img :src="row.headImgUrl" alt=""></div>
          </template>
        </el-table-column>
        <el-table-column label="昵称">
          <template slot-scope="{ row }">
            <div><span>{{ row.nickName }}</span></div>
          </template>
        </el-table-column>
        <el-table-column label="地区">
          <template slot-scope="{ row }"><span v-if="row.country || row.province || row.city"><span>{{ row.country }}</span>
  <span>{{ row.province }}</span>
  <span>{{ row.city }}</span></span><span v-else>--</span></template>
        </el-table-column>
        <el-table-column label="性别">
          <template slot-scope="{ row }"><span> {{ row.sex | filSex }}</span></template>
        </el-table-column>
        <el-table-column width="180" prop="firstFocusTime" label="首次关注时间"></el-table-column>
      </el-table>
      <pagination :total="~~wechatUserInfoResult.total" :pageNum="page.pageNum" :pageSize="page.pageSize" @sizeChange="handleSizeChange" @currentChange="handleCurrentChange"></pagination>
    </el-dialog>
  </div>
</template>
<script>
import moment from 'moment'
import addTag from '@/components/addTagDia'
import tip from '@/views/wxMarketing/components/Tip.vue'
import materialTabs from '@/components/materialTab/index.vue'
import API from '@/views/wxMarketing/api'
import { authMixin } from '../auth/authMixin'
import { pickerOptions, sexMap } from '@/views/wxMarketing/config.js'
const { saveOfficialAccountMessage, getSendNumber, getWeChatAreas } = API
export default {
  components: { materialTabs, addTag, tip },
  mixins: [authMixin],
  data() {
    return {
      text: '1.只有认证服务号才可以群发消息， 此次群发可选择发送给全部用户或某个标签\r\n2.无论在公众号后台，还是使用珍客scrm群发，用户每月只能接收4条群发消息，多于4条的群发将对该用户发送失败;',
      textList: [
        '温馨提示', '1.只有认证服务号才可以群发消息， 此次群发可选择发送给全部用户或某个标签;', '2.无论在公众号后台，还是使用珍客scrm群发，用户每月只能接收4条群发消息，多于4条的群发将对该用户发送失败;'
      ],
      pickerOptions,
      timeOptions: {
        // selectableRange: '00:00:00 - 23:59:59'
      },
      labelVisible: false,
      isloading: false,
      addForm: {
        type: '',
        sendType: 1,
        labelList: [],
        materialParam: {},
        city: [],
        country: '中国',
        attentionStartTime: '',
        attentionEndTime: '',
        sex: [],
        province: '' // 省份
      },
      visible: false,
      total: 0,
      page: {
        pageNum: 1,
        pageSize: 10
      },
      wechatUserInfoResult: {},
      sendTime: '',
      rangType: '1',
      sendTimeType: 1,
      tagType: 0,
      sendNumber: 0,
      options: [],
      time: [],
      areaData: []
    }
  },
  created() {
    this.queryArea()
  },
  filters: {
    filSex(val) {
      return val instanceof Array ? (val || []).map((v) => sexMap[v]).join(', ') : sexMap[val] ? sexMap[val] : '所有'
    }
  },
  methods: {
    changeRangType(val) {
      const [attentionStartTime = '', attentionEndTime = ''] = this.time || []
      const [country, province = '', city = ''] = this.addForm?.city?.includes('中国') || []
      const params = {
        ...this.copyParams,
        ...this.addForm,
        country,
        province: province,
        city,
        attentionStartTime,
        attentionEndTime
      }
      this.copyParams = params
      this.getSendNumber(this.copyParams)
    },
    handleFocus() {
      if (~~this.sendTimeType === 1) {
        this.timeOptions.selectableRange = `${moment(new Date())
          .add('year', 0)
          .format('YYYY-MM-DD HH:mm:ss')
          .slice(11, 20)} - 23:59:59`
      } else {
        this.timeOptions.selectableRange = '00:00:00 - 23:59:59'
      }
    },
    // 选择时间
    timeChange(val) {
      if (this.sendTimeType != 1) return
      const now = new Date()
      const ymd = this.$moment(now).format('YYYY-MM-DD')
      const ymdhm = this.$moment(now).format('YYYY-MM-DD HH:mm')
      // 同转化为 年月日时分 进行比较
      if (new Date(ymdhm) > new Date(ymd + ' ' + val)) {
        this.$message.warning('选择的时间不能小于当前时间，请重新选择')
        this.sendTime = ''
      }
    },
    getContent(val) {
      const { type, id, content } = val || {}
      if (type === undefined) return (this.addForm.materialParam = {})
      if (~~type === 4) {
        this.addForm.materialParam = { text: content }
      } else {
        this.addForm.materialParam = { materialId: id }
      }
      this.addForm.type = ~~type
    },
    watchDel(event) {
      event.preventDefault()
    },
    previewSendNumber() {
      this.visible = true
    },
    async queryArea() {
      const [err, data] = await getWeChatAreas()
      if (err !== null || ~~data.code !== 200) return
      this.areaData = data?.data?.[0]?.childList || []
    },
    handleChangeTime(val) {
      const [attentionStartTime, attentionEndTime] = val || []
      this.addForm.attentionStartTime = attentionStartTime || ''
      this.addForm.attentionEndTime = attentionEndTime || ''
      this.time = val || []
    },
    handleChange() {
      this.$refs.cascader.toggleDropDownVisible()
    },
    // 发送
    async handleSave() {
      const { materialId, text } = this.addForm.materialParam
      if (!materialId && !text) {
        return this.$message.error('请选择发送的内容')
      }
      if (this.addForm.sendType === 2 && !this.sendTime) {
        return this.$message.error('请完善发送时间')
      }
      this.$refs.addForm.validate(async(valid) => {
        if (!valid) return
        const { sendNumber } = this
        const [attentionStartTime = '', attentionEndTime = ''] = this.time || []
        const [country = '', province = '', city = ''] = ['中国', ...this.addForm.city] || []
        const params = {
          ...this.addForm,
          sendNumber,
          province,
          city,
          country,
          attentionStartTime,
          attentionEndTime,
          labelList: Array.from(this.addForm.labelList, ({ labelId }) => ({ labelId }))
        }
        if (~~params.sendType === 2) {
          const date =
            ~~this.sendTimeType === 1
              ? moment(new Date())
                  .add('year', 0)
                  .format('YYYY-MM-DD')
              : moment()
                  .add(1, 'd')
                  .format('YYYY-MM-DD')
          params.sendTime = date + ' ' + this.sendTime
        } else {
          // 不合理, 但是为了不极化前后端矛盾, ....
          params.sendTime = moment(new Date())
            .add('year', 0)
            .format('YYYY-MM-DD HH:mm')
        }
        this.isloading = true
        const [err, data] = await saveOfficialAccountMessage(params)
        this.isloading = false
        if (err !== null || ~~data.code !== 200) return
        this.$message.success(`${params.id ? '编辑' : '新建'}群发消息成功`)
        this.$router.back()
      })
    },
    submitTag(list) {
      console.log(this.addForm.labelList, 'this.addForm.labelList ')
      this.addForm.labelList = list
    },
    handleDelLabel(index) {
      this.addForm.labelList.splice(index, 1)
    },
    handleAddLabel() {
      this.page.pageNum = 1
      this.labelVisible = true
    },
    async getSendNumber(params) {
      const [err, data] = await getSendNumber(params)
      if (err !== null || ~~data?.code !== 200) return
      this.wechatUserInfoResult = data?.data?.wechatUserInfoResult
      this.sendNumber = this.wechatUserInfoResult.total
    },
    handleSizeChange(size) {
      this.page.pageNum = 1
      this.page.pageSize = size
      this.getSendNumber({ ...this.copyParams, ...this.page })
    },
    handleCurrentChange(page) {
      this.page.pageNum = page
      this.getSendNumber({ ...this.copyParams, ...this.page })
    }
  },
  watch: {
    time: {
      async handler(val) {
        const [attentionStartTime = '', attentionEndTime = ''] = val || []
        const [country, province = '', city = ''] = this.addForm.city.includes('中国')
          ? this.addForm.city
          : ['中国', ...this.addForm.city]
        const params = {
          ...this.copyParams,
          ...this.addForm,
          country,
          province: province,
          city,
          attentionStartTime,
          attentionEndTime
        }
        this.copyParams = params
        this.getSendNumber(params)
      },
      deep: true
    },
    addForm: {
      async handler(val) {
        const [attentionStartTime = '', attentionEndTime = ''] = this.time || []
        const [country, province = '', city = ''] = ['中国', ...val.city] || []
        const params = {
          ...this.copyParams,
          ...val,
          country,
          province: province,
          city,
          labelList: ~~this.rangType === 1 ? [] : val.labelList,
          attentionStartTime,
          attentionEndTime
        }
        this.copyParams = params
        this.getSendNumber(params)
      },
      deep: true,
      immediate: true
    }
  }
}
</script>
<style lang="scss">
.mass_message_manage_cascader_pop {
  .el-cascader-panel .el-radio {
    width: 100%;
    height: 100%;
    z-index: 10;
    position: absolute;
    top: 10px;
    right: 10px;
    .el-radio__input {
      visibility: hidden;
    }
    .el-cascader-node__postfix {
      top: 10px;
    }
  }
}
.massMessageManage_add {
  /deep/ .el-input__inner {
    padding: 0 50px 0 15px;
  }
  /deep/ .el-select__tags-text {
    margin-right: 10px;
  }
  /deep/ .el-input__count-inner {
    line-height: 100% !important;
  }
  .message-content {
    /deep/ .el-form-item__content {
      line-height: 100%;
    }
  }
  padding: 20px;
  background: #fff;
  .flex_center {
    display: flex;
    align-items: center;
    height: 100%;
    img {
      width: 32px;
      height: 32px;
      margin-right: 8px;
    }
  }
  .add-title {
    font: {
      weight: 500;
      size: 16px;
    }
    color: #303133;
    line-height: 24px;
    position: relative;
    margin-left: 8px;
    margin: 10px 0;
  }
  .el-cascader {
    width: 400px;
  }
  p {
    color: #aaa;
  }
  .flex {
    display: flex;
    position: relative;
    .sendNumber {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      position: absolute;
      cursor: pointer;
      top: 50px;
      width: 150px;
      height: 100px;
      color: #aaa;
      line-height: 40px;
      text-align: center;
      border-radius: 6px;
      left: calc(712px - 150px);
      font-size: 14px;
      box-shadow: 0px 2px 20px 0px rgba(0, 21, 41, 0.15);
      z-index: 999;
    }
    .form {
      flex: 2;
      .tag-btn {
        width: 129px;
        height: 22px;
        background: #c5e3ff;
        border-radius: 2px;
        border: 0px;
        color: #1890ff;
        padding: 4px 10px;
        text-decoration: none;
        margin-right: 4px;
      }
      .add-label {
        background: #f7fcff;
        border-radius: 2px;
        border: 1px solid #b0e1ff;
      }
    }
  }
}
/deep/ .el-alert{
  .el-alert__closebtn{
    display: none!important;
  }
  .el-alert__description{
    width: 700px!important;
  }

}
</style>
