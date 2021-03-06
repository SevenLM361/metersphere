<template>
  <div>
    <el-card class="table-card" v-loading="loading">
      <template v-slot:header>
        <ms-table-header :condition.sync="condition" @search="selectByParam" title=""
                         :show-create="false" />
      </template>

      <el-table ref="scenarioTable" border :data="tableData" class="adjust-table ms-select-all" @select-all="select" @select="select"
                v-loading="loading">

        <el-table-column type="selection" width="50"/>

        <ms-table-select-all v-if="!referenced"
                             :page-size="pageSize>total?total:pageSize"
                             :total="total"
                             @selectPageAll="isSelectDataAll(false)"
                             @selectAll="isSelectDataAll(true)"/>

        <el-table-column v-if="!referenced" width="30" :resizable="false" align="center">
          <template v-slot:default="scope">
            <show-more-btn :is-show="isSelect(scope.row)" :buttons="buttons" :size="selectDataCounts"/>
          </template>
        </el-table-column>

        <el-table-column prop="num" label="ID"
                         show-overflow-tooltip/>
        <el-table-column prop="name" :label="$t('api_test.automation.scenario_name')"
                         show-overflow-tooltip/>
        <el-table-column prop="level" :label="$t('api_test.automation.case_level')"
                         show-overflow-tooltip>
          <template v-slot:default="scope">
            <priority-table-item :value="scope.row.level"/>
          </template>
        </el-table-column>
        <el-table-column prop="status" :label="$t('test_track.plan.plan_status')"
                         show-overflow-tooltip>
          <template v-slot:default="scope">
            <plan-status-table-item :value="scope.row.status"/>
          </template>
        </el-table-column>
        <el-table-column prop="tags" :label="$t('api_test.automation.tag')" width="200px">
          <template v-slot:default="scope">
            <div v-for="(itemName,index)  in scope.row.tags" :key="index">
              <ms-tag type="success" effect="plain" :content="itemName"/>
            </div>
          </template>
        </el-table-column>
        <el-table-column prop="userId" :label="$t('api_test.automation.creator')" show-overflow-tooltip/>
        <el-table-column prop="updateTime" :label="$t('api_test.automation.update_time')" width="180">
          <template v-slot:default="scope">
            <span>{{ scope.row.updateTime | timestampFormatDate }}</span>
          </template>
        </el-table-column>
        <el-table-column prop="stepTotal" :label="$t('api_test.automation.step')" show-overflow-tooltip/>
        <el-table-column prop="lastResult" :label="$t('api_test.automation.last_result')">
          <template v-slot:default="{row}">
            <el-link type="success" @click="showReport(row)" v-if="row.lastResult === 'Success'">
              {{ $t('api_test.automation.success') }}
            </el-link>
            <el-link type="danger" @click="showReport(row)" v-if="row.lastResult === 'Fail'">
              {{ $t('api_test.automation.fail') }}
            </el-link>
          </template>
        </el-table-column>
        <el-table-column prop="passRate" :label="$t('api_test.automation.passing_rate')"
                         show-overflow-tooltip/>
        <el-table-column fixed="right" :label="$t('commons.operating')" width="200px" v-if="!referenced">
          <template v-slot:default="{row}">
            <div v-if="trashEnable">
              <ms-table-operator-button :tip="$t('commons.reduction')" icon="el-icon-refresh-left" @exec="reductionApi(row)" v-tester/>
              <ms-table-operator-button :tip="$t('api_test.automation.remove')" icon="el-icon-delete" @exec="remove(row)" type="danger" v-tester/>
            </div>
            <div v-else>
              <ms-table-operator-button :tip="$t('api_test.automation.edit')" icon="el-icon-edit" @exec="edit(row)" v-tester/>
              <ms-table-operator-button class="run-button" :is-tester-permission="true" :tip="$t('api_test.automation.execute')"
                                        icon="el-icon-video-play"
                                        @exec="execute(row)" v-tester/>
              <ms-table-operator-button :tip="$t('api_test.automation.copy')" icon="el-icon-document-copy" type=""
                                        @exec="copy(row)"/>
              <ms-table-operator-button :tip="$t('api_test.automation.remove')" icon="el-icon-delete" @exec="remove(row)" type="danger" v-tester/>
              <ms-scenario-extend-buttons :row="row"/>
            </div>
          </template>
        </el-table-column>
      </el-table>
      <ms-table-pagination :change="search" :current-page.sync="currentPage" :page-size.sync="pageSize"
                           :total="total"/>
      <div>
        <!-- 执行结果 -->
        <el-drawer :visible.sync="runVisible" :destroy-on-close="true" direction="ltr" :withHeader="true" :modal="false"
                   size="90%">
          <ms-api-report-detail @refresh="search" :infoDb="infoDb" :report-id="reportId" :currentProjectId="projectId"/>
        </el-drawer>
        <!--测试计划-->
        <el-drawer :visible.sync="planVisible" :destroy-on-close="true" direction="ltr" :withHeader="false"
                   :title="$t('test_track.plan_view.test_result')" :modal="false" size="90%">
          <ms-test-plan-list @addTestPlan="addTestPlan"/>
        </el-drawer>
      </div>
    </el-card>

    <batch-edit ref="batchEdit" @batchEdit="batchEdit" :typeArr="typeArr" :value-arr="valueArr" :dialog-title="$t('test_track.case.batch_edit_case')">
      <template v-slot:value>
        <environment-select :current-data="{}" :project-id="projectId"/>
      </template>
    </batch-edit>

    <batch-move @refresh="search" @moveSave="moveSave" ref="testBatchMove"/>

  </div>
</template>

<script>
  import MsTableHeader from "@/business/components/common/components/MsTableHeader";
  import MsTablePagination from "@/business/components/common/pagination/TablePagination";
  import ShowMoreBtn from "@/business/components/track/case/components/ShowMoreBtn";
  import MsTag from "../../../common/components/MsTag";
  import {getUUID, getCurrentProjectID} from "@/common/js/utils";
  import MsApiReportDetail from "../report/ApiReportDetail";
  import MsTableMoreBtn from "./TableMoreBtn";
  import MsScenarioExtendButtons from "@/business/components/api/automation/scenario/ScenarioExtendBtns";
  import MsTestPlanList from "./testplan/TestPlanList";
  import MsTableSelectAll from "../../../common/components/table/MsTableSelectAll";
  import {API_CASE_CONFIGS} from "@/business/components/common/components/search/search-components";
  import MsTableOperatorButton from "@/business/components/common/components/MsTableOperatorButton";
  import PriorityTableItem from "../../../track/common/tableItems/planview/PriorityTableItem";
  import PlanStatusTableItem from "../../../track/common/tableItems/plan/PlanStatusTableItem";
  import BatchEdit from "../../../track/case/components/BatchEdit";
  import {WORKSPACE_ID} from "../../../../../common/js/constants";
  import EnvironmentSelect from "../../definition/components/environment/EnvironmentSelect";
  import BatchMove from "../../../track/case/components/BatchMove";

  export default {
    name: "MsApiScenarioList",
    components: {
      BatchMove,
      EnvironmentSelect,
      BatchEdit,
      PlanStatusTableItem,
      PriorityTableItem,
      MsTableSelectAll,
      MsTablePagination,
      MsTableMoreBtn,
      ShowMoreBtn,
      MsTableHeader,
      MsTag,
      MsApiReportDetail,
      MsScenarioExtendButtons,
      MsTestPlanList,
      MsTableOperatorButton
    },
    props: {
      referenced: {
        type: Boolean,
        default: false,
      },
      selectNodeIds: Array,
      trashEnable: {
        type: Boolean,
        default: false,
      },
      moduleTree: {
        type: Array,
        default() {
          return []
        },
      },
      moduleOptions: {
        type: Array,
        default() {
          return []
        },
      }
    },
    data() {
      return {
        loading: false,
        condition: {
          components: API_CASE_CONFIGS
        },
        currentScenario: {},
        schedule: {},
        selection: [],
        tableData: [],
        selectDataRange: 'all',
        currentPage: 1,
        pageSize: 10,
        total: 0,
        reportId: "",
        batchReportId: "",
        content: {},
        infoDb: false,
        runVisible: false,
        planVisible: false,
        projectId: "",
        runData: [],
        report: {},
        selectDataSize: 0,
        selectAll: false,
        buttons: [
          {
            name: this.$t('api_test.automation.batch_add_plan'), handleClick: this.handleBatchAddCase
          },
          {
            name: this.$t('api_test.automation.batch_execute'), handleClick: this.handleBatchExecute
          },
          {
            name: this.$t('test_track.case.batch_edit_case'), handleClick: this.handleBatchEdit
          },
          {
            name: this.$t('test_track.case.batch_move_case'), handleClick: this.handleBatchMove
          }
        ],
        isSelectAllDate: false,
        unSelection: [],
        selectDataCounts: 0,
        typeArr: [
          {id: 'level', name: this.$t('test_track.case.priority')},
          {id: 'status', name: this.$t('test_track.plan.plan_status')},
          {id: 'principal', name: this.$t('api_test.definition.request.responsible'), optionMethod: this.getPrincipalOptions},
          {id: 'environmentId', name: this.$t('api_test.definition.request.run_env'), optionMethod: this.getEnvsOptions},
        ],
        valueArr: {
          level: [
            {name: 'P0', id: 'P0'},
            {name: 'P1', id: 'P1'},
            {name: 'P2', id: 'P2'},
            {name: 'P3', id: 'P3'}
          ],
          status: [
            {name: this.$t('test_track.plan.plan_status_prepare'), id: 'Prepare'},
            {name: this.$t('test_track.plan.plan_status_running'), id: 'Underway'},
            {name: this.$t('test_track.plan.plan_status_completed'), id: 'Completed'}
          ],
          principal: [],
          environmentId: []
        },
      }
    },
    created() {
      this.projectId = getCurrentProjectID();
      this.search();
    },
    watch: {
      selectNodeIds() {
        this.search();
      },
      trashEnable() {
        if (this.trashEnable) {
          this.search();
        }
      },
      batchReportId() {
        this.loading = true;
        this.getReport();
      }
    },
    computed: {
      isNotRunning() {
        return "Running" !== this.report.status;
      }
    },
    methods: {
      selectByParam() {
        this.changeSelectDataRangeAll();
        this.search();
      },
      search() {
        this.condition.filters = ["Prepare", "Underway", "Completed"];
        this.condition.moduleIds = this.selectNodeIds;
        if (this.trashEnable) {
          this.condition.filters = ["Trash"];
          this.condition.moduleIds = [];
        }

        if (this.projectId != null) {
          this.condition.projectId = this.projectId;
        }

        //检查是否只查询本周数据
        this.condition.selectThisWeedData = false;
        this.condition.executeStatus = null;
        this.isSelectThissWeekData();
        switch (this.selectDataRange) {
          case 'thisWeekCount':
            this.condition.selectThisWeedData = true;
            break;
          case 'unExecute':
            this.condition.executeStatus = 'unExecute';
            break;
          case 'executeFailed':
            this.condition.executeStatus = 'executeFailed';
            break;
          case 'executePass':
            this.condition.executeStatus = 'executePass';
            break;
        }
        this.selection = [];
        this.selectAll = false;
        this.unSelection = [];
        this.selectDataCounts = 0;
        let url = "/api/automation/list/" + this.currentPage + "/" + this.pageSize;
        if (this.condition.projectId) {
          this.loading = true;
          this.$post(url, this.condition, response => {
            let data = response.data;
            this.total = data.itemCount;
            this.tableData = data.listObject;
            this.tableData.forEach(item => {
              if (item.tags && item.tags.length > 0) {
                item.tags = JSON.parse(item.tags);
              }
            });
            this.loading = false;
            this.unSelection = data.listObject.map(s => s.id);
          });
        }
      },
      handleCommand(cmd) {
        let table = this.$refs.scenarioTable;
        switch (cmd) {
          case "table":
            this.selectAll = false;
            table.toggleAllSelection();
            break;
          case "all":
            this.selectAll = true;
            break
        }
      },
      handleBatchAddCase() {
        this.planVisible = true;
      },
      handleBatchEdit() {
        this.$refs.batchEdit.open(this.selectDataCounts);
      },
      handleBatchMove() {
        this.$refs.testBatchMove.open(this.moduleTree, [], this.moduleOptions);
      },
      moveSave(param) {
        this.buildBatchParam(param);
        param.apiScenarioModuleId = param.nodeId;
        this.$post('/api/automation/batch/edit', param, () => {
          this.$success(this.$t('commons.save_success'));
          this.$refs.testBatchMove.close();
          this.search();
        });
      },
      batchEdit(form) {
        let param = {};
        param[form.type] = form.value;
        this.buildBatchParam(param);
        this.$post('/api/automation/batch/edit', param, () => {
          this.$success(this.$t('commons.save_success'));
          this.search();
        });
      },
      getPrincipalOptions(option) {
        let workspaceId = localStorage.getItem(WORKSPACE_ID);
        this.$post('/user/ws/member/tester/list', {workspaceId: workspaceId}, response => {
          option.push(...response.data);
        });
      },
      getEnvsOptions(option) {
        this.$get('/api/environment/list/' + this.projectId, response => {
          option.push(...response.data);
          option.forEach(environment => {
            if (!(environment.config instanceof Object)) {
              environment.config = JSON.parse(environment.config);
            }
            environment.name = environment.name + (environment.config.httpConfig.socket ?
              (': ' + environment.config.httpConfig.protocol + '://' + environment.config.httpConfig.socket) : '');
          });
        });
      },
      addTestPlan(plans) {
        let obj = {planIds: plans, scenarioIds: this.selection};

        obj.projectId = getCurrentProjectID();
        obj.selectAllDate = this.isSelectAllDate;
        obj.unSelectIds = this.unSelection;
        obj = Object.assign(obj, this.condition);

        this.planVisible = false;
        this.$post("/api/automation/scenario/plan", obj, response => {
          this.$success(this.$t("commons.save_success"));
        });
      },
      getReport() {
        if (this.batchReportId) {
          let url = "/api/scenario/report/get/" + this.batchReportId;
          this.$get(url, response => {
            this.report = response.data || {};
            if (response.data) {
              if (this.isNotRunning) {
                try {
                  this.content = JSON.parse(this.report.content);
                } catch (e) {
                  throw e;
                }
                this.loading = false;
                this.$success("批量执行成功，请到报告页面查看详情！");
              } else {
                setTimeout(this.getReport, 2000)
              }
            } else {
              this.loading = false;
              this.$error(this.$t('api_report.not_exist'));
            }
          });
        }
      },
      buildBatchParam(param) {
        param.scenarioIds = this.selection;
        param.projectId = getCurrentProjectID();
        param.selectAllDate = this.isSelectAllDate;
        param.unSelectIds = this.unSelection;
        param = Object.assign(param, this.condition);
      },
      handleBatchExecute() {
        this.infoDb = false;
        let url = "/api/automation/run/batch";
        let run = {};
        run.id = getUUID();
        this.buildBatchParam(run);
        this.$post(url, run, response => {
          let data = response.data;
          this.runVisible = false;
          this.batchReportId = run.id;
        });
      },
      select(selection) {
        this.selection = selection.map(s => s.id);

        //统计应当展示选择了多少行
        this.selectRowsCount(this.selection)

        this.$emit('selection', selection);
      },
      isSelect(row) {
        return this.selection.includes(row.id)
      },
      edit(row) {
        this.$emit('edit', row);
      },
      reductionApi(row) {
        row.scenarioDefinition = null;
        row.tags = null;
        let rows = [row];
        this.$post("/api/automation/reduction", rows, response => {
          this.$success(this.$t('commons.save_success'));
          this.search();
        })
      },
      execute(row) {
        this.infoDb = false;
        let url = "/api/automation/run";
        let run = {};
        let scenarioIds = [];
        scenarioIds.push(row.id);
        run.id = getUUID();
        run.projectId = getCurrentProjectID();
        run.scenarioIds = scenarioIds;
        this.$post(url, run, response => {
          let data = response.data;
          this.runVisible = true;
          this.reportId = run.id;
        });
      },
      copy(row) {
        let rowParam = JSON.parse(JSON.stringify(row));
        rowParam.copy = true;
        rowParam.name = 'copy_'+rowParam.name;
        this.$emit('edit', rowParam);
      },
      showReport(row) {
        this.runVisible = true;
        this.infoDb = true;
        this.reportId = row.reportId;
      },
      //是否选择了全部数据
      isSelectDataAll(dataType) {
        this.isSelectAllDate = dataType;
        this.selectRowsCount(this.selection);
        //如果已经全选，不需要再操作了
        if (this.selection.length != this.tableData.length) {
          this.$refs.scenarioTable.toggleAllSelection(true);
        }
      },
      //选择数据数量统计
      selectRowsCount(selection) {
        let selectedIDs = selection;
        let allIDs = this.tableData.map(s => s.id);
        this.unSelection = allIDs.filter(function (val) {
          return selectedIDs.indexOf(val) === -1
        });
        if (this.isSelectAllDate) {
          this.selectDataCounts = this.total - this.unSelection.length;
        } else {
          this.selectDataCounts = this.selection.length;
        }
      },
      //判断是否只显示本周的数据。  从首页跳转过来的请求会带有相关参数
      isSelectThissWeekData() {
        let dataRange = this.$route.params.dataSelectRange;
        this.selectDataRange = dataRange;
      },
      changeSelectDataRangeAll() {
        this.$emit("changeSelectDataRangeAll");
      },
      remove(row) {
        if (this.trashEnable) {
          this.$get('/api/automation/delete/' + row.id, () => {
            this.$success(this.$t('commons.delete_success'));
            this.search();
          });
          return;
        }
        this.$alert(this.$t('api_test.definition.request.delete_confirm') + ' ' + row.name + " ？", '', {
          confirmButtonText: this.$t('commons.confirm'),
          callback: (action) => {
            if (action === 'confirm') {
              let ids = [row.id];
              this.$post('/api/automation/removeToGc/', ids, () => {
                this.$success(this.$t('commons.delete_success'));
                this.search();
              });
            }
          }
        });
      },
    }
  }
</script>

<style scoped>
  /deep/ .el-drawer__header {
    margin-bottom: 0px;
  }
  /deep/ .run-button {
    background-color: #409EFF;
    border-color: #409EFF;
  }
</style>
