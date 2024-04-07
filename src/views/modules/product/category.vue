<template>
    <div>
        <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"> </el-switch>
        <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
        <el-button type="danger" @click="batchDelete">批量删除</el-button>
        <el-tree :data="menus" :props="defaultProps" :expand-on-click-node="false" show-checkbox node-key="catId" :default-expanded-keys="expandedKey" :draggable="draggable" :allow-drop="allowDrop"
            @node-drop="handleDrop" ref="treeMenu">
            <span class="custom-tree-node" slot-scope="{ node, data }">
                <span>{{ node.label }}</span>
                <span>
                    <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)">
                        Append
                    </el-button>
                    <el-button type="text" size="mini" @click="() => edit(data)">
                        Edit
                    </el-button>
                    <el-button v-if="node.childNodes.length == 0" type="text" size="mini" @click="() => remove(node, data)">
                        Delete
                    </el-button>
                </span>
            </span>
        </el-tree>

        <!-- 用于添加和修改的表单 数据与category绑定 -->
        <el-dialog :title=title :visible.sync="dialogVisible" width="30%" :close-on-click-modal="false">
            <el-form :model="category">
                <el-form-item label="分类名称">
                    <el-input v-model="category.name" autocomplete="off"></el-input>
                </el-form-item>
                <el-form-item label="商品图标">
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
//这里可以导入其他文件（比如：组件，工具 js，第三方插件 js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
    //import 引入的组件需要注入到对象中才能使用
    components: {},
    props: {},
    data() {
        return {
            pCidList: [], // 保存需要展开的父节点ID
            draggable: false,
            updateNodes: [],
            maxLevel: 0,
            title: "", // 修改或添加操作对话框标题内容
            dialogType: "", // edit add 用于判断是修改还是添加操作
            category: {
                name: "",
                parentCid: -1,
                catLevel: -1,
                showStatus: 1,
                sort: 0,
                catId: null,
                icon: "",
                productUnit: "",
            },
            dialogVisible: false,
            menus: [],
            expandedKey: [],
            defaultProps: {
                children: "children",
                label: "name",
            },
        };
    },
    methods: {
        getMenus() {
            this.$http({
                url: this.$http.adornUrl("/product/category/list/tree"),
                method: "get",
            }).then(({ data }) => {
                console.log("成功获取到菜单数据...", data.data);
                this.menus = data.data;
            });
        },
        batchDelete() {
            let nodes = this.$refs.treeMenu.getCheckedNodes();
            console.log("选中删除的菜单列表", nodes);
            let nameList = [];
            let idList = [];
            for (let i = 0; i < nodes.length; i++) {
                nameList.push(nodes[i].name);
                idList.push(nodes[i].catId);
            }
            console.log("将要删除的商品ID列表 ", idList);
            this.$confirm(
                `此操作将永久刪除菜单【${nameList}】, 是否继续?`,
                "提示",
                {
                    confirmButtonText: "确定",
                    cancelButtonText: "取消",
                    type: "warning",
                }
            )
                .then(() => {
                    this.$http({
                        url: this.$http.adornUrl("/product/category/delete"),
                        method: "post",
                        data: this.$http.adornData(idList, false),
                    }).then(({ data }) => {
                        this.$message({
                            type: "success",
                            message: "删除成功!",
                        });
                        // 刷新菜单
                        this.getMenus();
                    });
                })
                .catch(() => {});
        },
        batchSave() {
            this.$http({
                url: this.$http.adornUrl("/product/category/update/sort"),
                method: "post",
                data: this.$http.adornData(this.updateNodes, false),
            }).then(({ data }) => {
                this.$message({
                    type: "success",
                    message: "修改成功!",
                });
                // 刷新菜单
                this.getMenus();
                // 设置需要默认展开的菜单
                this.expandedKey = this.pCidList;

                // 清空
                this.updateNodes = [];
                this.pCidList = [];
            });
        },
        handleDrop(draggingNode, dropNode, dropType, ev) {
            console.log("handleDrop: ", draggingNode, dropNode, dropType);
            let pCid = 0;
            let siblings = null;
            if (dropType == "before" || dropType == "after") {
                pCid =
                    dropNode.parent.data.catId == undefined
                        ? 0
                        : dropNode.parent.data.catId;
                siblings = dropNode.parent.childNodes;
            } else {
                pCid = dropNode.data.catId;
                siblings = dropNode.childNodes;
            }

            this.pCidList.push(pCid);

            for (let i = 0; i < siblings.length; i++) {
                if (siblings[i].data.catId == draggingNode.data.catId) {
                    // 如果遍历的节点是当前正在拖拽的节点
                    let catLevel = draggingNode.length;
                    if (siblings[i].level != draggingNode.level) {
                        // 当前节点的层级发生变化
                        catLevel = siblings[i].level;
                        // 递归修改子节点的层级
                        this.updateChildNodeLevel(siblings[i]);
                    }
                    // 更新的时候需要修改层级
                    this.updateNodes.push({
                        catId: siblings[i].data.catId,
                        sort: i,
                        parentCid: pCid,
                        catLevel: catLevel,
                    });
                } else {
                    this.updateNodes.push({
                        catId: siblings[i].data.catId,
                        sort: i,
                        parentCid: pCid,
                    });
                }
            }
            console.log("updateNodes", this.updateNodes);
        },
        updateChildNodeLevel(node) {
            if (node.childNodes.length > 0) {
                for (let i = 0; i < node.childNodes.length; i++) {
                    var cNode = node.childNodes[i].data;
                    this.updateNodes.push({
                        catId: cNode.catId,
                        catLevel: node.childNodes[i].level,
                    });
                    // 递归
                    this.updateChildNodeLevel(node.childNodes[i]);
                }
            }
        },
        allowDrop(draggingNode, dropNode, type) {
            // 被拖动节点的深度 + 父节点的总层数不能大于3
            this.maxLevel = draggingNode.data.catLevel;
            this.countNodeLevel(draggingNode);
            let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
            if (type == "inner") {
                return deep + dropNode.level <= 3;
            }
            return deep + dropNode.parent.level <= 3;
        },
        countNodeLevel(node) {
            if (node.childNodes != null && node.childNodes.length > 0) {
                for (let i = 0; i < node.childNodes.length; i++) {
                    let childNode = node.childNodes[i];
                    if (childNode.level > this.maxLevel) {
                        this.maxLevel = childNode.level;
                    }
                    this.countNodeLevel(childNode);
                }
            }
        },
        edit(data) {
            console.log("需要修改的数据", data);
            this.dialogVisible = true;
            this.dialogType = "edit"; // 对话框是修改操作
            this.title = "修改内容";

            // 发送请求获取最新的数据
            this.$http({
                url: this.$http.adornUrl(
                    `/product/category/info/${data.catId}`
                ),
                method: "get",
            }).then(({ data }) => {
                console.log("获取到的数据", data);
                this.category.name = data.data.name;
                this.category.catId = data.data.catId;
                this.category.icon = data.data.icon;
                this.category.productUnit = data.data.productUnit;
                // 用于默认回显父菜单
                this.category.parentCid = data.data.parentCid;
            });
        },
        append(data) {
            console.log("append", data);
            this.dialogVisible = true;
            this.dialogType = "add"; // 对话框是添加操作
            this.title = "添加内容";

            this.category.parentCid = data.catId; // 新增菜单的父菜单是当前点击元素
            this.category.catLevel = data.catLevel * 1 + 1;
            this.category.catId = null; // 设置为空 使用数据库的自增
            this.category.name = "";
            this.category.icon = "";
            this.category.productUnit = "";
            this.category.sort = 0;
            this.category.showStatus = 1;
        },
        submitData() {
            if (this.dialogType == "add") {
                this.addCategory();
            } else if ((this.dialogType = "edit")) {
                this.editCategory();
            }
        },
        editCategory() {
            // 只传输要修改的字段 解析
            var { name, catId, icon, productUnit } = this.category;
            this.$http({
                url: this.$http.adornUrl("/product/category/update"),
                method: "post",
                data: this.$http.adornData(
                    { name, catId, icon, productUnit },
                    false
                ),
            }).then(({ data }) => {
                this.$message({
                    type: "success",
                    message: "修改成功!",
                });
                // 关闭对话框
                this.dialogVisible = false;
                // 刷新菜单
                this.getMenus();
                // 设置需要默认展开的菜单
                this.expandedKey = [this.category.parentCid];
            });
        },
        addCategory() {
            console.log("添加数据", this.category);
            this.$http({
                url: this.$http.adornUrl("/product/category/save"),
                method: "post",
                data: this.$http.adornData(this.category, false),
            }).then(({ data }) => {
                this.$message({
                    type: "success",
                    message: "添加成功!",
                });
                // 关闭对话框
                this.dialogVisible = false;
                // 刷新菜单
                this.getMenus();
                // 设置需要默认展开的菜单
                this.expandedKey = [this.category.parentCid];
            });
        },
        remove(node, data) {
            var ids = [data.catId];
            this.$confirm(
                `此操作将永久刪除菜单【${data.name}】, 是否继续?`,
                "提示",
                {
                    confirmButtonText: "确定",
                    cancelButtonText: "取消",
                    type: "warning",
                }
            )
                .then(() => {
                    this.$http({
                        url: this.$http.adornUrl("/product/category/delete"),
                        method: "post",
                        data: this.$http.adornData(ids, false),
                    }).then(({ data }) => {
                        this.$message({
                            type: "success",
                            message: "删除成功!",
                        });
                        // 刷新菜单
                        this.getMenus();
                        // 设置需要默认展开的菜单
                        this.expandedKey = [node.parent.data.catId];
                    });
                })
                .catch(() => {
                    this.$message({
                        type: "info",
                        message: "已取消删除",
                    });
                });

            console.log("remove", node, data);
        },
    },
    //计算属性 类似于 data 概念
    computed: {},
    //监控 data 中的数据变化
    watch: {},
    //方法集合

    //生命周期 - 创建完成（可以访问当前 this 实例）
    created() {
        this.getMenus();
    },
    //生命周期 - 挂载完成（可以访问 DOM 元素）
    mounted() {},
    beforeCreate() {}, //生命周期 - 创建之前
    beforeMount() {}, //生命周期 - 挂载之前
    beforeUpdate() {}, //生命周期 - 更新之前
    updated() {}, //生命周期 - 更新之后
    beforeDestroy() {}, //生命周期 - 销毁之前
    destroyed() {}, //生命周期 - 销毁完成
    activated() {}, //如果页面有 keep-alive 缓存功能，这个函数会触发
};
</script>
<style scoped>
</style>