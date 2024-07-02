const identifierList = ref(window.identifierList)



<el-select
								clearable
								v-model="dataForm.identifier"
								@change="handleIdFn"
								placeholder="请输入设备编码"
							>
								<el-option
									v-for="item in identifierList"
									:key="item.identifier"
									:label="item.identifier"
									:value="item.identifier"
								/>
							</el-select>
