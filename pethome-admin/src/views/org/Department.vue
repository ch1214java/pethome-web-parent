<template>
	<section>
		<!--工具条-->
		<el-col :span="24" class="toolbar" style="padding-bottom: 0px;">
			<el-form :inline="true" :model="filters">
				<el-form-item>
					<el-input v-model="filters.keyword" placeholder="关键字"></el-input>
				</el-form-item>
				<el-form-item>
					<el-button type="primary" v-on:click="getDepartments" icon="el-icon-search">&nbsp;查询</el-button>
				</el-form-item>
				<el-form-item>
					<el-button type="primary" @click="handleAdd"  icon="el-icon-plus">&nbsp;新增</el-button>
				</el-form-item>
			</el-form>
		</el-col>

		<!--列表-->
		<el-table :data="departments" :height="450" highlight-current-row v-loading="listLoading" @selection-change="selsChange" style="width: 100%;">
			<el-table-column type="selection" width="55">
			</el-table-column>
			<el-table-column type="index" width="60">
			</el-table-column>
			<el-table-column prop="name" label="名称" width="120" sortable>
			</el-table-column>
			<el-table-column prop="sn" label="标识" width="100" sortable>
			</el-table-column>
			<el-table-column prop="dirPath" label="路径" width="100" sortable>
			</el-table-column>
			<el-table-column prop="state" label="状态" width="120" :formatter="formatSex" sortable>
				<template scope="scope">
					<span style="color: green; font-weight: 700" v-if="scope.row.state==0">启用</span>
					<span style="color: red; font-weight: 700" v-else-if="scope.row.state==-1">禁用</span>
					<span style="color: rebeccapurple; font-weight: 700" v-else="scope.row.state">未知</span>
				</template>
			</el-table-column>
			<el-table-column prop="manager.username" label="管理员" min-width="180" sortable>
			</el-table-column>
			<el-table-column prop="parent.name" label="上级部门" min-width="180" sortable>
			</el-table-column>
			<el-table-column fixed="right" label="操作" width="120">
				<template scope="scope">
					<el-button type="primary" icon="el-icon-edit" size="medium" circle @click="handleEdit(scope.$index, scope.row)"></el-button>
					<el-button type="danger" icon="el-icon-delete" size="medium" circle @click="handleDel(scope.$index, scope.row)"></el-button>
				</template>
			</el-table-column>
		</el-table>

		<!--工具条-->
		<el-col :span="24" class="toolbar">
			<el-button size="medium" type="danger" @click="batchRemove" :disabled="this.sels.length===0" icon="el-icon-delete">批量删除</el-button>
			<el-pagination
					background
					layout="total, sizes, prev, pager, next, jumper"
					@size-change="handleSizeChange" @current-change="handleCurrentChange"
					:page-sizes="[10, 50, 100, 500]" :current-page="page" :page-size="pageSize" :total="total" style="float:right;">
			</el-pagination>
		</el-col>
		<!--新增或编辑-->
		<el-dialog :title="title" :visible.sync="departmentFormVisible" :close-on-click-modal="false">
			<el-form :model="departmentForm" label-width="80px" :rules="departmentFormRules" ref="departmentForm">
				<el-form-item label="名称" prop="name">
					<el-input v-model="departmentForm.name" auto-complete="off"></el-input>
				</el-form-item>
				<el-form-item label="标识" prop="sn">
					<el-input v-model="departmentForm.sn" auto-complete="off"></el-input>
				</el-form-item>
				<el-form-item label="状态">
					<el-radio-group v-model="departmentForm.state">
						<el-radio class="radio" :label="0">启用</el-radio>
						<el-radio class="radio" :label="-1">禁用</el-radio>
					</el-radio-group>
				</el-form-item>
				<el-form-item label="管理员" prop="manager">
					<!--
					:key:选择项的key
					:value="item":选择项对应的value也就是需要传给后端的值
					value-key="id"：如果value需要传递对象那么该值必传
					:label="item.username"：选项中显示的内容
					-->
					<el-select v-model="departmentForm.manager" placeholder="请选择" value-key="id">
						<el-option

							v-for="item in employees"
							:key="item.id"
							:label="item.username"
							:value="item">
							<span style="float: left">{{ item.username }}</span>
							<span style="float: right; color: #8492a6; font-size: 13px">{{ item.email }}</span>
						</el-option>
					</el-select>
				</el-form-item>
				<el-form-item label="父级部门" prop="parent">
					<!--<el-input v-model="departmentForm.parent" auto-complete="off"></el-input>-->
					<el-cascader
							v-model="departmentForm.parent"
							:options="deptTree"
							:props="{
							 checkStrictly: true,
							 label:'name',
							 value:'id'}"
							clearable></el-cascader>
				</el-form-item>
			</el-form>
			<div slot="footer" class="dialog-footer">
				<el-button @click.native="departmentFormVisible = false">取消</el-button>
				<el-button type="primary" @click.native="departmentSubmit" :loading="departmentLoading">提交</el-button>
			</div>
		</el-dialog>

	</section>
</template>

<script>

	export default {
		data() {
			return {
				filters: {
                    keyword: ''
				},
				departments: [],
                employees: [],
                deptTree: [],
				total: 0,
				page: 1,
                pageSize: 10,
				listLoading: false,
				sels: [],//列表选中列
                title: '',

				departmentFormVisible: false,//编辑界面是否显示
				departmentLoading: false,//忙等框
				departmentFormRules: {//声明校验规则
					name: [
						{ required: true, message: '请输入姓名', trigger: 'blur' }
					],
                    sn: [
                        { required: true, message: '请输入标识', trigger: 'blur' }
                    ]
				},
				//编辑界面数据
				departmentForm: {
					id: null,
					name: '',
                    sn: '',
					state: 0,
                    manager: null,
					parent: null
				},

			}
		},
		methods: {
			//性别显示转换
			formatSex: function (row, column) {
				return row.state == 0 ? '启用' : row.state == -1 ? '禁用' : '未知';
			},
			handleCurrentChange(val) {
				this.page = val;
				this.getDepartments();
			},
            handleSizeChange(val){
			    this.pageSize = val;
			    this.getDepartments();
			},
			//获取部门列表
			getDepartments() {
				let para = {
                    currentPage: this.page,
					pageSize: this.pageSize,
                    keyword: this.filters.keyword
				};
				this.listLoading = true;
				//NProgress.start();

				this.$http.post("/dept",para)
					.then((result) => {
					    result = result.data;
                        this.listLoading = false;
						this.total = result.total;
						this.departments = result.rows;
					//NProgress.done();
					})
					.catch(result=>{
					    alert("系统错误");
					});
			},
			//删除
			handleDel: function (index, row) {
				this.$confirm('确认删除该记录吗?', '提示', {
					type: 'warning'
				}).then((result) => {
                    this.listLoading = true;
                    //NProgress.start();
                    let url = "/dept/"+row.id;
                    this.$http.delete(url,{})
                        .then(result=>{
                            this.listLoading = false;
                            result = result.data;//直接抽取了一层
                            if(result.success){//true
                                //this.page = 1;//定位第一页
                                this.$message({
									message: '删除成功',
                                    type: 'success'
                                });
                            }else{//false
                                this.$message({
                                    message: result.message,
                                    type: 'error'
                                });
                            }
                            this.getDepartments();//刷新页面
                        }).catch();
                }).catch(() => {

                });
			},
			//显示编辑界面
			handleEdit: function (index, row) {
                //Object.assign({}, row)  克隆数据到新对象
                this.getDeptTree();
                this.title='编辑',
				this.departmentFormVisible = true;//弹出编辑框
                //双向绑定，弹出框一旦改变，原来数据也会随之改变；赋值一个新对象可以避免
				this.departmentForm = Object.assign({}, row);//数据回显

                //回显父级部门
                let dirParth = this.departmentForm.dirPath;
                let temp = [];
                if (dirParth){//数据库中脏数据
                    //['','1','27']
                    let arry = dirParth.trim().split("/");
					if (arry.length == 2){
                        for(let i = 1;i<arry.length;i++){
                            temp[i-1] = parseInt(arry[i]);
                        }
					}else {
                        for(let i = 1;i<arry.length-1;i++){
                            temp[i-1] = parseInt(arry[i]);
                        }
					}
                    this.departmentForm.parent = temp;
					console.log(temp)
                }

                //清空校验规则 需要在回显之后（否则，刷新后第一次点击编辑，没有数据回显）
                this.$refs['departmentForm'].resetFields();
			},
			//显示新增界面
			handleAdd: function () {
                this.getDeptTree();
                this.title='新增',
                this.departmentFormVisible = true;
                this.$refs['departmentForm'].resetFields();
                this.departmentForm = {
                    id: null,
                    name: '',
                    sn: '',
                    state: 0,
                    manager: null,
                    parent: null
				};
			},
			//新增或编辑
			departmentSubmit: function () {
				this.$refs.departmentForm.validate((valid) => {
					if (valid) {
						this.$confirm('确认提交吗？', '提示', {}).then(() => {
							this.departmentLoading = true;
							//NProgress.start();
							let para = Object.assign({}, this.departmentForm);
							//处理数据parent传到后台

							if(para.parent){//如果是顶级部门
								let parent = {"id":para.parent[para.parent.length-1]}
								console.log("parent"+parent)
								para.parent = parent;
								//处理数据dirpath传到后台
								/*let strPath = "/";
								for(let i=0;i<para.parent.length;i++){
								    strPath += para.parent[i] +"/";
								}
								para.dirPath = strPath;*/
							}
                            console.log("==============="+para.parent)
							//数据格式处理
							//para.birth = (!para.birth || para.birth == '') ? '' : util.formatDate.format(new Date(para.birth), 'yyyy-MM-dd');

							this.$http.put("/dept",para)
								.then((result) => {
									this.departmentLoading = false;
									result = result.data;
									if(result.success){
                                        this.$message({
                                            message: '提交成功',
                                            type: 'success'
                                        });
									}else {
                                        this.$message({
                                            message: result.message,
                                            type: 'error'
                                        });
									}
									this.page = 1;
									this.departmentFormVisible = false;
									this.getDepartments();
								})
								.catch(result=>{
									alert("系统错误");
								});
                            this.$refs['departmentForm'].resetFields();
						});
					}
				});
			},
			selsChange: function (sels) {
				this.sels = sels;
			},
			//批量删除
			batchRemove: function () {
			    //.toString()
				var ids = this.sels.map(item => item.id);
				this.$confirm('确认删除选中记录吗？', '提示', {
					type: 'warning'
				}).then(() => {
					this.listLoading = true;
					//let para = { ids: ids };
					this.$http.patch("/dept",ids)
						.then(result=>{
						    result = result.data;
                            if(result.success){
                                this.$message({
                                    message: result.message,
                                    type: 'success'
                                });
                            }else {
                                this.$message({
                                    message: result.message,
                                    type: 'error'
                                });
							}
                            this.listLoading = false;//关闭忙等框
                            this.page = 1;
                            this.getDepartments();
						})
						.catch(result=>{
						    this.$message({
								message:'系统异常',
								type:'error'
							});
						})
				}).catch(() => {

				});
			},
            getEmployees(){
			    this.$http.get("/employee",{})
					.then(result=>{
					    result = result.data;
					    if (result){
					        this.employees = result;
						}
					})
			},
            getDeptTree(){
			    this.$http.get("/dept/deptTree",{})
					.then(result=>{
					    if (result){
					        this.deptTree = this.getTreeData(result.data);
						}
					})
			},
            getTreeData(data){
                for(var i=0;i<data.length;i++){
                    if(data[i].children.length<1){
                        // children若为空数组，则将children设为undefined
                        data[i].children=undefined;
                    }else {
                        // children若不为空数组，则继续 递归调用 本方法
                        this.getTreeData(data[i].children);
                    }
                }
                return data;
            }
		},
		mounted() {
			this.getDepartments();
            this.getEmployees();
		}
	}

</script>

<style scoped>

</style>