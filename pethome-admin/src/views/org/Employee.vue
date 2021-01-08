<template>
	<section>
		<!--工具条-->
		<el-col :span="24" class="toolbar" style="padding-bottom: 0px;">
			<el-form :inline="true" :model="filters">
				<el-form-item>
					<el-input v-model="filters.keyword" placeholder="关键字"></el-input>
				</el-form-item>
				<el-form-item>
					<el-button type="primary" v-on:click="getEmployees" icon="el-icon-search">&nbsp;查询</el-button>
				</el-form-item>
				<el-form-item>
					<el-button type="primary" @click="handleAdd"  icon="el-icon-plus">&nbsp;新增</el-button>
				</el-form-item>
			</el-form>
		</el-col>

		<!--列表-->
		<!--
		username;
		email;
		phone;
		password;
		age;
		state;-->
		<el-table :data="employees" :height="450" highlight-current-row v-loading="listLoading" @selection-change="selsChange" style="width: 100%;">
			<el-table-column type="selection" width="55">
			</el-table-column>
			<el-table-column type="index" width="60">
			</el-table-column>
			<el-table-column prop="username" label="姓名" width="120" sortable>
			</el-table-column>
			<el-table-column prop="email" label="邮件" width="180" sortable>
			</el-table-column>
			<el-table-column prop="phone" label="电话" width="120" sortable>
			</el-table-column>
			<el-table-column prop="age" label="年龄" width="100" sortable>
			</el-table-column>
			<el-table-column prop="state" label="状态" width="120" :formatter="formatSex" sortable>
				<template scope="scope">
					<span style="color: green; font-weight: 700" v-if="scope.row.state==1">启用</span>
					<span style="color: red; font-weight: 700" v-else-if="scope.row.state==0">禁用</span>
					<span style="color: rebeccapurple; font-weight: 700" v-else="scope.row.state">未知</span>
				</template>
			</el-table-column>
			<el-table-column prop="shop.name" label="店铺" min-width="180" sortable>
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
		<el-dialog :title="title" :visible.sync="employeeFormVisible" :close-on-click-modal="false">
			<el-form :model="employeeForm" label-width="80px" :rules="employeeFormRules" ref="employeeForm">
				<el-form-item label="姓名" prop="username">
					<el-input v-model="employeeForm.username" auto-complete="off"></el-input>
				</el-form-item>
				<el-form-item label="邮件" prop="email">
					<el-input v-model="employeeForm.email" auto-complete="off"></el-input>
				</el-form-item>
				<el-form-item label="电话" prop="phone">
					<el-input v-model.number="employeeForm.phone" auto-complete="off" placeholder="请输入11位手机号" maxlength="11"></el-input>
				</el-form-item>
				<el-form-item label="年龄" prop="age">
					<el-input v-model.number="employeeForm.age" auto-complete="off" placeholder="请输入年龄" maxlength="3" min="1" max="200"></el-input>
				</el-form-item>
				<el-form-item label="状态">
					<el-radio-group v-model="employeeForm.state">
						<el-radio class="radio" :label="1">启用</el-radio>
						<el-radio class="radio" :label="0">禁用</el-radio>
					</el-radio-group>
				</el-form-item>
				<el-form-item label="门店" prop="shop">
					<!--
					:key:选择项的key
					:value="item":选择项对应的value也就是需要传给后端的值
					value-key="id"：如果value需要传递对象那么该值必传
					:label="item.username"：选项中显示的内容
					-->
					<el-select v-model="employeeForm.shop" placeholder="请选择" value-key="id">
						<el-option
								v-for="item in shops"
								:key="item.id"
								:label="item.name"
								:value="item">
							<span style="float: left">{{ item.name }}</span>
						</el-option>
					</el-select>
				</el-form-item>
			</el-form>
			<div slot="footer" class="dialog-footer">
				<el-button @click.native="employeeFormVisible = false">取消</el-button>
				<el-button type="primary" @click.native="employeeSubmit" :loading="employeeLoading">提交</el-button>
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
				employees: [],
                shops: [],
                employeeTree: [],
				total: 0,
				page: 1,
                pageSize: 10,
				listLoading: false,
				sels: [],//列表选中列
                title: '',

				employeeFormVisible: false,//编辑界面是否显示
				employeeLoading: false,//忙等框
				employeeFormRules: {//声明校验规则
                    username: [
						{ required: true, message: '请输入姓名', trigger: 'blur' }
					],
                    email: [
                        { required: true, message: '请输入邮箱地址', trigger: 'blur' },
                        { type: 'email', message: '请输入正确的邮箱地址', trigger: ['blur', 'change'] }
                    ],
					age: [
                        { required: true, message: '年龄不能为空'},
                        { type: 'number', message: '年龄必须为数字值'}
					],
                    phone: [
                        { required: true, message: '电话不能为空'}
					]
				},
				//编辑界面数据
				employeeForm: {
					id: null,
					username: '',
                	email: '',
                	phone: '',
                	age: '',
					state: 1,
					shop: null
				},

			}
		},
		methods: {
			//性别显示转换
			formatSex: function (row, column) {
				return row.state == 1 ? '启用' : row.state == 0 ? '禁用' : '未知';
			},
			handleCurrentChange(val) {
				this.page = val;
				this.getEmployees();
			},
            handleSizeChange(val){
			    this.pageSize = val;
			    this.getEmployees();
			},
			//获取员工列表
			getEmployees() {
				let para = {
                    currentPage: this.page,
					pageSize: this.pageSize,
                    keyword: this.filters.keyword
				};
				this.listLoading = true;
				//NProgress.start();

				this.$http.post("/employee",para)
					.then((result) => {
					    result = result.data;
                        this.listLoading = false;
						this.total = result.total;
						this.employees = result.rows;
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
                    let url = "/employee/"+row.id;
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
                            this.getEmployees();//刷新页面
                        }).catch();
                }).catch(() => {

                });
			},
			//显示编辑界面
			handleEdit: function (index, row) {
                //Object.assign({}, row)  克隆数据到新对象
                // this.getEmployeeTree();
                this.title='编辑',
				this.employeeFormVisible = true;//弹出编辑框
                //双向绑定，弹出框一旦改变，原来数据也会随之改变；赋值一个新对象可以避免
				this.employeeForm = Object.assign({}, row);//数据回显
                //清空校验规则 需要在回显之后（否则，刷新后第一次点击编辑，没有数据回显）
                this.$refs['employeeForm'].resetFields();
			},
			//显示新增界面
			handleAdd: function () {
                // this.getEmployeeTree();
                this.title='新增',
                this.employeeFormVisible = true;
                this.$refs['employeeForm'].resetFields();
                this.employeeForm = {
                    id: null,
                    username: '',
                    email: '',
                    phone: '',
                    age: '',
                    state: 1,
                    shop: null
				};
			},
			//新增或编辑
			employeeSubmit: function () {
				this.$refs.employeeForm.validate((valid) => {
					if (valid) {
						this.$confirm('确认提交吗？', '提示', {}).then(() => {
							this.employeeLoading = true;
							//NProgress.start();
							let para = Object.assign({}, this.employeeForm);
							//数据格式处理
							this.$http.put("/employee",para)
								.then((result) => {
									this.employeeLoading = false;
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
									this.employeeFormVisible = false;
									this.getEmployees();
								})
								.catch(result=>{
									alert("系统错误");
								});
                            this.$refs['employeeForm'].resetFields();
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
					this.$http.patch("/employee",ids)
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
                            this.getEmployees();
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
            getShops(){
			    this.$http.get("/shop",{})
					.then(result=>{
					    result = result.data;
					    if (result){
					        this.shops = result;
						}
					})
			},
           /* getEmployeeTree(){
			    this.$http.get("/employee/employeeTree",{})
					.then(result=>{
					    if (result){
					        this.employeeTree = this.getTreeData(result.data);
						}
					})
			},*/
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
			this.getEmployees();
            this.getShops();
		}
	}

</script>

<style scoped>

</style>