<template>
    <div class="margin-20">
        <div>
            <el-row :gutter="20">
                <el-col :span="8">
                    <Panel type="success">
                        <div slot="title">启用作业数 / 总作业数</div>
                        <div slot="content">{{jobTotalInfo.enabledNumber}} / {{jobTotalInfo.totalNumber}}</div>
                    </Panel>
                </el-col>
                <el-col :span="8">
                    <a @click="toAbnormalJobPage">
                        <Panel type="danger">
                            <div slot="title">异常作业数</div>
                            <div slot="content">{{jobTotalInfo.abnormalNumber}}</div>
                        </Panel>
                    </a>
                </el-col>
            </el-row>
        </div>
        <div class="page-container">
            <FilterPageList ref="pageListRef" :get-data="getJobList" :total="total" :order-by="orderBy" :filters="filters" v-loading="tableLoading" element-loading-text="请稍等···">
                <template slot-scope="scope">
                    <el-form :inline="true" class="table-filter">
                        <el-form-item label="">
                            <el-select style="width: 250px;" placeholder="请选择分组" multiple collapse-tags v-model="filters.groups.value" @change="scope.search">
                                <el-option v-for="item in groupList" :label="item" :value="item" :key="item"></el-option>
                            </el-select>
                        </el-form-item>
                        <el-form-item label="">
                            <el-select style="width: 140px;" v-model="filters.status.value" @change="scope.search">
                                <el-option label="全部状态" value=""></el-option>
                                <el-option v-for="item in $option.jobStatusTypes" :label="item.label" :value="item.value" :key="item.value"></el-option>
                            </el-select>
                        </el-form-item>
                        <el-form-item label="">
                            <el-select style="width: 140px;" v-model="filters.jobType.value" @change="scope.search">
                                <el-option label="全部作业类型" value=""></el-option>
                                <el-option v-for="item in $option.jobTypes" :label="item.label" :value="item.value" :key="item.value"></el-option>
                            </el-select>
                        </el-form-item>
                        <el-form-item label="">
                            <el-input :placeholder="filterColumnPlaceholder" v-model.trim="filters[selectColumn].value" @keyup.enter.native="scope.search">
                              <el-select style="width: 120px;" slot="prepend" v-model="selectColumn" @change="selectColumnChange">
                                  <el-option v-for="item in $option.jobOverviewInputFilters" :key="item.value" :label="item.label" :value="item.value"></el-option>
                              </el-select>
                            </el-input>
                        </el-form-item>
                        <el-form-item>
                            <el-button type="primary" icon="el-icon-search" @click="scope.search">查询</el-button>
                        </el-form-item>
                    </el-form>
                    <div class="page-table" v-loading="loading" element-loading-text="请稍等···">
                        <div class="page-table-header">
                            <div class="page-table-header-title"><i class="fa fa-list"></i>作业列表
                                <el-button type="text" @click="scope.search"><i class="fa fa-refresh"></i></el-button>
                            </div>
                            <div class="page-table-header-separator"></div>
                            <div>
                                <el-button @click="batchEnabled()" v-if="$common.hasPerm('job:batchEnable', domainName)"><i class="fa fa-play-circle text-btn"></i>启用</el-button>
                                <el-button @click="batchDisabled()" v-if="$common.hasPerm('job:batchDisable', domainName)"><i class="fa fa-stop-circle text-warning"></i>禁用</el-button>
                                <el-button @click="batchDelete()" v-if="$common.hasPerm('job:batchRemove', domainName)"><i class="fa fa-trash text-danger"></i>删除</el-button>
                                <el-button @click="batchPriority()" v-if="$common.hasPerm('job:batchSetPreferExecutors', domainName)"><i class="fa fa-level-up text-btn"></i>优先</el-button>
                                <el-button @click="batchGroup()"><i class="fa fa-object-group text-btn"></i>分组</el-button>
                            </div>
                            <div class="pull-right">
                                <el-button @click="handleAdd()" v-if="$common.hasPerm('job:add', domainName)"><i class="fa fa-plus-circle text-btn"></i>添加</el-button>
                                <el-button @click="handleImport()" v-if="$common.hasPerm('job:import', domainName)"><i class="fa fa-arrow-circle-o-down text-btn"></i>导入</el-button>
                                <el-button @click="handleExport()"><i class="fa fa-arrow-circle-o-up text-btn"></i>导出</el-button>
                                <el-button @click="handleArrangeLayout()"><i class="fa fa-line-chart text-btn"></i>作业依赖图</el-button>
                            </div>
                        </div>
                        <el-table stripe border ref="multipleTable" @selection-change="handleSelectionChange" :data="jobList" @sort-change="scope.onSortChange" style="width: 100%">
                            <el-table-column type="selection" width="55"></el-table-column>
                            <el-table-column prop="jobName" label="作业名" sortable="custom">
                                <template slot-scope="scope">
                                    <router-link tag="a" :to="{ name: 'job_setting', params: { domain: domainName, jobName: scope.row.jobName } }">
                                        <i class="iconfont icon-java" v-if="scope.row.jobType === 'JAVA_JOB' || scope.row.jobType === 'PASSIVE_JAVA_JOB'"></i>
                                        <i class="iconfont icon-msnui-logo-linux" v-if="scope.row.jobType === 'SHELL_JOB' || scope.row.jobType === 'PASSIVE_SHELL_JOB'"></i>
                                        <i class="fa fa-envelope-o" v-if="scope.row.jobType === 'MSG_JOB'"></i>
                                        {{scope.row.jobName}}
                                    </router-link>
                                </template>
                            </el-table-column>
                            <el-table-column prop="jobType" label="作业类型" width="100px">
                                <template slot-scope="scope"> 
                                    {{$map.jobTypeMap[scope.row.jobType]}}
                                </template>
                            </el-table-column>
                            <el-table-column label="状态" prop="status" width="90px">
                                <template slot-scope="scope"> 
                                    <el-tag :type="statusTag[scope.row.status]" close-transition>{{$map.jobStatusMap[scope.row.status]}}</el-tag>
                                </template>
                            </el-table-column>
                            <el-table-column prop="description" show-overflow-tooltip label="描述">
                                <template slot-scope="scope"> 
                                    {{scope.row.description}}
                                </template>
                            </el-table-column>
                            <el-table-column prop="groups" label="分组">
                                <template slot-scope="scope">
                                    <el-tag v-if="scope.row.groups === ''" type="success" size="mini">未分组</el-tag>
                                    <el-tag v-else type="success" size="mini" style="margin-right: 3px;margin-bottom: 3px;" v-for="item in $array.strToArray(scope.row.groups)" :key="item">{{item}}</el-tag>
                                </template>
                            </el-table-column>
                            <el-table-column prop="shardingTotalCount" label="分片数" width="70px"></el-table-column>
                            <el-table-column prop="shardingList" label="是否已分配分片" width="130px">
                                <template slot-scope="scope">
                                    <span v-if="scope.row.shardingList === ''">-</span>
                                    <el-tag v-else :type="scope.row.shardingList === '已分配' ? 'primary' : ''">{{ scope.row.shardingList }}</el-tag>
                                </template>
                            </el-table-column>
                            <el-table-column label="操作" width="120px" align="center">
                                <template slot-scope="scope">
                                    <el-tooltip content="启用" placement="top" v-if="$common.hasPerm('job:enable', domainName) && (scope.row.status === 'STOPPING' || scope.row.status === 'STOPPED')">
                                        <el-button type="text" @click="handleActive(scope.row, true)"><i class="fa fa-play-circle"></i></el-button>
                                    </el-tooltip>
                                    <el-tooltip content="禁用" placement="top" v-if="$common.hasPerm('job:disable', domainName) && (scope.row.status === 'READY' || scope.row.status === 'RUNNING')">
                                        <el-button type="text" @click="handleActive(scope.row, false)"><i class="fa fa-stop-circle text-warning"></i></el-button>
                                    </el-tooltip>
                                    <el-tooltip content="复制" placement="top" v-if="$common.hasPerm('job:copy', domainName)">
                                        <el-button type="text" @click="handleCopy(scope.row)"><i class="fa fa-clone"></i></el-button>
                                    </el-tooltip>
                                    <el-tooltip content="删除" placement="top" v-if="$common.hasPerm('job:remove', domainName) && scope.row.status === 'STOPPED'">
                                        <el-button type="text" icon="el-icon-delete text-danger" @click="handleDelete(scope.row)"></el-button>
                                    </el-tooltip>
                                </template>
                            </el-table-column>
                        </el-table>
                    </div>
                </template>
            </FilterPageList>
            <div v-if="isJobInfoVisible">
                <job-info-dialog :domain-name="domainName" :job-info="jobInfo" :job-info-title="jobInfoTitle" :job-info-operation="jobInfoOperation" @job-info-success="jobInfoSuccess" @close-dialog="closeDialog"></job-info-dialog>
            </div>
            <div v-if="isBatchPriorityVisible">
                <batch-priority-dialog :job-names-array="jobNamesArray" :domain-name="domainName" @close-dialog="closePriorityDialog" @batch-priority-success="batchPrioritySuccess"></batch-priority-dialog>
            </div>
            <div v-if="isImportVisible">
                <ImportFileDialog :import-data="importData" import-template-url="/console/static/jobTemplate/download" :import-url="importUrl" import-title="导入作业" @close-dialog="closeImportDialog" @import-success="importSuccess"></ImportFileDialog>
            </div>
            <div v-if="isImportResultVisible">
                <import-result-dialog :import-result="importResult" @close-dialog="closeImportResultDialog"></import-result-dialog>
            </div>
            <div v-if="isArrangeLayoutVisible">
                <arrange-layout-dialog :arrange-layout-info="arrangeLayoutInfo" @job-redirect="jobRedirect" @close-dialog="closeArrangeLayoutDialog"></arrange-layout-dialog>
            </div>
            <div v-if="isJobGroupVisible">
                <job-group-dialog :job-groups="jobGroups" @job-group-success="jobGroupSuccess" @close-dialog="closeJobGroupDialog"></job-group-dialog>
            </div>
        </div>
    </div>
</template>

<script>


export default {
  data() {
    return {
      loading: false,
      tableLoading: false,
      isJobInfoVisible: false,
      jobInfoTitle: '',
      jobInfoOperation: '',
      jobInfo: {},
      isBatchPriorityVisible: false,
      isImportVisible: false,
      importResult: [],
      isImportResultVisible: false,
      importData: {
        namespace: this.$route.params.domain,
      },
      importUrl: '',
      isArrangeLayoutVisible: false,
      arrangeLayoutInfo: {},
      jobNamesArray: [],
      jobTotalInfo: {
        totalNumber: 0,
        enabledNumber: 0,
        abnormalNumber: 0,
      },
      filters: this.$option.jobOverviewFilters,
      jobList: [],
      total: 0,
      orderBy: '',
      groupList: [],
      selectColumn: 'jobName',
      statusTag: {
        READY: 'primary',
        RUNNING: 'success',
        STOPPING: 'warning',
        STOPPED: '',
      },
      multipleSelection: [],
      isJobGroupVisible: false,
      jobGroups: [],
    };
  },
  methods: {
    batchGroup() {
      this.batchOperation('分组', (arr) => {
        this.jobGroups = arr.map((obj) => {
          const rObj = {};
          rObj.jobName = obj.jobName;
          rObj.groups = obj.groups;
          return rObj;
        });
        this.isJobGroupVisible = true;
      });
    },
    jobGroupSuccess() {
      this.closeJobGroupDialog();
      this.refreshPage();
      this.$message.successNotify('批量作业分组操作成功');
    },
    closeJobGroupDialog() {
      this.isJobGroupVisible = false;
    },
    handleArrangeLayout() {
      this.$http.get(`/console/namespaces/${this.domainName}/jobs/arrangeLayout`).then((data) => {
        this.arrangeLayoutInfo = data;
        if (this.arrangeLayoutInfo.paths.length > 0) {
          this.isArrangeLayoutVisible = true;
        } else {
          this.$message.errorMessage('很抱歉！该域没有作业依赖');
        }
      })
      .catch(() => { this.$http.buildErrorHandler('请求失败！'); });
    },
    jobRedirect(jobName) {
      const routeData = this.$router.resolve({ name: 'job_setting', params: { domain: this.domainName, jobName } });
      window.open(routeData.href, '_blank');
    },
    closeArrangeLayoutDialog() {
      this.isArrangeLayoutVisible = false;
    },
    getShardings() {
      const jobNamesStr = this.getJobNameArray(this.jobList).join(',');
      this.$http.get(`/console/namespaces/${this.domainName}/jobs/sharding/status`, { jobNames: jobNamesStr }).then((data) => {
        this.jobList.forEach((ele) => {
          this.$set(ele, 'shardingList', data[ele.jobName]);
        });
      })
      .catch(() => { this.$http.buildErrorHandler('获取作业分片分配失败！'); });
    },
    selectColumnChange() {
      this.$option.jobOverviewInputFilters.forEach((ele) => {
        this.filters[ele.value].value = '';
      });
    },
    toAbnormalJobPage() {
      this.$router.push({ name: 'namespace_abnormal_jobs', params: { domain: this.domainName } });
    },
    handleExport() {
      window.location.href = `/console/namespaces/${this.domainName}/jobs/export`;
    },
    handleImport() {
      this.isImportVisible = true;
      this.importUrl = `/console/namespaces/${this.domainName}/jobs/import`;
    },
    closeImportDialog() {
      this.isImportVisible = false;
    },
    importSuccess(importResult) {
      this.isImportVisible = false;
      this.importResult = importResult;
      this.isImportResultVisible = true;
    },
    closeImportResultDialog() {
      this.isImportResultVisible = false;
      this.refreshPage();
    },
    handleAdd() {
      this.isJobInfoVisible = true;
      this.jobInfoTitle = '添加作业';
      this.jobInfoOperation = 'add';
      const jobAddInfo = {
        jobType: 'JAVA_JOB',
        jobName: '',
        jobClass: '',
        cron: '0/5 * * * * ?',
        shardingTotalCount: 1,
        shardingItemParameters: '',
        jobParameter: '',
        timeZone: 'Asia/Shanghai',
        preferList: [],
        queueName: '',
        description: '',
        upStream: [],
        downStream: [],
      };
      this.jobInfo = JSON.parse(JSON.stringify(jobAddInfo));
    },
    closeDialog() {
      this.isJobInfoVisible = false;
    },
    jobInfoSuccess() {
      this.isJobInfoVisible = false;
      this.refreshPage();
      this.$message.successNotify('保存作业操作成功');
    },
    batchEnabled() {
      this.batchOperation('启用', (arr) => {
        const stopedJob = this.getStoptedJobArray(arr);
        if (stopedJob.length === 0) {
          this.$message.errorMessage('没有可以启用的作业,请重新勾选!');
        } else {
          const params = {
            jobNames: this.getJobNameArray(stopedJob).join(','),
          };
          this.handleBatchActive(params, this.getJobNameArray(stopedJob), true);
        }
      });
    },
    batchDisabled() {
      this.batchOperation('禁用', (arr) => {
        const unstopedJob = this.getUnstoptedJobArray(arr);
        if (unstopedJob.length === 0) {
          this.$message.errorMessage('没有可以禁用的作业,请重新勾选!');
        } else {
          const params = {
            jobNames: this.getJobNameArray(unstopedJob).join(','),
          };
          this.handleBatchActive(params, this.getJobNameArray(unstopedJob), false);
        }
      });
    },
    batchDelete() {
      this.batchOperation('删除', (arr) => {
        const stopedJob = this.getStoptedJobArray(arr);
        if (stopedJob.length === 0) {
          this.$message.errorMessage('没有可以删除的作业,请重新勾选!');
        } else {
          const params = {
            jobNames: this.getJobNameArray(stopedJob).join(','),
          };
          const confirmText = stopedJob.length < 10 ? `确认删除作业 ${params.jobNames} 吗?` : `确认删除已选的 ${stopedJob.length} 条作业吗?`;
          this.$message.confirmMessage(confirmText, () => {
            this.$http.delete(`/console/namespaces/${this.domainName}/jobs`, params).then(() => {
              this.refreshPage();
              this.$message.successNotify('批量删除作业操作成功');
            })
            .catch(() => { this.$http.buildErrorHandler('批量删除作业请求失败！'); });
          });
        }
      });
    },
    batchPriority() {
      this.batchOperation('优先Executors', (arr) => {
        const jobNames = this.getJobNameArray(arr);
        this.jobNamesArray = JSON.parse(JSON.stringify(jobNames));
        this.isBatchPriorityVisible = true;
      });
    },
    closePriorityDialog() {
      this.isBatchPriorityVisible = false;
    },
    batchPrioritySuccess() {
      this.isBatchPriorityVisible = false;
      this.refreshPage();
      this.$message.successNotify('批量设置作业的优先Executors成功');
    },
    batchOperation(text, callback) {
      if (this.multipleSelection.length <= 0) {
        this.$message.errorMessage(`请选择要批量${text}的作业！！！`);
      } else {
        const selectedJobNameArray = [];
        this.multipleSelection.forEach((element) => {
          selectedJobNameArray.push(element);
        });
        callback(selectedJobNameArray);
      }
    },
    handleSelectionChange(val) {
      this.multipleSelection = val;
    },
    handleCopy(row) {
      this.$http.get(`/console/namespaces/${this.domainName}/jobs/${row.jobName}/config`).then((data) => {
        const jobCopyInfo = {
          jobType: data.jobType,
          jobName: '',
          jobNameCopied: row.jobName,
          jobClass: data.jobClass,
          cron: data.cron,
          shardingTotalCount: data.shardingTotalCount,
          shardingItemParameters: data.shardingItemParameters,
          timeZone: data.timeZone,
          queueName: data.queueName,
          description: data.description,
          upStream: data.upStream || [],
          downStream: data.downStream || [],
        };
        this.isJobInfoVisible = true;
        this.jobInfoTitle = '复制作业';
        this.jobInfoOperation = 'copy';
        this.jobInfo = JSON.parse(JSON.stringify(jobCopyInfo));
      })
      .catch(() => { this.$http.buildErrorHandler('获取作业信息请求失败！'); });
    },
    handleDelete(row) {
      this.$message.confirmMessage(`确认删除作业 ${row.jobName} 吗?`, () => {
        this.$http.delete(`/console/namespaces/${this.domainName}/jobs/${row.jobName}`).then(() => {
          this.refreshPage();
          this.$message.successNotify('删除作业操作成功');
        })
        .catch(() => { this.$http.buildErrorHandler('删除作业请求失败！'); });
      });
    },
    handleBatchActive(params, jobArray, enabled) {
      const operation = enabled ? '启用' : '禁用';
      const activeRequest = enabled ? 'enable' : 'disable';
      const confirmText = jobArray.length < 10 ? `确定${operation}作业${params.jobNames}吗?` : `确认${operation}已选的 ${jobArray.length} 条作业吗?`;
      this.$message.confirmMessage(confirmText, () => {
        this.batchActiveRequest(params, activeRequest);
      });
    },
    handleActive(row, enabled) {
      const operation = enabled ? '启用' : '禁用';
      const activeRequest = enabled ? 'enable' : 'disable';
      this.$message.confirmMessage(`确定${operation}作业${row.jobName}吗?`, () => {
        this.activeRequest(row.jobName, activeRequest);
      });
    },
    batchActiveRequest(params, reqUrl) {
      this.$http.post(`/console/namespaces/${this.domainName}/jobs/${reqUrl}`, params).then(() => {
        this.$message.successNotify('操作成功');
        this.refreshPage();
      })
      .catch(() => { this.$http.buildErrorHandler(`${reqUrl}请求失败！`); });
    },
    activeRequest(jobName, reqUrl) {
      this.$http.post(`/console/namespaces/${this.domainName}/jobs/${jobName}/${reqUrl}`, '').then(() => {
        this.$message.successNotify('操作成功');
        this.refreshPage();
      })
      .catch(() => { this.$http.buildErrorHandler(`${reqUrl}请求失败！`); });
    },
    getJobNameArray(arr) {
      const jobNameArray = [];
      arr.forEach((ele) => {
        jobNameArray.push(ele.jobName);
      });
      return jobNameArray;
    },
    getUnstoptedJobArray(arr) {
      const resultArr = [];
      arr.forEach((ele) => {
        if (ele.status !== 'STOPPED') {
          resultArr.push(ele);
        }
      });
      return resultArr;
    },
    getStoptedJobArray(arr) {
      const resultArr = [];
      arr.forEach((ele) => {
        if (ele.status === 'STOPPED') {
          resultArr.push(ele);
        }
      });
      return resultArr;
    },
    getJobTotal() {
      return this.$http.get(`/console/namespaces/${this.domainName}/jobs/counts`).then((data) => {
        this.jobTotalInfo.totalNumber = data.totalNumber;
        this.jobTotalInfo.enabledNumber = data.enabledNumber;
        this.jobTotalInfo.abnormalNumber = data.abnormalNumber;
      })
      .catch(() => { this.$http.buildErrorHandler('获取作业总数失败！'); });
    },
    getJobList(filterParams) {
      const params = JSON.parse(JSON.stringify(filterParams));
      if (params.groups && params.groups.length > 0) {
        this.$set(params, 'groups', params.groups.join(','));
      }
      this.tableLoading = true;
      this.$http.get(`/console/namespaces/${this.domainName}/jobs`, params).then((data) => {
        this.jobList = data.jobs;
        this.total = data.totalNumber;
        if (this.jobList) {
          this.getShardings();
        }
      })
      .catch(() => { this.$http.buildErrorHandler('获取作业列表请求失败！'); })
      .finally(() => {
        this.tableLoading = false;
      });
    },
    getGroupList() {
      return this.$http.get(`/console/namespaces/${this.domainName}/jobs/groups`).then((data) => {
        this.groupList = data;
      })
      .catch(() => { this.$http.buildErrorHandler('获取groups失败！'); });
    },
    init() {
      this.loading = true;
      Promise.all([this.getJobTotal(), this.getGroupList()]).then(() => {
        this.loading = false;
      });
    },
    refreshPage() {
      this.$refs.pageListRef.search();
      this.init();
    },
  },
  created() {
    this.init();
  },
  watch: {
    $route: 'init',
  },
  computed: {
    domainName() {
      return this.$route.params.domain;
    },
    filterColumnPlaceholder() {
      let str = '';
      switch (this.selectColumn) {
        case 'jobName':
          str = '请输入作业名';
          break;
        case 'description':
          str = '请输入作业描述';
          break;
        default:
          str = '';
          break;
      }
      return str;
    },
  },
};
</script>
<style lang="sass" scoped>
.devicon-java-plain:before {
    content: "\F144";
}
</style>
