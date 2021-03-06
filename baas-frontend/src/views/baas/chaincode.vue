<template>
  <div class="app-container">
    <div class="filter-container">
      <el-input v-model="listQuery.chaincodeName" :placeholder="$t('chaincode.chaincodeName')" style="width: 200px;" class="filter-item" @keyup.enter.native="handleFilter" />
      <el-button v-waves class="filter-item" type="primary" icon="el-icon-search" @click="handleFilter">
        {{ $t('button.search') }}
      </el-button>
      <el-button class="filter-item" style="margin-left: 10px;" type="primary" icon="el-icon-plus" @click="handleCreate">
        {{ $t('button.add') }}
      </el-button>
    </div>
    <el-table
      :key="tableKey"
      v-loading="listLoading"
      :data="list"
      border
      fit
      highlight-current-row
      style="width: 100%;"
    >
      <el-table-column :label="$t('chaincode.id')" prop="id" align="center" width="80">
        <template slot-scope="scope">
          <span>{{ scope.row.id }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('chaincode.chaincodeName')" min-width="80px">
        <template slot-scope="{row}">
          <span>{{ row.chaincodeName }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('chaincode.version')" min-width="40px">
        <template slot-scope="{row}">
          <span>{{ row.version }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('chaincode.created')" min-width="80px">
        <template slot-scope="{row}">
          <span>{{ row.created | parseTime('{y}-{m}-{d} {h}:{i}') }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('chaincode.status')" min-width="40px">
        <template slot-scope="{row}">
          <el-tag v-if="row.status==0" type="info">定义</el-tag>
          <el-tag v-if="row.status==1" type="warning">运行中</el-tag>
        </template>
      </el-table-column>
      <el-table-column :label="$t('chaincode.policy')" min-width="80px">
        <template slot-scope="{row}">
          <span>{{ row.policy }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('chaincode.args')" min-width="120px">
        <template slot-scope="{row}">
          <span>{{ row.args }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t('button.actions')" align="center" width="230" class-name="small-padding fixed-width">
        <template slot-scope="{row}">
          <el-button v-if="row.status==0" type="primary" size="mini" :loading="isload" @click="handleDeploy(row)">
            {{ $t('button.deploy') }}
          </el-button>
          <el-button v-if="row.status==1" type="success" size="mini" :loading="isload" @click="handleUpdate(row)">
            {{ $t('button.upgrade') }}
          </el-button>
          <el-button v-if="row.status==1" type="warning" size="mini" @click="handleOperation(row)">
            {{ $t('button.operation') }}
          </el-button>
          <el-button v-if="row.status==0" size="mini" type="danger" @click="handleDelete(row)">
            {{ $t('button.delete') }}
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination v-show="total>0" :total="total" :page.sync="listQuery.page" :limit.sync="listQuery.limit" @pagination="getList" />

    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogFormVisible">
      <el-form ref="dataForm" :rules="rules" :model="temp" label-position="left" label-width="120px" style="width: 500px; margin-left:50px;">

        <el-form-item :label="$t('chaincode.chaincodeName')" prop="chaincodeName">
          <el-input v-model="temp.chaincodeName" :disabled="isupgrade" />
        </el-form-item>
        <el-form-item :label="$t('chaincode.code')" prop="githubPath">
          <el-upload
            class="upload-demo"
            :action="uploadUrl"
            :on-remove="handleRemove"
            :before-remove="beforeRemove"
            :on-success="handleSuccess"
            multiple
            :limit="1"
            :on-exceed="handleExceed"
            :file-list="fileList"
          >
            <el-button size="small" type="primary">点击上传</el-button>
            <div slot="tip" class="el-upload__tip">只能上传go文件</div>
          </el-upload>
        </el-form-item>
        <el-form-item :label="$t('chaincode.policy')" prop="policy">
          <el-input v-model="temp.policy" disabled />
          <el-popover placement="right" width="300" trigger="click" style="position: absolute;margin-left: 5px;">
            <el-select v-model="policy" placeholder="范围" style="margin:2px 5px;">
              <el-option label="任意" value="OR" />
              <el-option label="全部" value="AND" />
            </el-select>
            <el-select v-model="policymsps" multiple placeholder="请选择" style="margin:2px 5px;">
              <el-option
                v-for="item in msps"
                :key="item.value"
                :label="item.label"
                :value="item.value"
              />
            </el-select>
            <el-button size="mini" style="margin:2px 5px;" @click="makePolicy()">确定</el-button>
            <el-button slot="reference" type="primary" icon="el-icon-edit" circle />
          </el-popover>

          <!-- <span style="color: #E74C3C;">MSP:{{ msps }}</span> -->
          <!-- <a class="el-upload__tip" target="_blank" href="https://hyperledger-fabric.readthedocs.io/en/latest/endorsement-policies.html">查看文档</a> -->
        </el-form-item>
        <el-form-item :label="$t('chaincode.initArgs')">
          <el-tag v-for="tag in argTags" :key="tag" closable :disable-transitions="false" @close="handleClose(tag)">{{ tag }}</el-tag>
          <el-input v-if="inputVisible" ref="saveTagInput" v-model="inputValue" size="small" class="input-new-tag" @keyup.enter.native="handleInputConfirm" @blur="handleInputConfirm" />
          <el-button v-else class="button-new-tag" size="small" @click="showInput">+ New Arg</el-button>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">
          {{ $t('button.cancel') }}
        </el-button>
        <el-button :loading="isload" type="primary" @click="dialogStatus==='create'?createData():updateData()">
          {{ $t('button.confirm') }}
        </el-button>
      </div>
    </el-dialog>

    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogFormVisible2">
      <el-form ref="dataForm2" :rules="rules" :model="temp" label-position="left" label-width="120px" style="width: 500px; margin-left:50px;">
        <el-form-item :label="$t('chaincode.fcn')" prop="fcn">
          <el-input v-model="temp.fcn" />
        </el-form-item>
        <el-form-item :label="$t('chaincode.fcntype')" prop="fcntype">
          <el-select v-model="temp.fcntype" placeholder="请选择">
            <el-option
              v-for="item in options"
              :key="item.value"
              :label="item.label"
              :value="item.value"
            />
          </el-select>
        </el-form-item>
        <el-form-item :label="$t('chaincode.args')">
          <el-tag v-for="tag in argTags" :key="tag" closable :disable-transitions="false" @close="handleClose(tag)">{{ tag }}</el-tag>
          <el-input v-if="inputVisible" ref="saveTagInput" v-model="inputValue" size="small" class="input-new-tag" @keyup.enter.native="handleInputConfirm" @blur="handleInputConfirm" />
          <el-button v-else class="button-new-tag" size="small" @click="showInput">+ New Arg</el-button>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible2 = false">
          {{ $t('button.cancel') }}
        </el-button>
        <el-button :loading="isload" type="primary" @click="operaData()">
          {{ $t('button.confirm') }}
        </el-button>
      </div>
      <div v-for="t in tips" :key="t.val">
        <el-alert :title="t.val" :type="t.type" />
      </div>
    </el-dialog>

    <el-table
      :data="blockData"
      style="width: 100%"
    >
      <el-table-column type="expand">
        <template slot-scope="props">
          <el-form label-position="left" inline class="demo-table-expand">
            <div v-for="t in props.row.transactions" :key="t.no">
              <el-form-item label="交易NO">
                <span>{{ t.no }}</span>
              </el-form-item>
              <el-form-item label="交易ID">
                <span>{{ t.txid }}</span>
              </el-form-item>
              <el-form-item label="通道">
                <span>{{ t.channel }}</span>
              </el-form-item>
              <el-form-item label="状态">
                <span>{{ t.status }}</span>
              </el-form-item>
              <el-form-item label="时间">
                <span>{{ t.timestamp | parseTime('{y}-{m}-{d} {h}:{i}') }}</span>
              </el-form-item>
              <el-form-item label="交易对象">
                <span>{{ t.subject }}</span>
              </el-form-item>
              <el-form-item label="类型">
                <span>{{ t.type }}</span>
              </el-form-item>
              <el-form-item label="行为">
                <div v-html="printJson(t.actions)" />
              </el-form-item>
              <el-form-item label="配置内容">
                <div v-html="printJson(t.config)" />
              </el-form-item>
            </div>
          </el-form>
        </template>
      </el-table-column>
      <el-table-column
        prop="number"
        label="高度"
        width="50"
      />
      <el-table-column
        prop="currentBlockHash"
        label="当前块哈希"
      />
      <el-table-column
        prop="previousBlockHash"
        label="上一个块哈希"
      />
      <el-table-column
        align="center"
      >
        <template slot="header" slot-scope="scope">
          <el-input
            v-model="search"
            placeholder="输入高度,区块哈希,交易哈希搜索"
            @change="queryBlocks(scope)"
          />
        </template>
        <template slot-scope="scope">
          <span>总共 {{ scope.row.transactions.length }} 条交易</span>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script>
import { fetchList, add, upgrade, del, deploy, invoke, query, queryLedger, queryLatestBlocks, queryBlock } from '@/api/chaincode'
import { fetch } from '@/api/channel'
import waves from '@/directive/waves' // waves directive
import Pagination from '@/components/Pagination' // secondary package based on el-pagination
import { parseTime } from '@/utils'
import { mapGetters } from 'vuex'
let vm = {}
export default {
  name: 'ComplexTable',
  components: { Pagination },
  directives: { waves },
  msp() {
    return vm.msps
  },
  filters: {
  },
  data() {
    vm = this
    // 定时刷新
    var intervalId = setInterval(function() {
      var path = sessionStorage.getItem('ROUTE_PATH')
      if (path.indexOf('/baas/chaincode') !== 0) {
        clearInterval(intervalId)
      }
      if (vm.search === '') {
        queryLatestBlocks(vm.channelId).then(response => {
          if (response.code === 0) {
            if (response.data.length > 0 && vm.blockData.length > 0) {
              if (vm.blockData[0].number !== response.data[0].number) {
                vm.blockData = response.data
              }
            }
          }
        })
      }
    }, 10 * 1000)

    return {
      search: '',
      uploadUrl: process.env.VUE_APP_BASE_API + '/upload',
      channelId: 0,
      channel: {
        id: 0,
        userAccount: ''
      },
      tableKey: 0,
      list: null,
      total: 0,
      listLoading: true,
      listQuery: {
        page: 1,
        limit: 10,
        chaincodeName: undefined,
        channelId: undefined
      },
      tips: [],
      options: [{
        value: 'invoke',
        label: 'Invoke'
      }, {
        value: 'query',
        label: 'Query'
      }],
      msps: [],
      policy: '',
      policymsps: [],
      isupgrade: false,
      isload: false,
      temp: {
        chaincodeName: undefined,
        channelId: 0,
        created: 0,
        version: undefined,
        githubPath: undefined,
        userAccount: undefined,
        args: undefined,
        policy: undefined
      },
      dialogFormVisible: false,
      dialogFormVisible2: false,
      dialogStatus: '',
      textMap: {
        update: '编辑链码',
        create: '创建链码',
        opera: '操作链码'
      },
      rules: {
        chaincodeName: [{ required: true, message: 'chaincodeName is required', trigger: 'blur' }],
        githubPath: [{ required: true, message: 'file is required', trigger: 'blur' }],
        policy: [{ required: true, message: 'policy is required', trigger: 'blur' }],
        fcn: [{ required: true, message: 'fcn is required', trigger: 'blur' }],
        fcntype: [{ required: true, message: 'fcntype is required', trigger: 'blur' }]
      },
      fileList: [],
      argTags: [],
      blockData: [],
      inputVisible: false,
      inputValue: ''
    }
  },
  computed: {
    ...mapGetters([
      'account'
    ])
  },
  created() {
    const id = this.$route.params && this.$route.params.id
    this.channelId = Number(id)
    this.getList()
    this.getChannel()
    this.queryBlocks()
    queryLedger(this.channelId).then(response => {
      console.log(response.data)
    })
  },
  methods: {
    makePolicy() {
      var ply = this.policy
      var msp = this.policymsps
      if (ply === '' || msp.length === 0) {
        this.$message.error('范围和组织不能为空')
        return
      }
      this.temp.policy = ply + '(' + msp.join(',') + ')'
    },
    handleClose(tag) {
      this.argTags.splice(this.argTags.indexOf(tag), 1)
      this.temp.args = this.argTags.join(',')
    },
    showInput() {
      this.inputVisible = true
      this.$nextTick(_ => {
        this.$refs.saveTagInput.$refs.input.focus()
      })
    },
    handleInputConfirm() {
      const inputValue = this.inputValue
      if (inputValue) {
        this.argTags.push(inputValue)
        this.temp.args = this.argTags.join(',')
      }
      this.inputVisible = false
      this.inputValue = ''
    },
    handleRemove(file, fileList) {
      console.log(file, fileList)
      this.temp.githubPath = ''
    },
    handleSuccess(response, file, fileList) {
      console.log(response, file, fileList)
      this.temp.githubPath = response
    },
    handleExceed(files, fileList) {
      this.$message.warning('限制选择 1 个文件')
    },
    beforeRemove(file, fileList) {
      return this.$confirm('确定移除' + file.name + '？')
    },
    getChannel() {
      this.channel.id = this.channelId
      this.channel.userAccount = this.account
      fetch(this.channel).then(response => {
        if (response.code === 0) {
          const orgs = response.data.orgs.split(',')
          const msp = []
          orgs.forEach(v => {
            var o = v.toLowerCase().replace(/( |^)[a-z]/g, (L) => L.toUpperCase())
            var org = {}
            org.label = o
            org.value = "'" + o + 'MSP' + '.member' + "'"
            msp.push(org)
          })
          this.msps = msp
        }
      })
    },
    getList() {
      this.listLoading = true
      this.listQuery.channelId = this.channelId
      fetchList(this.listQuery).then(response => {
        this.list = response.data
        this.total = response.total
        // Just to simulate the time of the request
        setTimeout(() => {
          this.listLoading = false
        }, 1.5 * 1000)
      })
    },
    handleFilter() {
      this.listQuery.page = 1
      this.getList()
    },
    resetTemp() {
      this.fileList = []
      this.temp = {
        chaincodeName: undefined,
        channelId: 0,
        created: 0,
        version: undefined,
        githubPath: undefined,
        userAccount: undefined,
        args: undefined,
        policy: undefined
      }
    },
    handleCreate() {
      this.resetTemp()
      this.dialogStatus = 'create'
      this.dialogFormVisible = true
      this.isupgrade = false
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    createData() {
      this.$refs['dataForm'].validate((valid) => {
        if (valid) {
          this.temp.userAccount = this.account
          this.temp.channelId = this.channelId
          add(this.temp).then(() => {
            this.dialogFormVisible = false
            this.$notify({
              title: '成功',
              message: '创建成功',
              type: 'success',
              duration: 2000
            })
            this.getList()
          })
        }
      })
    },
    queryBlocks() {
      if (this.search === '') {
        queryLatestBlocks(this.channelId).then(response => {
          this.blockData = response.data
        })
      } else {
        queryBlock(this.channelId, this.search).then(response => {
          this.blockData = response.data
        })
      }
    },
    handleDeploy(row) {
      this.$confirm('确认部署链码?', '部署链码', {
        confirmButtonText: this.$t('button.confirm'),
        cancelButtonText: this.$t('button.cancel'),
        type: 'warning'
      })
        .then(async() => {
          this.isload = true
          await deploy(row)
          this.$notify({
            title: '成功',
            message: '部署成功',
            type: 'success',
            duration: 2000
          })
          this.isload = false
          this.getList()
        })
        .catch(err => { console.error(err) })
    },
    handleUpdate(row) {
      this.temp = Object.assign({}, row) // copy obj
      this.dialogStatus = 'update'
      this.dialogFormVisible = true
      this.isupgrade = true
      this.fileList = []
      this.argTags = this.temp.args.split(',')
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    updateData() {
      this.$refs['dataForm'].validate((valid) => {
        if (valid) {
          this.isload = true
          const tempData = Object.assign({}, this.temp)
          upgrade(tempData).then(() => {
            this.dialogFormVisible = false
            this.isload = false
            this.$notify({
              title: '成功',
              message: '升级成功',
              type: 'success',
              duration: 2000
            })
            this.getList()
          })
        }
      })
    },
    handleOperation(row) {
      this.temp = Object.assign({}, row) // copy obj
      this.dialogFormVisible2 = true
      this.dialogStatus = 'opera'
      this.argTags = []
      this.$nextTick(() => {
        this.$refs['dataForm2'].clearValidate()
      })
    },
    operaData() {
      this.$refs['dataForm2'].validate((valid) => {
        if (valid) {
          this.isload = true
          this.temp.args = this.temp.fcn + ',' + this.temp.args
          const tempData = Object.assign({}, this.temp)
          if (this.temp.fcntype === 'invoke') {
            invoke(tempData).then(response => {
              this.dialogFormVisible = false
              this.isload = false
              var result = {}
              result.val = response.data
              result.type = 'success'
              this.tips.push(result)
            })
          } else {
            query(tempData).then(response => {
              this.dialogFormVisible = false
              this.isload = false
              var result = {}
              result.val = response.data
              result.type = 'warning'
              this.tips.push(result)
            })
          }
        }
      })
    },
    handleDelete(row) {
      this.$confirm('确认删除链?', '删除链', {
        confirmButtonText: this.$t('button.confirm'),
        cancelButtonText: this.$t('button.cancel'),
        type: 'warning'
      })
        .then(async() => {
          await del(row)
          this.$notify({
            title: '成功',
            message: '删除成功',
            type: 'success',
            duration: 2000
          })
          this.getList()
        })
        .catch(err => { console.error(err) })
    },
    formatJson(filterVal, jsonData) {
      return jsonData.map(v => filterVal.map(j => {
        if (j === 'timestamp') {
          return parseTime(v[j])
        } else {
          return v[j]
        }
      }))
    },
    printJson(msg) {
      if (msg === null || msg === '') {
        return '&nbsp;'
      }
      var rep = '~'
      var jsonStr = ''
      if (typeof (msg) === 'string') {
        jsonStr = JSON.stringify(JSON.parse(msg), null, rep)
      } else {
        jsonStr = JSON.stringify(msg, null, rep)
      }

      var str = ''
      for (var i = 0; i < jsonStr.length; i++) {
        var text2 = jsonStr.charAt(i)
        if (i > 1) {
          var text = jsonStr.charAt(i - 1)
          if (rep !== text && rep === text2) {
            str += '<br/>'
          }
        }
        str += text2
      }
      jsonStr = ''
      for (i = 0; i < str.length; i++) {
        text = str.charAt(i)
        if (rep === text) { jsonStr += '&nbsp;&nbsp;&nbsp;&nbsp;' } else {
          jsonStr += text
        }
        if (i === str.length - 2) { jsonStr += '<br/>' }
      }
      return jsonStr
    }
  }
}
</script>

<style>
  .el-tag + .el-tag {
    margin-left: 5px;
  }
  .button-new-tag {
    margin-left: 10px;
    height: 32px;
    line-height: 30px;
    padding-top: 0;
    padding-bottom: 0;
  }
  .input-new-tag {
    width: 90px;
    margin-left: 10px;
    vertical-align: bottom;
  }

.demo-table-expand {
  font-size: 0;
}
.demo-table-expand label {
  width: 90px;
  color: #99a9bf;
}
.demo-table-expand .el-form-item {
  margin-right: 0;
  margin-bottom: 0;
  width: 100%;
}
</style>
