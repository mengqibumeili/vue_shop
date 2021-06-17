<template>
  <div>
    <!-- 面包屑导航区域 -->
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>权限管理</el-breadcrumb-item>
      <el-breadcrumb-item>角色列表</el-breadcrumb-item>
    </el-breadcrumb>

    <el-table :data="rolesList">
      <!-- 添加角色按钮 -->
      <el-row>
        <el-col>
          <el-button type="primary"> 添加角色 </el-button>
        </el-col>
      </el-row>

      <!-- 角色列表区域 -->
      <el-table-column type="expand" label="">
        <template slot-scope="scope">
          <el-row
            :class="['bdbottom', 'vcenter', i1 === 0 ? 'bdtop' : '']"
            v-for="(item1, i1) in scope.row.children"
            :key="item1.id"
          >
            <!-- 渲染一级权限 -->
            <el-col :span="5">
              <el-tag @close="removeRightById(scope.row, item1.id)">
                {{ item1.authName }}
              </el-tag>
              <i class="el-icon-caret-right"></i>
            </el-col>
            <!-- 渲染二级和三级权限 -->
            <el-col :span="19">
              <el-row
                v-for="(item2, i2) in item1.children"
                :key="item2.id"
                :class="[i2 === 0 ? '' : 'bdtop', 'vcenter']"
              >
                <!-- 渲染二级权限 -->
                <el-col :span="6">
                  <el-tag
                    type="success"
                    @close="removeRightById(scope.row, item2.id)"
                  >
                    {{ item2.authName }}
                  </el-tag>
                  <i class="el-icon-caret-right"></i>
                </el-col>
                <!-- 渲染三级权限 -->
                <el-col :span="18">
                  <el-tag
                    type="warning"
                    v-for="(item3, i3) in item2.children"
                    :key="item3.id"
                    closable
                    @close="removeRightById(scope.row, item3.id)"
                  >
                    {{ item3.authName }}
                  </el-tag>
                </el-col>
              </el-row>
            </el-col>
          </el-row>
        </template>
      </el-table-column>
      <el-table-column type="index" label="#"></el-table-column>
      <el-table-column label="角色名称" prop="roleName"></el-table-column>
      <el-table-column label="角色描述" prop="roleDesc"></el-table-column>
      <el-table-column label="操作" width="300px">
        <template slot-scope="scope">
          <el-button type="primary" icon="el-icon-edit" size="mini">
            编辑
          </el-button>

          <el-button type="danger" icon="el-icon-delete" size="mini">
            删除
          </el-button>

          <el-button
            type="warning"
            icon="el-icon-setting"
            size="mini"
            @click="showSetRightDialog(scope.row)"
          >
            分配权限
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- 分配权限的对话框 -->
    <el-dialog
      title="分配权限"
      :visible.sync="setRightDialogVisible"
      width="50%"
      @close="setRightDialogClosed"
    >
      <!-- 树形控件 -->
      <el-tree
        :data="rightslist"
        :props="treeProps"
        show-checkbox
        node-key="id"
        default-expand-all
        :default-checked-keys="defkeys"
        ref="treeRef"
      ></el-tree>

      <span slot="footer" class="dialog-footer">
        <el-button @click="setRightDialogVisible = false">取 消</el-button>
        <el-button
          type="primary"
          @click="allotRights"
          >确 定</el-button
        >
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // 获取所有用户列表
      rolesList: [],

      //   控制分配权限对话框的显示与隐藏
      setRightDialogVisible: false,
      rightslist: [],

      //   树形控件的属性绑定对象
      treeProps: {
        label: 'authName',
        children: 'children',
      },

      //   默认选中的节点数组
      defkeys: [],

      // 当前即将分配权限的角色Id
      roleId: '',
    }
  },
  created() {
    // 调用getRolesList,获取用户数据
    this.getRolesList()
  },
  methods: {
    //   获取用户权限列表
    async getRolesList() {
      const { data: res } = await this.$http.get('roles')
      if (res.meta.status != 200)
        return this.$message.console.error('获取用户角色列表失败')
      this.rolesList = res.data
    },

    //   删除用户权限
    async removeRightById(role, rightId) {
      const confirmResult = await this.$confirm(
        '此操作将永久删除该权限, 是否继续?',
        '提示',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning',
        }
      ).catch((err) => err)
      if (confirmResult !== 'confirm') {
        return this.$message.info('已取消删除')
      }

      const { data: res } = await this.$http.delete(
        `roles/${role.id}/rights/${rightId}`
      )
      console.log(res)
      if (res.meta.status !== 200) {
        return this.$message.error('删除用户失败')
      }
      this.$message.success('删除用户成功')
      //   this.getRolesList()
      role.children = res.data
    },

    // 展示分配权限的对话框
    async showSetRightDialog(role) {
      this.roleId = role.id
      const { data: res } = await this.$http.get('rights/tree')
      if (res.meta.status != 200)
        return this.$message.error('获取权限数据失败！')
      this.rightslist = res.data
      this.getLeafKeys(role, this.defkeys)
      this.setRightDialogVisible = true
    },

    // 获取所有的三级权限，并保存到defkey[]中
    getLeafKeys(node, arr) {
      // 如果当前node不包含children属性，则是三级节点
      if (!node.children) {
        return arr.push(node.id)
      }
      node.children.forEach((item) => {
        this.getLeafKeys(item, arr)
      })
    },

    // 监听分配权限对话框的关闭事件
    setRightDialogClosed() {
      this.defkeys = []
    },

    // 点击确定添加权限
    async allotRights() {
      const keys = [
        ...this.$refs.treeRef.getCheckedKeys(),
        ...this.$refs.treeRef.getHalfCheckedKeys()
      ]
      const idStr = keys.join(',')
      console.log(idStr)
      console.log(this.roleId)
      const { data: res } = await this.$http.post(
        `roles/${this.roleId}/rights`,
        { rids: idStr }
      )
      if (res.meta.status != 200) {
        return this.$message.error('获取权限数据失败！')
      }
      this.$message.success('获取权限数据成功！')
      this.getRolesList()
      this.setRightDialogVisible = false
    },
  },
}
</script>

<style lang="less" scoped>
.el-tag {
  margin: 7px;
}
.bdtop {
  border-top: 1px solid #eee;
}
.bdbottom {
  border-bottom: 1px solid #eee;
}
</style>