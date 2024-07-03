<template>
	<el-dialog ref="dom" v-model="dialogShow" title="属性值更新" width="20%">
		<div class="align-center">
			<span style="width: 100px">修改值：</span>
			<el-switch
				v-if="props.option.valueType === 'BOOLEAN'"
				:model-value="switchValue"
				@change="changeInputValue"
				active-text="开启"
				inactive-text="关闭"
			/>
			<el-input
				v-else
				type="number"
				v-model.number="inputValue"
				@keyup.enter.exact="setData"
			></el-input>
		</div>
		<template #footer>
			<span class="dialog-footer">
				<el-button @click="emits('close')">取消</el-button>
				<el-button type="primary" @click="setData"> 确定 </el-button>
			</span>
		</template>
	</el-dialog>
</template>

<script setup lang="ts">
import { getIotData, setIotData, getIotDetail } from '@/api/request.js'
import useStore from '@/store'
const store = useStore()
const dialogShow = ref(true)

const switchValue = computed(() => (inputValue.value === 'false' ? false : true))
// const dialogVisible = defineModel<Boolean>({})
const inputValue = ref('')
const dom = ref(null)
const props = defineProps({
	dataIdentifier: {
		default: '',
	},
	option: {
		defaule: {},
	},
	dialogVisible: {
		default: false,
	},
})
const emits = defineEmits(['close'])
const changeInputValue = v => {
	inputValue.value = v + ''
	console.log(switchValue)
}
const setData = async () => {
	const deviceName = props.dataIdentifier.split('_')[0] + '_' + props.dataIdentifier.split('_')[1]
	const detailRes = await getIotDetail(deviceName)
	const gatewayIdentifier = detailRes.data?.data?.regCodeIdentifier
	if (!gatewayIdentifier) return ElMessage.error('属性获取失败')
	const params = {
		sendType: 2,
		sendData: [
			{
				gatewayIdentifier,
				deviceName,
				data: [
					{
						measurementIdentifier: props.dataIdentifier,
						sendContent: inputValue.value,
					},
				],
			},
		],
	}
	const res = await setIotData(params)
	if (res?.data?.data?.length) {
		ElMessage.success('修改成功')
		//FIXME:
		window.tempRealData = inputValue.value

		// window.mapDialog.getData(globalData.graphicsParameters[store.activeId])
		// dialogVisible.value = false
		emits('close')
	} else ElMessage.error('修改失败')
}

// const getData = async () => {
//   // let data = await getDataByApi(params.data)
//   const arr = params.data.keyMap.map((v) => v.key)
//   if (!arr.length) return ElMessage.error('请先配置字段映射')
//   const res = await getIotData({ propertyIdentifiers: arr })
//   const data = res?.data?.data
//   if (!Array.isArray(data)) return ElMessage.error('返回数据错误')
//   list.value = params.data.keyMap.map((v, index) => ({
//     value: data[index].dataValue,
//     name: v.name,
//     dataIdentifier: data[index].dataIdentifier,
//   }))
// }
onMounted(() => {
	console.warn(props.option)
})
</script>

<style lang="scss" scoped>
.popup {
	position: fixed;
	top: 100px;
	left: 300px;
	z-index: 10;
	pointer-events: auto;
}
</style>


temp1

<template>
	<div
		v-if="dialogStyle === 'custom2'"
		:id="`mapDialog_${props.id}`"
		ref="popupDomRef"
		:style="{
			background: deviceStatusStr === '运行中' ? 'rgb(111, 238, 200)' : '',
		}"
		class="custom2 hc-dialog"
		@click="toggleStatus"
	>
		<div flex items-center px-1>
			<i class="huichuan-iconfont icon-kaiguan"></i>
			<span ml-2>{{ deviceStatusStr === '运行中' ? '  关' : '  开' }}</span>
		</div>
	</div>
	<div
		v-else
		:class="`hc-marker-wrap hc-dialog mod1 animate__animated animate__bounceIn ${dialogStyle}`"
		:id="`mapDialog_${props.id}`"
		ref="popupDomRef"
	>
		<div class="hc-marker-title">
			<div class="hc-marker-title-dot"></div>
			<!-- <div class="hc-marker-title-label">${data.dialogOption.dialogTitle}</div> -->
			<div class="hc-marker-title-label">{{ baseData.dialogTitle }}</div>
			<i
				class="huichuan-iconfont icon-guanbi gl_close"
				v-if="!baseData.isConstantDialog || baseData.dataTypeList.includes(1)"
				@click="closeDialog"
			></i>
			<span
				v-else-if="baseData.headStatusVisible && deviceStatusStr"
				class="status-title"
				absolute
				right-0
				:style="{
					color: deviceStatusStr === '运行中' ? 'rgb(111, 238, 200)' : '#e73a3a',
				}"
				>{{ deviceStatusStr }}</span
			>
		</div>
		<div class="hc-marker-divider-line" v-if="isEmptyContent">
			<div class="line"></div>
			<div class="line"></div>
		</div>
		<div class="hc-marker-item-wrap" v-if="isSameDataType('REALTIME_DATA')">
			<div class="hc-marker-item" v-for="(n, k) in list" :key="k">
				<template
					v-if="
						(!isScrollRef || !n.isScroll) &&
						!(Object.keys(formatterObj).includes(n.value) && baseData.headStatusVisible)
					"
				>
					<div class="hc-marker-item-label">
						{{ n.name }}
					</div>
					<div style="width: 6rem" class="hc-marker-item-value">
						<ElTooltip :content="formatterValue(n.value)" placement="top">
							<span class="bold" :id="n.dataIdentifier" :style="styleFn(n.value)">{{
								formatterValue(n.value)
							}}</span>
						</ElTooltip>
					</div>
					<div class="hc-marker-item-modified" @click="modifiedValue(n)">修改</div>
				</template>
			</div>
		</div>

		<div v-if="isSameDataType('HISTORY_DATA')" class="table-wrap">
			<el-table style="max-width: 35rem" :data="tableData" show-overflow-tooltip height="230px">
				<el-table-column v-for="(n, k) in columns" :key="n.prop" :prop="n.prop" :label="n.label" />
			</el-table>
			<div class="pagination-wrap">
				<el-pagination
					layout="prev, pager, next"
					:page-size="5"
					:total="total"
					@current-change="getHistoryDataFn"
				/>
			</div>
		</div>
		<div class="bottom" v-if="isEmptyContent" @click="toggleScroll">
			<span>{{ isScrollRef ? '展开' : '收起' }}</span>
			<i :class="`huichuan-iconfont ${isScrollRef ? 'icon-zhankai' : 'icon-shouqi1'}`"></i>
		</div>
	</div>
</template>

<script setup lang="ts">
import { getIotData, setIotData, getIotDetail, getHisToryData } from '@/api/request.js'
import createGraphicDom from './createGraphicDom.js'
import EventBus from '@/utils/bus'
import useStore from '@/store'
import { ElTooltip } from 'element-plus'
const store = useStore()
const isScrollRef = ref(true)
const props = defineProps({
	id: {
		default: '',
		type: String,
	},
	warningData: {
		default: () => ({}),
		type: Object,
	},
})
const deviceStatusStr = ref('')
const tableData = ref([])
const columns = ref([])
const dialogStyle = ref('')
let domGraphic
let criticalId = ''
let previousCurrentPage = 0
const params = globalData.graphicsParameters[props.id]
const popupDomRef = ref()
const list = ref([
	// { name: '名称', value: '数据' }
])
type baseDataType = {
	dialogTitle: string
	isConstantDialog: boolean
	dataTypeList: ['REALTIME_DATA'?, 'HISTORY_DATA'?, 'ALARM_DATA'?]
	headStatusVisible: boolean
}
const toggleScroll = () => {
	isScrollRef.value = !isScrollRef.value
}

const toggleStatus = () => {
	const bool = deviceStatusStr.value === '运行中' ? 'false' : 'true'
	const item = list.value.find(v => {
		return Object.keys(formatterObj).includes(v.value)
	})
	if (item?.dataIdentifier) setData(item.dataIdentifier, bool)
}

const baseData = reactive<baseDataType>({
	dialogTitle: '标题',
	isConstantDialog: false,
	dataTypeList: [],
	headStatusVisible: false,
})
const total = ref(0)

const isSameDataType = type => {
	return baseData.dataTypeList.includes(type)
}

const isEmptyContent = computed(
	() =>
		!!list.value.filter(
			n => !(Object.keys(formatterObj).includes(n.value) && baseData.headStatusVisible)
		).length
)
const formatterObj = {
	false: '非运行',
	true: '运行中',
}

const styleFn = value => {
	let color = value > 0 ? 'rgb(111, 238, 200)' : '#e73a3a'
	if (value === 'false') {
		deviceStatusStr.value = formatterObj[value as 'false']
		color = '#e73a3a'
	} else if (value === 'true') {
		color = 'rgb(111, 238, 200)'
		deviceStatusStr.value = formatterObj[value as 'true']
	}
	return {
		color,
	}
}

const formatterValue = (v: keyof typeof formatterObj) => {
	const value = formatterObj[v]
	return value || v
}

const setData = async (dataIdentifier, value) => {
	const deviceName = dataIdentifier.split('_')[0] + '_' + dataIdentifier.split('_')[1]
	const detailRes = await getIotDetail(deviceName)
	const gatewayIdentifier = detailRes.data?.data?.regCodeIdentifier
	if (!gatewayIdentifier) return ElMessage.error('属性获取失败')
	const paramsObj = {
		sendType: 2,
		sendData: [
			{
				gatewayIdentifier,
				deviceName,
				data: [
					{
						measurementIdentifier: dataIdentifier,
						sendContent: value,
					},
				],
			},
		],
	}
	const res = await setIotData(paramsObj)
	if (res?.data?.data?.length) {
		ElMessage.success('修改成功')
		//FIXME:
		// window.tempRealData = inputValue.value
	} else ElMessage.error('修改失败')
}

const getHistoryDataFn = async page => {
	const arr0 = params.data.keyMap.map(v => v.key)
	if (!arr0.length) return ElMessage.error('请先配置字段映射')
	if (!params.data.historyDataRange[0] || !params.data.historyDataRange[1])
		return ElMessage.error('请选择时间范围')
	const res1 = await getIotData({ propertyIdentifiers: arr0 })
	const resData = res1?.data?.data
	const arr = params.data.keyMap.map(v => {
		return {
			prop: v.key,
			label: v.name,
			id: v.id,
		}
	})
	arr.unshift({
		prop: 'collectTimeFormat',
		label: '采集时间',
	})
	columns.value = arr
	const instanceIds = resData.map(v => v.instanceId).join('::')
	const beginTime = +new Date(params.data.historyDataRange[0])
	const endTime = +new Date(params.data.historyDataRange[1])
	const formData = new FormData()
	let offset = page - previousCurrentPage
	previousCurrentPage = page || 1
	formData.append('$page', previousCurrentPage)
	formData.append('$pageSize', 5 * resData.length)
	formData.append('recordType', 1)
	formData.append(
		'$filter',
		`instanceId eq '${instanceIds}',$queryTime gt ${beginTime},$queryTime lt ${endTime}`
	)
	if (page) {
		formData.append('criticalId', criticalId)
		formData.append('offset', offset)
	}
	const res = await getHisToryData(formData)
	const data = res.data.data?.result
	if (!data?.length) {
		tableData.value = []
		total.value = 0
		return
	}
	criticalId = data.at(-1).id
	total.value = (res.data.data.pageInfo.leftPage + res.data.data.pageInfo.page) * 5
	const obj = Object.groupBy(data, ({ collectTimeFormat }) => collectTimeFormat)
	const list = Object.values(obj).map(arrItem => {
		const temp = {}
		arrItem.forEach(v => {
			temp[v.dataIdentifier] = v.dataValue
		})
		temp.collectTimeFormat = arrItem[0].collectTimeFormat
		return temp
	})
	tableData.value = list
}

const getData = async () => {
	if (isSameDataType('HISTORY_DATA')) getHistoryDataFn(1)
	else {
		const arr = params.data.keyMap.map(v => v.key)
		if (!arr.length) return ElMessage.error('请先配置字段映射')
		const res = await getIotData({ propertyIdentifiers: arr })
		const resData = res?.data?.data
		if (!Array.isArray(resData)) return ElMessage.error('返回数据错误')
		let reqList = <any>[]
		resData.map(item => {
			if (arr.includes(item.dataIdentifier)) {
				reqList.push(item)
			}
		})
		updateData(reqList)
	}
}

const updateData = data => {
	list.value = params.data.keyMap.map((v, index) => {
		const obj = data.find(item => item.dataIdentifier === v.key)
		if (obj)
			return {
				value: obj.dataValue,
				valueType: obj.dataType,
				name: v.name,
				isScroll: v.isScroll || false,
				dataIdentifier: obj.dataIdentifier,
			}
		return {
			value: '暂无数据',
			name: v.name,
			valueType: obj?.dataType,
			isScroll: v.isScroll || false,
			dataIdentifier: v.identifier,
		}
	})
	//极简模式，需要手动更新设备状态
	if (dialogStyle.value === 'custom2') {
		const item = list.value.find(v => v.value === 'true' || v.value === 'false')
		if (item) {
			deviceStatusStr.value = formatterObj[window.testStatus ?? item.value]
		}
	}
	if (baseData.headStatusVisible) setRightHeadText()
}

let tableCurrentItemIndex = null
const modifiedValue = item => {
	EventBus.$emit('openModifiedValue', item)
}
const updateScrollStatus = bool => {
	isScrollRef.value = bool ?? (globalData.graphicsParameters[props.id].data.isScroll || false)
}

const updateHeadStatus = (bool: boolean) => {
	baseData.headStatusVisible = bool
	setRightHeadText()
}
const setRightHeadText = () => {
	//指定的开关的属性
	const item = list.value.find(v => v.valueType === 'BOOLEAN')
	if (!item) return (deviceStatusStr.value = '暂无数据')
	deviceStatusStr.value = formatterObj[item.value]
}

const updateDialogStyle = (value = '') => {
	dialogStyle.value = value
}
const baseDataInit = () => {
	baseData.dialogTitle = params.dialogOption.dialogTitle
	baseData.isConstantDialog = params.dialogOption.isConstantDialog
	baseData.dataTypeList = params.data.dataTypeList
	updateDialogStyle(params.dialogOption.dialogStyle)
	updateHeadStatus(params.dialogOption.headStatusVisible)
}

const init = () => {
	if (params.dialogOption.isDialogVisible && !params.dialogOption?.isConstantDialog) {
		getData()
	}
	domGraphic.show()
	// 弹窗打开并且弹窗常驻,才会自动打开。否则就是隐藏状态
	// if (
	// 	params.dialogOption?.isDialogVisible &&
	// 	params.dialogOption?.isConstantDialog &&
	// 	isSameDataType('REALTIME_DATA')
	// ) {
	// 	domGraphic.show()
	// 	getData()
	// } else {
	// 	domGraphic.hide()
	// }
}

const isConstantDialog = () => {
	return params.dialogOption?.isConstantDialog
}

// 是否启用弹窗
/* const isDialogVisibleChange = () => {
	if (params.dialogOption?.isDialogVisible) {
		baseData.isConstantDialog = params.dialogOption.isConstantDialog
		domGraphic.show()
	} else {
		domGraphic.hide()
	}
} */
// 是否常驻弹窗
/* const isConstantDialogChange = () => {
	baseData.isConstantDialog = params.dialogOption.isConstantDialog
	if (params.dialogOption?.isDialogVisible) {
		domGraphic.show()
	}
} */

// 标题修改
/* const dialogTitleChange = () => {
	baseData.dialogTitle = params.dialogOption.dialogTitle
} */

// 经纬度改变
const positionChange = ({ x, y, z }) => {
	domGraphic.setPosition([x, y, z])
}

// 点击显示方法
const show = async () => {
	if (params.dialogOption?.isDialogVisible) {
		baseData.isConstantDialog = params.dialogOption.isConstantDialog
		// const loadingInstance = ElLoading.service({
		//   fullscreen: false,
		//   text: '获取数据中',
		//   background: 'transparent',
		// })
		await getData()
		// loadingInstance.close()
		domGraphic.show()
		// if (params.data.autoUpdate) loop()
	}
}

/* const receiveModified = key => {
	if (['autoUpdate', 'autoRequest'].includes(key)) loop()
} */

const closeDialog = () => {
	if (isConstantDialog()) {
		store.dialogList.delete(params.id)
	} else {
		store.clickedDialogId = ''
	}
}

const removeDialog = () => {
	if (domGraphic) {
		domGraphic.remove()
		domGraphic = null
	}
	globalData.popups[params.id] = null
}

onMounted(() => {
	baseDataInit()
	updateScrollStatus(true)
	const { position } = params.dialogOption
	domGraphic = new createGraphicDom({
		position: [position?.x, position?.y, position?.z],
		dom: popupDomRef.value,
	})
	globalData.popups[params.id] = {
		domGraphic: domGraphic,
		show: show,
		close(e) {
			console.warn(e)
		},
		baseData: baseData,
		// isDialogVisibleChange: isDialogVisibleChange,
		// isConstantDialogChange: isConstantDialogChange,
		// dialogTitleChange,

		positionChange,
		updateData,
		updateDialogStyle,
		updateHeadStatus,
		updateScrollStatus,
		getHistoryDataFn,
		// receiveModified: receiveModified,
	}
	init()
})

onBeforeUnmount(removeDialog)
</script>

<style lang="scss" scoped>
.gl_popup_template {
	width: 260px;
	min-height: 120px;
	box-sizing: border-box;
	background: rgba(63, 72, 84, 0.9);
	box-shadow: 0 3px 14px rgba(0, 0, 0, 0.4);
	border-radius: 4px;
	padding: 5px 10px 15px;
	color: #fff;
	.gl_popup_template_title {
		width: 100%;
		line-height: 30px;
		font-size: 16px;
		border-bottom: 1px #fff solid;
		position: relative;
		.gl_close {
			color: #fff;
			position: absolute;
			right: 0px;
			top: 5px;
			font-size: 16px;
			z-index: 999;
			cursor: pointer;
		}
	}
	.gl_popup_template_content {
		.item {
			display: flex;
			font-size: 13px;
			margin-top: 5px;
			.label {
				min-width: 50px;
				max-width: 100px;
				padding-right: 5px;
			}
		}

		:deep(.el-table) {
			border-color: red;
			th,
			td {
				background: rgba(63, 72, 84, 0.9);
				color: #fff;
				border-color: rgba(255, 255, 255, 0.5);
			}
			tr:hover > td.el-table__cell {
				background: rgba(0, 0, 0, 0.7);
			}
		}
	}
	.hlsVideo {
		margin-top: 5px;
	}
}
.table-wrap {
	height: 270px;
	overflow: auto;
}
.pagination-wrap {
	background: #fff;
	display: flex;
	height: 40px;
	justify-content: center;
}
.hc-marker-wrap {
	:deep(.el-table) {
		-webkit-backdrop-filter: blur(8px);
		backdrop-filter: blur(20px);
		box-shadow: 0 0 30px 10px rgba(0, 0, 0, 0.3);
		background: transparent;
		tr {
			background: transparent;
		}
		tr:hover > td.el-table__cell {
			background: #0558a794;
		}
		tbody tr:nth-child(odd) {
			background-color: rgba(255, 255, 255, 0.1);
		}
		.el-table__cell {
			background: transparent;
			color: #fff;
		}
	}
	.pagination-wrap {
		background: transparent;
		:deep(.el-pagination) {
			button,
			li {
				background: transparent;
				&:not(.is-active) {
					color: #fff;
				}
			}
		}
	}
}
.bottom {
	display: flex;
	color: rgb(0 192 250);
	justify-content: center;
	align-items: center;
	cursor: pointer;
	i {
		font-size: 1rem;
		margin-left: 0.5rem;
	}
}
</style>

warning-dialog

<script setup lang="ts">
import { getWarningData } from '@/api/request.js'
import temp2 from './temp2.vue'
import { LOOP_TIME } from '@/constants'
const graphicsParametersList = ref([])
let timeout: ReturnType<typeof setTimeout> | null = null
const showWarningDialog = ref(false)
window.showWarningDialog = showWarningDialog
const getData = async () => {
	const list = Object.values(globalData.graphicsParameters)
		.filter(v => v.data?.dataTypeList?.includes('ALARM_DATA'))
		.map(v => v.data.regCodeIdentifier)
		.filter(v => v)
	if (!list.length) {
		graphicsParametersList.value = []
		return (timeout = setTimeout(getData, LOOP_TIME * 1000))
	}
	const str = list.join('::')
	const res = await getWarningData(1, 1, str)
	//FIXME:
	const data = res.data.data.result.map(v => {
		v.deviceIdentifier = 'KELAN03_INS1'
		v.projectIdentifier = 'KELAN03'
		return v
	})
	graphicsParametersList.value = data.map(item => {
		const params = Object.values(globalData.graphicsParameters).filter(
			v =>
				v.data?.identifier === item.deviceIdentifier && v.data?.dataTypeList.includes('ALARM_DATA')
		)
		params.forEach(v => {
			v.data.warningData = item
		})
		return params
	})
	timeout = setTimeout(getData, LOOP_TIME * 1000)
}

const closeDialog = (index1, index2) => {
	// graphicsParametersList.value
}

onMounted(() => {
	getData()
})
onBeforeUnmount(() => {
	if (timeout) {
		clearTimeout(timeout)
		timeout = null
	}
})
</script>
<template>
	<div class="warning-dialog">
		<template v-for="(warningList, k) in graphicsParametersList" :key="k">
			<div
				v-for="(graphicsParametersItem, kk) in warningList"
				:key="kk"
				:id="graphicsParametersItem.id"
				:warningData="graphicsParametersItem"
			>
				<temp2
					v-if="showWarningDialog"
					:graphicsParametersItem="graphicsParametersItem"
					@closeDialog="closeDialog(k, kk)"
				></temp2>
			</div>
		</template>
	</div>
</template>
<style scoped lang="scss">
.warning-dialog {
	// :deep(.el-dialog) {
	//   background-color: rgba(32, 27, 21, 0.6);
	//   .el-dialog__title {
	//     font-weight: bold;
	//     color: #fff;
	//     font-family: Source Han Sans CN, Source Han Sans CN;
	//   }
	// }
}
</style>

ws.js

import IM from 'webtools-instant-messaging'
// 前后端服务部署到一起可使用window.location.host
const host = '10.44.218.94'
// 可根据实际情况获取token
const token = huichuanToken
// 可根据实际情况生成clientId
const clientId = IM.getSign(`${window.location.origin}${window.location.pathname}${token}`)

export default function (store) {
	const im = new IM({
		url: `ws://${host}/api/instant-messaging/ws?token=bearer%20${token}&clientId=${clientId}`,
		messageListener: promise => {
			if (promise instanceof Promise) {
				promise
					.then(res => {
						if (res.messageType === 'REALTIME_DATA_REPORT') {
							window.bindClass.handleRealData(res.data)
							for (const id of store.dialogList) {
								// if (id === 'glModel_178c8daf-47ac-9f86-39b5-60c6def2103a') {
								if (
									window.globalData.graphicsParameters[id].data.identifier ===
									res.data[0].deviceIdentifier
								) {
									window.globalData.popups[id].updateData(res.data)
								}
							}
						}
					})
					.catch(err => {
						// ...异常处理
					})
			}
		},
	})
	window.im = im
}

