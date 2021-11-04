<template>
  <div>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      draggable
      :allow-drop="allowDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            type="text"
            v-if="node.level <= 2"
            size="mini"
            @click="() => append(data)"
          >Append</el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">edit</el-button>
          <el-button
            type="text"
            v-if="node.childNodes.length == 0"
            size="mini"
            @click="() => remove(node, data)"
          >Delete</el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
//  这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//  例如：import 《组件名称》 from '《组件路径》'

export default {
  //  import引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    //这里存放数据
    return {
      maxLevel: 0,
      title: "",
      dialogType: "",
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name"
      },
      dialogVisible: false,
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        productUnit: "",
        icon: ""
      }
    };
  },
  //计算属性 类似于data概念
  computed: {},
  //监控data中的数据变化
  watch: {},
  //方法集合
  methods: {
    batchDelete() {
      let catIds = [];
      let names = [];
      let checkedNodes = this.$refs.menuTree.getCheckedNodes();
      console.log("被选中菜单", checkedNodes);
      checkedNodes.forEach(element => {
        catIds.push(element.catId);
        names.push(element.name);
      });
      this.$confirm(`是否批量删除【${names}】菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false)
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "菜单批量删除成功",
                type: "success"
              });
              this.getMenus();
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除"
          });
        });
    },
    allowDrop(draggingNode, dropNode, type) {
      console.log(draggingNode, dropNode, type);
      this.countLevelNode(draggingNode.data);
      return true;
    },
    countLevelNode(node) {
      if (node.children != null && node.children.length > 0) {
        for (var i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          this.countLevelNode(node.children[i]);
        }
      }
    },
    handleNodeClick(data) {
      console.log(data);
    },
    edit(data) {
      console.log("修改", data);
      this.dialogVisible = true;
      this.dialogType = "edit";
      this.title = "修改分类";
      //发送请求，获取当前节点最新数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
        params: this.$http.adornParams({})
      }).then(({ data }) => {
        console.log("返回的数据", data);
        this.category.catId = data.category.catId;
        this.category.name = data.category.name;
        this.category.icon = data.category.icon;
        this.category.productUnit = data.category.productUnit;
        this.category.parentCid = data.category.parentCid;
        // this.category.catLevel = data.category.catLevel;
        // this.category.showStatus = data.category.showStatus;
        // this.category.sort = data.category.sort;
      });
    },

    editCategory() {
      console.log("三级分类", this.category);
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false)
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "菜单修改成功",
            type: "success"
          });
          //刷新菜单
          this.getMenus();
          this.expandedKey = [this.category.parentCid];
        } else {
          this.$message.error(data.msg);
        }
      });
      //关闭对话框
      this.dialogVisible = false;
    },

    getMenus() {
      this.dataListLoading = true;
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get"
      }).then(({ data }) => {
        console.log("成功获取到菜单数据...", data.data);
        this.menus = data.data;
      });
    },

    submitData(data) {
      if (this.dialogType == "add") {
        this.addCategory(data);
      } else {
        this.editCategory(data);
      }
    },

    append(data) {
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.catId = null;
      this.category.name = "";
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.showStatus = 1;
      this.category.sort = 0;
      this.dialogType = "add";
      this.title = "添加分类";
      console.log("append", data);
    },

    addCategory() {
      console.log("三级分类", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "菜单添加成功",
            type: "success"
          });
          this.getMenus();
          this.expandedKey = [this.category.parentCid];
        } else {
          this.$message.error(data.msg);
        }
      });
      this.dialogVisible = false;
    },

    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false)
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "菜单删除成功",
                type: "success"
                // duration: 1500,
                // onClose: () => {
                //   this.getMenus();
                //   this.expandedKey = [node.parent.data.catId];
                // }
              });
              this.getMenus();
              this.expandedKey = [node.parent.data.catId];
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除"
          });
        });

      console.log("remove", node, data);
    }
  },
  //声明周期 - 创建完成（可以访问当前this实例）
  created() {
    this.getMenus();
  },
  //声明周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {} //如果页面有keep-alive缓存功能，这个函数会触发
};
</script>
<style scoped>
</style>
