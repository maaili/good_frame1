<template>
  <div>
    <sticky :z-index="10" class-name="sub-navbar">
      <el-input v-model="saveActionForm.name" placeholder="action名" style="width: 200px" clearable />
      <el-input v-model="saveActionForm.description" placeholder="描述" style="width: 200px" clearable />
      <el-button-group>
        <el-button :loading="debugBtnLoading" @click="debugAction" type="info">调试</el-button>
        <el-button @click="saveAction" type="success">保存</el-button>
      </el-button-group>
      <span v-if="!isTestCase"><!-- 不是测试用例，显示page select选择page，以及查看page布局信息的el-icon-view -->
        <el-select v-model="saveActionForm.pageId" clearable filterable style="width: 150px" @change="pageSelected" placeholder="选择page">
          <el-option v-for="page in pages" :key="page.id" :label="page.name" :value="page.id" />
        </el-select>
        <el-popover trigger="click" placement="left">
          <div style="width: 1400px;height: 850px;margin-bottom: 10px">
            <android-inspector :canvas-id="canvasId" :img-info="imgInfo" :window-hierarchy-json="windowHierarchyJson" :tree-loading="treeLoading" />
          </div>
          <el-button slot="reference" icon="el-icon-view" :disabled="!saveActionForm.pageId > 0 " />
        </el-popover>
      </span>
      <span v-if="isTestCase"><!-- 测试用例，提供测试集选择 -->
        <el-select v-model="saveActionForm.testSuiteId" clearable filterable style="width: 150px" placeholder="选择测试集">
          <el-option v-for="testSuite in testSuites" :key="testSuite.id" :label="testSuite.name" :value="testSuite.id" />
        </el-select>
      </span>
    </sticky>
    <div class="app-container">
      <el-tabs tab-position="left" style="height: 250px;overflow: auto">
        <el-tab-pane label="方法参数">
          <action-param-list ref="paramList" :is-add="isAdd" />
        </el-tab-pane>
        <el-tab-pane label="局部变量">
          <action-local-var-list ref="localVarList" />
        </el-tab-pane>
        <el-tab-pane label="全局变量">
          <global-var-list />
        </el-tab-pane>
        <el-tab-pane label="返回值">
          <el-input v-model="saveActionForm.returnValue" clearable :disabled="!isAdd" />
        </el-tab-pane>
      </el-tabs>
      <action-step-list ref="stepList" />
    </div>
  </div>
</template>
<script>
import AndroidInspector from '@/pages/android/components/AndroidInspector'
import ActionParamList from '../components/ActionParamList'
import ActionLocalVarList from '../components/ActionLocalVarList'
import GlobalVarList from '../components/GlobalVarList'
import ActionStepList from '../components/ActionStepList'
import Sticky from '@/components/Sticky'
import { getPageList } from '@/api/page'
import { getTestSuiteList } from '@/api/testSuite'
import { addAction, updateAction, getActionList, debugAction } from '@/api/action'
export default {
  components: {
    AndroidInspector,
    ActionParamList,
    ActionLocalVarList,
    GlobalVarList,
    ActionStepList,
    Sticky
  },
  props: {
    isAdd: Boolean,
    isTestCase: Boolean
  },
  data() {
    return {
      saveActionForm: {
        id: undefined,
        name: '',
        description: '',
        type: this.isTestCase ? 3 : 2,
        hasReturnValue: null,
        returnValue: null,
        returnValueDesc: null,
        params: [],
        localVars: [],
        steps: [],
        platform: this.$store.state.project.platform,
        pageId: undefined,
        projectId: this.$store.state.project.id,
        testSuiteId: undefined
      },
      pages: [],
      testSuites: [],
      debugBtnLoading: false,
      // start-传递给AndroidInspctor组件的数据
      canvasId: 'page-action-canvas',
      imgInfo: {
        imgWidth: null,
        imgHeight: null,
        imgUrl: null
      },
      windowHierarchyJson: null,
      treeLoading: false
      // end-传递给AndroidInspctor组件的数据
    }
  },
  async created() {
    if (!this.isTestCase) {
      const { data } = await getPageList({ projectId: this.saveActionForm.projectId })
      this.pages = data
    } else {
      const { data } = await getTestSuiteList({ projectId: this.saveActionForm.projectId })
      this.testSuites = data
    }
    if (!this.isAdd) {
      const editActionId = this.$route.params.actionId
      const { data } = await getActionList({ id: editActionId })
      this.saveActionForm = data[0]
      if (this.saveActionForm.pageId) { // 编辑时，默认绑定了page，需要初始化布局数据，否则点击右上角眼睛看不到数据
        this.initPageWindowHierarchyData(this.saveActionForm.pageId)
      }
      this.$refs.paramList.params = this.saveActionForm.params
      this.$refs.localVarList.localVars = this.saveActionForm.localVars
      this.$refs.stepList.steps = this.saveActionForm.steps
    } else {
      // 复制，传递过来的数据
      this.saveActionForm = this.$route.params
      if (this.saveActionForm.pageId) { // 编辑时，默认绑定了page，需要初始化布局数据，否则点击右上角眼睛看不到数据
        this.initPageWindowHierarchyData(this.saveActionForm.pageId)
      }
      this.$refs.paramList.params = this.saveActionForm.params
      this.$refs.localVarList.localVars = this.saveActionForm.localVars
      this.$refs.stepList.steps = this.saveActionForm.steps
    }
  },
  methods: {
    pageSelected(id) {
      if (id) {
        this.initPageWindowHierarchyData(id)
      }
    },
    initPageWindowHierarchyData(pageId) {
      const currentPage = this.pages.filter(page => page.id === pageId)[0]
      this.imgInfo = {
        imgWidth: currentPage.imgWidth,
        imgHeight: currentPage.imgHeight,
        imgUrl: currentPage.imgUrl
      }
      this.windowHierarchyJson = JSON.parse(currentPage.windowHierarchyJson)
    },
    saveAction() {
      this.saveActionForm.params = this.$refs.paramList.params
      this.saveActionForm.localVars = this.$refs.localVarList.localVars
      this.saveActionForm.steps = this.$refs.stepList.steps
      this.saveActionForm.hasReturnValue = this.saveActionForm.returnValue ? 1 : 0

      // 返回值必须在局部变量里
      if (this.saveActionForm.returnValue) {
        const localVars = this.saveActionForm.localVars.filter(localVar => localVar.name === this.saveActionForm.returnValue)
        if (localVars && localVars.length > 0) {
          this.saveActionForm.returnValueDesc = localVars[0].description
        } else {
          this.$notify.error('返回值必须为局部变量')
          return
        }
      } else {
        this.saveActionForm.returnValueDesc = ''
      }

      if (this.isAdd) {
        addAction(this.saveActionForm).then(response => {
          this.onSaveSuccess(response)
        })
      } else {
        updateAction(this.saveActionForm).then(response => {
          this.onSaveSuccess(response)
        })
      }
    },
    onSaveSuccess(response) {
      this.$notify.success(response.msg)
      if (this.isTestCase) {
        this.$router.push('/action/testcase/list')
      } else {
        this.$router.push('/action/encapsulation/list')
      }
    },
    debugAction() {
      if (!this.$store.state.device.show) {
        this.$notify.error('先选择一台设备使用后才能调试')
        return
      }
      const action = {}
      action.name = this.saveActionForm.name
      action.params = this.$refs.paramList.params
      action.localVars = this.$refs.localVarList.localVars
      action.steps = this.$refs.stepList.selectedSteps
      action.projectId = this.$store.state.project.id
      action.platform = this.$store.state.project.platform
      action.hasReturnValue = 0 // 调试不用处理返回值
      this.debugBtnLoading = true
      debugAction({
        action: action,
        debugInfo: {
          agentIp: this.$store.state.device.agentIp,
          agentPort: this.$store.state.device.agentPort,
          port: this.$store.state.device.port,
          deviceId: this.$store.state.device.id
        }
      }).then(response => {
        this.$notify.success(response.msg)
      }).finally(() => {
        this.debugBtnLoading = false
      })
    }
  }
}
</script>