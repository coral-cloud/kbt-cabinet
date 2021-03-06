<style lang="less">
@import "./list.less";
</style>
<template>
  <div class="search">
    <Row>
      <Col>
        <Card>
          <Row class="operation">
            <Button @click="add" type="primary" icon="md-add">添加开发者</Button>
            <Button @click="delAll" icon="md-trash">批量删除</Button>
            <Button @click="init" icon="md-refresh">刷新</Button>
            <circleLoading v-if="operationLoading"/>
          </Row>
          <Row>
            <Alert show-icon>
              已选择
              <span class="select-count">{{selectCount}}</span> 项
              <a class="select-clear" @click="clearSelectAll">清空</a>
            </Alert>
          </Row>
          <Row>
            <Table
              :loading="loading"
              border
              :columns="columns"
              :data="data"
              sortable="custom"
              @on-sort-change="changeSort"
              @on-selection-change="showSelect"
              ref="table"
            ></Table>

          </Row>
          <Row type="flex" justify="end" class="page">
            <Page
              :current="searchForm.pageNumber"
              :total="total"
              :page-size="searchForm.pageSize"
              @on-change="changePage"
              @on-page-size-change="changePageSize"
              :page-size-opts="[10,20,50]"
              size="small"
              show-total
              show-elevator
              show-sizer
            ></Page>
          </Row>
        </Card>
      </Col>
    </Row>
    <Modal
      :title="modalTitle"
      v-model="dataModalVisible"
      :mask-closable="false"
      :width="500"
      :styles="{top: '30px'}"
    >
      <Form ref="dataFrom" :model="dataFrom" :label-width="70" :rules="dataFromValidate">
        <FormItem label="开发者名称" prop="username">
          <Input v-model="dataFrom.username" autocomplete="off"/>
        </FormItem>
        <FormItem label="企业名称" prop="companyName">
          <Input v-model="dataFrom.companyName" autocomplete="off"/>
        </FormItem>
        <FormItem label="邮箱" prop="email">
          <Input v-model="dataFrom.email" autocomplete="off"/>
        </FormItem>
      </Form>
      <div slot="footer">
        <Button type="text" @click="cancelData">取消</Button>
        <Button type="primary" :loading="submitLoading" @click="submitData">提交</Button>
      </div>
    </Modal>

  </div>
</template>

<script>
import {
  getDeveloperListData,
  addDeveloper,
  editDeveloper,
  deleteDeveloper,
} from "@/api/app";
import circleLoading from "../../view-components/circle-loading.vue";
export default {
  name: "app",
  components: {
    circleLoading
  },
  data() {
    return {
      loading: true,
      operationLoading: false,
      drop: false,
      selectCount: 0,
      selectList: [],
      dataDep: [],
      searchKey: "",
      searchForm: {
        username: "",
        companyName: "",
        email: "",
        pageNumber: 1,
        pageSize: 10,
        sort: "createTime",
        order: "desc",
        startDate: "",
        endDate: ""
      },
      selectDate: null,
      modalType: 0,
      dataModalVisible: false,
      modalTitle: "",
      dataFrom: {
        username: "",
        companyName: "",
        email: "",
      },
      dataFromValidate: {
        username: [
          { required: true, message: "开发者名称", trigger: "blur" }
        ],
        companyName: [
          { required: true, message: "企业名称", trigger: "blur" },
        ],
        email: [
          { required: true, message: "邮箱", trigger: "blur" },
        ]
      },
      submitLoading: false,
      columns: [
        {
          type: "selection",
          width: 60,
          align: "center",
          fixed: "left"
        },
        {
          type: "index",
          width: 60,
          align: "center",
          fixed: "left"
        },
        {
          title: "编号",
          key: "id",
          minWidth: 60,
          sortable: true,
          fixed: "left"
        },
        {
          title: "开发者名称",
          key: "username",
          minWidth: 80,
          sortable: true,
          fixed: "left"
        },
        {
          title: "企业名称",
          key: "companyName",
          minWidth: 80,
          sortable: true,
          fixed: "left"
        },
        {
          title: "邮箱",
          key: "email",
          minWidth: 145,
          sortable: true,
          fixed: "left"
        },
        {
          title: "创建时间",
          key: "createTime",
          sortable: true,
          sortType: "desc",
          width: 150
        },
        {
          title: "更新时间",
          key: "updateTime",
          sortable: true,
          sortType: "desc",
          width: 150
        },
        {
          title: "操作",
          key: "action",
          width: 200,
          align: "center",
          fixed: "right",
          render: (h, params) => {
            return h("div", [
              h(
                "Button",
                {
                  props: {
                    type: "primary",
                    size: "small"
                  },
                  style: {
                    marginRight: "5px"
                  },
                  on: {
                    click: () => {
                      this.edit(params.row);
                    }
                  }
                },
                "编辑"
              ),
              h(
                "Button",
                {
                  props: {
                    type: "error",
                    size: "small"
                  },
                  on: {
                    click: () => {
                      this.remove(params.row);
                    }
                  }
                },
                "删除"
              )
            ]);
          }
        }
      ],
      exportColumns: [
        {
          title: "开发者名称",
          key: "username"
        },
        {
          title: "企业名称",
          key: "companyName"
        },
        {
          title: "更新时间",
          key: "updateTime"
        }
      ],
      data: [],
      exportData: [],
      total: 0,
      dictSex: []
    };
  },
  methods: {
    init() {
      this.getDeveloperList();
    },

    changePage(v) {
      this.searchForm.pageNumber = v;
      this.getDeveloperList();
      this.clearSelectAll();
    },
    changePageSize(v) {
      this.searchForm.pageSize = v;
      this.getDeveloperList();
    },
    selectDateRange(v) {
      if (v) {
        this.searchForm.startDate = v[0];
        this.searchForm.endDate = v[1];
      }
    },
    getDeveloperList() {
      // 多条件搜索用户列表
      this.loading = true;
      // 避免后台默认值
      if (!this.searchForm.type) {
        this.searchForm.type = "";
      }
      if (!this.searchForm.status) {
        this.searchForm.status = "";
      }
      getDeveloperListData(this.searchForm).then(res => {
        this.loading = false;
        if (res.success === true) {
          this.data = res.result.content;
          this.total = res.result.totalElements;
        }
      });
    },
    handleSearch() {
      this.searchForm.pageNumber = 1;
      this.searchForm.pageSize = 10;
      this.getDeveloperList();
    },
    handleReset() {
      this.$refs.searchForm.resetFields();
      this.searchForm.pageNumber = 1;
      this.searchForm.pageSize = 10;
      this.selectDate = null;
      this.searchForm.startDate = "";
      this.searchForm.endDate = "";
      // 重新加载数据
      this.getDeveloperList();
    },
    changeSort(e) {
      this.searchForm.sort = e.key;
      this.searchForm.order = e.order;
      if (e.order === "normal") {
        this.searchForm.order = "";
      }
      this.getDeveloperList();
    },
    cancelData() {
      this.dataModalVisible = false;
    },
    submitData() {
      this.$refs.dataFrom.validate(valid => {
        if (valid) {
          if (this.modalType === 0) {
            // 添加用户 避免编辑后传入id
            delete this.dataFrom.id;
            delete this.dataFrom.status;

            this.submitLoading = true;
            addDeveloper(this.dataFrom).then(res => {
              this.submitLoading = false;
              if (res.success === true) {
                this.$Message.success("操作成功");
                this.getDeveloperList();
                this.dataModalVisible = false;
              }
            });
          } else {
            // 编辑
            this.submitLoading = true;
            editDeveloper(this.dataFrom).then(res => {
              this.submitLoading = false;
              if (res.success === true) {
                this.$Message.success("操作成功");
                this.getDeveloperList();
                this.dataModalVisible = false;
              }
            });
          }
        }
      });
    },
    handleUpload(v) {
      this.dataFrom.avatar = v;
    },
    add() {
      this.modalType = 0;
      this.modalTitle = "添加开发者";
      this.$refs.dataFrom.resetFields();
      this.dataModalVisible = true;
    },
    edit(v) {
      this.modalType = 1;
      this.modalTitle = "编辑开发者";
      this.$refs.dataFrom.resetFields();
      // 转换null为""
      for (let attr in v) {
        if (v[attr] === null) {
          v[attr] = "";
        }
      }
      let str = JSON.stringify(v);
      let data = JSON.parse(str);
      this.dataFrom = data;
      this.dataModalVisible = true;
    },
    remove(v) {
      this.$Modal.confirm({
        title: "确认删除",
        content: "您确认要删除类别 " + v.name + " ?",
        onOk: () => {
          this.operationLoading = true;
          deleteDeveloper(v.id).then(res => {
            this.operationLoading = false;
            if (res.success === true) {
              this.$Message.success("删除成功");
              this.getDeveloperList();
            }
          });
        }
      });
    },
    showSelect(e) {
      this.exportData = e;
      this.selectList = e;
      this.selectCount = e.length;
    },
    clearSelectAll() {
      this.$refs.table.selectAll(false);
    },
    delAll() {
      if (this.selectCount <= 0) {
        this.$Message.warning("您还未选择要删除的数据");
        return;
      }
      this.$Modal.confirm({
        title: "确认删除",
        content: "您确认要删除所选的 " + this.selectCount + " 条数据?",
        onOk: () => {
          let ids = "";
          this.selectList.forEach(function(e) {
            ids += e.id + ",";
          });
          ids = ids.substring(0, ids.length - 1);
          this.operationLoading = true;
          deleteArrange(ids).then(res => {
            this.operationLoading = false;
            if (res.success === true) {
              this.$Message.success("删除成功");
              this.clearSelectAll();
              this.getDeveloperList();
            }
          });
        }
      });
    }
  },
  mounted() {
    this.init();
  }
};
</script>