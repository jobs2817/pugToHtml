<template>  
  <div class="massMessageManage">
    <div class="w_block"><Tabs-box v-model="sendType" @tabClick="handleClick" :tabs="tabsList"></Tabs-box>
      <el-form class="w_block reset_form_padding" :model="form" :inline="true">
        <el-form-item>
          <el-input v-model="name" clearable style="width: 150px" placeholder="请输入搜索关键字" size="large"></el-input>
        </el-form-item>
        <el-form-item>
          <el-date-picker size="large" v-model="time" type="daterange" align="right" unlink-panels range-separator="至" start-placeholder="发送开始日期" end-placeholder="发送结束日期" value-format="yyyy-MM-dd" style="width: 300px" :picker-options="pickerOptions"></el-date-picker>
        </el-form-item>
        <el-form-item>
          <el-select size="large" placeholder="请选择消息类型" style="width: 80px" v-model="type">
            <el-option v-for="(item, index) in msgTypeList" :label="item.label" :value="item.value" :key="index"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item>
          <el-button size="large" type="primary" @click="search()">查询</el-button>
        </el-form-item>
        <el-form-item>
          <el-button size="large" type="plain" @click="handleReset()">重置</el-button>
        </el-form-item>
        <el-form-item class="fr" v-if="hasAuth('775','781')">
          <el-button class="mb-5" size="large" type="primary" icon="el-icon-plus" @click="handleAddMassMessage()">新建群发</el-button>
        </el-form-item>
      </el-form>
    </div>
    <div class="w_block" style="min-height: calc(100vh - 320px)">
      <el-table class="table" ref="table" :header-cell-style="{background: '#f4f7ff'}" :data="list">
        <el-table-column label="发送时间" width="180px">
          <template slot-scope="scope">
            <div class="flex_column"><span>{{ scope.row.sendTime || '--' }}</span><span style="color: rgba(80, 93, 105, 0.5);text-align: center">{{ scope.row.sendStatus | status }}</span></div>
          </template>
        </el-table-column>
        <el-table-column label="消息类型" prop="type" width="200px">
          <template slot-scope="{ row }">
            <div class="icon"><img :src="require('@/assets/wxMarketing/type-' + row.type + '.png')" alt="alt"><span>{{ row.type | filType }}</span></div>
          </template>
        </el-table-column>
        <el-table-column width="150px">
          <template slot-scope="{ row }">
            <template v-if="row.type === 1">
              <div class="flex_center"><i :style="{backgroundImage: `url(${getContent(row).compressImageUrl || getContent(row).fileUrl })`}"></i></div>
            </template>
            <template v-if="row.type === 2">
              <div class="flex_center"><i class="pause_audio" v-if="id === getContent(row).id" @click.stop.prevent="handleAudioPlay()"></i><i class="play_audio" v-else @click.stop.prevent="handleAudioPlay(getContent(row))"></i></div>
            </template>
            <template v-if="row.type === 3">
              <div class="flex_center"><i class="playerBox" :style="{backgroundImage: `url(${getContent(row).coverUrl})`}"><img class="center" @click.stop.prevent="handlePreviewVideo(getContent(row))" src="@/assets/wxMarketing/player.png" alt="alt"></i></div>
            </template>
            <template v-if="row.type === 0">
              <div class="flex_center"><i :style="{backgroundImage: `url(${getContent(row).url})`}"></i></div>
            </template>
          </template>
        </el-table-column>
        <el-table-column label="内容" width="250px">
          <template slot-scope="{ row }">
            <template v-if="row.type === 1">
              <div class="flex_center">
                <div style="flex: 1; width: 0;">
                  <p class="title ml-10" v-html="getContent(row).title"></p>
                </div>
              </div>
            </template>
            <template v-if="row.type === 2">
              <div class="flex_center">
                <div style="flex: 1; width: 0">
                  <p class="title ml-10" v-html="getContent(row).title"></p>
                </div>
              </div>
            </template>
            <template v-if="row.type === 3">
              <div class="flex_center">
                <div style="flex: 1; width: 0">
                  <p class="title ml-10" v-html="getContent(row).title"></p>
                </div>
              </div>
            </template>
            <template v-if="row.type === 4">
              <div class="flex_center">
                <div style="flex: 1; width: 0">
                  <p class="ml-10" v-html="getContent(row).title"></p>
                </div>
              </div>
            </template>
            <template v-if="row.type === 0">
              <div class="flex_center">
                <div style="flex: 1; width: 0">
                  <p class="title ml-10" v-html="getContent(row).title"></p>
                  <p class="ml-10" v-if="showContent" v-html="((getContent(row) || {}).content || '')"></p>
                </div>
              </div>
            </template>
          </template>
        </el-table-column>
        <el-table-column label="发送/成功">
          <template slot-scope="scope"><span>{{ scope.row.sendNumber || '--' }}</span><span>/</span><span>{{ scope.row.successNumber || 0 }}人</span></template>
        </el-table-column>
        <el-table-column v-if="permission(['776','782','777','783'])" label="操作" fixed="right" width="150px">
          <template slot-scope="{ row }">
            <el-button v-if="hasAuth('776','782')" type="text" size="large" iconPoi="left" @click="handleToDetail(row)">详情</el-button>
            <el-button v-if="hasAuth('777','783')" type="text" size="large" iconPoi="left" style="color:red;" @click="handleDel(row)">删除</el-button>
          </template>
        </el-table-column>
      </el-table>
      <pagination v-if="!!total" :total="~~total" :pageNum="page.pageNum" :pageSize="page.pageSize" @sizeChange="handleSizeChange" @currentChange="handleCurrentChange"></pagination>
    </div>
    <videoPlayer ref="videoPlayer"></videoPlayer>
  </div>
</template>
<script>
import API from '@/views/wxMarketing/api'
import { authMixin } from '../auth/authMixin'
import { msgTypeList } from '../autoResponse/config.js'
import { pickerOptions } from '@/views/wxMarketing/config.js'
import { fieldMap } from '@/components/materialTab/config.js'
import { audioMixin } from '@/components/materialTab/toolComponents/audioMixin.js'
import audioPlayer from '@/components/materialTab/toolComponents/audioPlayer.vue'
import videoPlayer from '@/components/materialTab/toolComponents/player.vue'
const { selectOfficialAccountMessageList, deleteOfficialAccountMessage } = API
export default {
  mixins: [{ ...authMixin, ...audioMixin }],
  components: { audioPlayer, videoPlayer },
  data() {
    return {
      tabsList: [
        { label: '立即群发', value: '1', code: '773' }, { label: '定时群发', value: '2', code: '779' }
      ],
      pickerOptions,
      msgTypeList,
      name: '',
      type: '',
      form: {
        name: '',
        type: ''
      },
      total: 0,
      showContent: false,
      page: {
        pageSize: 10,
        pageNum: 1
      },
      visible: false,
      time: null,
      list: [],
      row: {},
      sendType: 0
    }
  },
  computed: {
    getContent() {
      return function(row = {}) {
        const type = ~~row.type
        if (type === 4) return { title: row.title }
        const info = row?.materialDetailResult?.[fieldMap[type]] || {}
        switch (type) {
          case 1:
          case 2:
          case 3:
            return { ...info, title: row.title }
          case 0:
            return { url: row?.url?.image?.fileUrl ? row?.url?.image?.fileUrl : info?.articles?.[0].picUrl, title: row.title, content: row?.content }
        }
      }
    }
  },
  created() {
    const queryTab = this.$route.query.tabIndex
    if (queryTab && this.permission([this.tabsList[queryTab].code])) {
      this.sendType = +queryTab
    } else if (this.permission(['773'])) {
      this.sendType = 0
    } else if (this.permission(['779'])) {
      this.sendType = 1
    }

    this.search()
  },
  filters: {
    filType(val) {
      return msgTypeList.find(v => v.value === val)?.label
    },
    status(val) {
      const map = {
        1: '已群发',
        2: '待群发',
        3: '发送失败',
        4: '无法群发'
      }
      return map[val]
    }
  },
  methods: {
    hasAuth(code1, code2) {
      return (this.sendType == 0 && this.permission([code1])) || (this.sendType == 1 && this.permission([code2]))
    },
    handleClick(row) {
      this.$router.replace({
        query: {
          tabIndex: row.originValue
        }
      })
      const i = this.tabsList.findIndex(x => x.label === row.label)
      this.sendType = i
      this.name = ''
      this.type = ''
      this.form = this.$options.data().form
      this.time = null
      this.search()
    },
    handlePreviewVideo(row) {
      this.$refs.videoPlayer.change(row)
    },
    cptSex(val = []) {
      const map = { 0: '未知', 1: '男', 2: '女' }
      return (val || []).map(v => {
        return map[v]
      }).join(' / ') || '--'
    },
    handleReset() {
      this.name = ''
      this.type = ''
      this.page = this.$options.data().page
      this.form = this.$options.data().form
      this.time = null
      this.search()
    },
    search() {
      this.form.name = this.name
      this.form.type = this.type
      const [sendStartTime, sendEndTime] = this.time || []
      this.form = { ...this.form, sendStartTime, sendEndTime }
      this.query()
    },
    async query(page = 1) {
      this.page.pageNum = page
      const params = { ...this.form, ...this.page, sendType: this.tabsList[this.sendType].value }
      const [err, data] = await selectOfficialAccountMessageList(params)
      if (err !== null || ~~data?.code !== 200) return
      const { total, list = [] } = data?.data || {}
      this.total = ~~total
      this.list = list
      if (this.form.name.trim()) this.showContent = true
      else this.showContent = false
    },
    async handleAddMassMessage(row) {
      this.$router.push({ path: '/oc/wxMarketing/massMessageManage/add' })
    },
    handleToDetail(row) {
      this.$router.push({ path: '/oc/wxMarketing/massMessageManage/massMessageReport', query: { id: row.id } })
    },
    async handleDel(row) {
      const { id = '' } = row || {}
      const params = [id]
      this.$confirm('你确定要删除这条群发消息吗? 是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(async() => {
        const [err, data] = await deleteOfficialAccountMessage(params)
        if (err !== null || ~~data.code !== 200) return
        this.$message.success('删除成功')
        this.query()
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        })
      })
    },
    handleSizeChange(size) {
      this.page.pageSize = size
      this.query()
    },
    handleCurrentChange(page) {
      this.query(page)
    }
  }
}
</script>
<style lang="scss" scoped>
/deep/ .heightLight {
  font-size: 14px;
  font-family: PingFangSC-Regular, PingFang SC;
  font-weight: 400;
  color: #1890FF;
}
.massMessageManage {
  font-family: PingFangSC-Regular, PingFang SC;
  .el-form {
    padding: 0px;
  }
  .icon {
    display: flex;
    // justify-content: center;
    align-items: center;
    height: 100%;
    width: 100%;
    img {
      margin-right: 8px;
      width: 24px;
      height: 24px;
    }
  }
  i {
    width: 144px;
    height: 100%;
    display: inline-block;
    vertical-align: middle;
    background-size: cover;
    // background-size: contain;
    // background-position: 50% 50%;
    // background-repeat: no-repeat;
  }
  .pause_audio {
    width: 32px;
    height: 32px;
    margin-left: 50px;
    background-image: url('~@/assets/wxMarketing/play_audio.png');
  }
  .play_audio {
    width: 32px;
    height: 32px;
    margin-left: 50px;
    background-image: url('~@/assets/wxMarketing/pause_audio.png');
  }
  .flex_center {
    display: flex; align-items: center;
    height: 100%;
    padding: 2px;
    /deep/ .title {
      font-weight: 500;
      font-size: 14px;
    }
    /deep/ a {
      text-decoration:none;
      color: #409eff;
    }
  }
  .playerBox {
    position: relative;
    img {
      cursor: pointer;
      width: 30px;
      height: 30px;
      position: absolute;
      left: 50%;
      top: 50%;
      transform: translate(-50%, -50%);
    }
  }
  /deep/ .audioPlayer {
    img {
      width: 144px;
      height: 144px;
    }
  }
}
.flex_column {
  display: flex;
  flex-direction: column;
}
.operation_icon {
  img {
    margin-left: 8px;
    width: 20px;
    height: 20px;
  }
}
.w_block{
  padding: 0;
}
.reset_form_padding{
  padding: 16px 24px!important;
}
.el-form-item{
  margin-bottom: 0;
}
</style>
