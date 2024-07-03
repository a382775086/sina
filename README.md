const huichuanToken = 'ZDM0ZWQ1MDctMjkxYi00NmEyLWJhNzMtODI3MTZlNDlmMGE4'

onMounted(async () => {
	const projectIdentifier = 'KELAN02'
	const res = await getProgramMsg(projectIdentifier) //地址栏传入的 projectIdentifier
	const property = res.data.data[0].property
	const res1 = await getProgramMsg1(property)
	const pathName = res1.data.data['model.pathName']
	const res2 = await getProgramMsg2(pathName)
	const res3 = await getProgramMsg3({
		filter: `status ne 2,pathName leftlike ${pathName}/`,
		models: res2.data.data.map(item => item.identifier).join(','),
		orderBy: 'createTime desc',
		projectIdentifier: projectIdentifier, //地址栏传入的 projectIdentifier
	})
	window.identifierList = res3.data.data
})

//home 
<!-- /**
 * @Date 2023-03-15 10:16:03 施磊鉴
 * @introduction 地图组件
 * @param {参数类型} 参数 参数说明
 * @return {返回类型说明}
 */ -->
<template>
	<div class="map">
		<div id="map"></div>
		<div id="map2"></div>
	</div>
	<rightClickMenu />
</template>
<script setup>
import { nextTick, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { getMapById, getTemplateById } from '@/api/request.js'
import { setSaveJson } from '@/utils/jsonSetting'
import getElementList from '@/utils/getElementList.ts'
import useStore from '@/store'
import { initHcFunction, BindEventClass } from '@/utils/mapUtils'
import { addServerData, addCustomLayer } from '@/utils/methods/sceneLayer.js'
import graphic from '@/utils/graphic.js'
import rightClickMenu from '@/components/rightClickMenu/index.vue'
import bindEvent from './eventTrigger'
import EventBus from '@/utils/bus'
const emits = defineEmits(['getDataCallBack'])
const store = useStore()
const router = useRouter()
const initScene = () => {
	// window.globalData.scene.methods.skyBoxInit()
	// window.globalData.scene.methods.groundInit()
	// window.globalData.scene.methods.cameraInit()
	// window.globalData.scene.methods.globeInit()
	// map.clock.currentTime = CooGL.JulianDate.fromDate(
	//   new Date('2022-10-27T12:00:00')
	// )
	globalData.scene.methods.init()
}

const initLayer = () => {
	globalData.layer.methods.init()
}

const initServerData = () => {
	if (globalData.serverDataMap.length) {
		globalData.serverDataMap.forEach(item => {
			addServerData(Object.assign(item, { layerShow: true }), false)
		})
	}
}
const initCustomData = () => {
	if (globalData.customLayer.length) {
		globalData.customLayer.forEach(item => {
			addCustomLayer(Object.assign(item, { layerShow: true }), false)
		})
	}
}
const initGraphics = () => {
	Object.keys(globalData.graphicsParameters).forEach(id => {
		globalData.graphicMethods.addByGraphicParameters(globalData.graphicsParameters[id])
		const paramObj = globalData.graphicsParameters[id]
		if (paramObj.show === false) {
			setTimeout(() => {
				globalData.graphics[id].show = false
			}, 3000)
		}
	})
}

const initCooGis = async content => {
	// cooserver配置
	// const isDev = import.meta.env.DEV
	CooGL.CooServer.address = cooServerAddress
	CooGL.CooServer.defaultAccessToken = globalData.token
	const id = router.currentRoute.value.query?.id
	const tmp = router.currentRoute.value.query?.tmp
	let res
	if (!content) {
		if (tmp === 'true') {
			res = await getTemplateById(id)
		} else {
			res = await getMapById(id)
		}
	} else {
		let mks = Object.keys(globalData.graphics)
		if (mks.length) {
			mks.map(k => {
				globalData.graphicMethods.removeGraphicByid(k)
			})
		}
		res = { data: { data: { content } } }
	}

	// res.data.data.content = res.data.data.content.replace(
	//   /http:\/\/192\.168\.10\.202:9091/g,
	//   ''
	// )
	setSaveJson(res.data.data.content)
	emits('getDataCallBack')
	getElementList()
	// HTMLCanvasElement.prototype.toDataURL = () => {}
	// setTimeout(() => {
	//   HTMLCanvasElement.prototype.toDataURL = toDataURLFn
	// }, 10000)
	const mapConfig = {
		bottom: true,
		// initializeAnimation: false,
		// menu: false,
		// timeline: true,
		// environment: './env.hdr',
		// defaultView: globalData.scene.methods.getCameraView(),
	}
	if (!Reflect.has(window.map || {}, 'layers')) {
		const map = new CooGL.EditorViewer('map', {
			defaultView: globalData.scene.methods.getCameraView(),
		})
		window.map = map
		// map.camera.maximumZoomDistance = 10000
		map.camera.minimumZoomDistance = 0
		let canMove = false
		//吸附事件初始化
		document.addEventListener('keydown', function (ev) {
			const keyCode = ev.key
			if (keyCode === 'g' && CooGL.defined(map.link.editor)) {
				canMove = true
			}
		})
		document.addEventListener('dblclick', ev => {
			map.link.move()
			canMove = false
			// map.linkEnabled = false
			map.link.editor = undefined
		})
		map.event.mousemove(function (ev) {
			if (canMove && CooGL.defined(map.link.editor)) {
				const position = ev.point
				map.link.editor.position = position
				map.link.update()
			}
		})
		map.event.pick(ev => {
			const source = ev.layer
			if (source instanceof CooGL.EditorModel) {
				map.link.editor = source
				if (canMove) canMove = false
			}
		})
	}

	observeModelHeight()
	//泛光关闭
	// map.effects.bloomEnabled = false
	// globalData.vueMount()
	globalData.graphicConfig.editor.enabled = true
	initScene()
	// initLayer()
	initGraphics()
	bindEvent(store, EventBus)
	addBaseMap()
	//TODO:临时代码
	window.customPipeMap = {}
	initHcFunction()
	addCustomCode()
	const bindClass = new BindEventClass()
	window.bindClass = bindClass

	//加载基础图层服务数据
	initServerData()
	//加载自定义图层
	initCustomData()
}
window.initCooGis = initCooGis

const addCustomCode = () => {
	customCode()
}

window.closeDialog = () => {
	window.tempDialog?.destroy()
}

//添加默认底图
const addBaseMap = () => {
	setTimeout(() => {
		const { x, y } = map.getView().destination
		map
			.addModel(
				new CooGL.GltfEditorModel({
					// position: {
					// 	x: 0,
					// 	y: 0,
					// 	z: 0,
					// },
					position: {
						x,
						y,
						z: -0.01,
					},
					url: './baseMap.glb',
					color: 'rgba(0,0,0,0.01)',
					scale: new CooGL.Vector3(100, 100, 100),
				})
			)
			.then(res => {
				ElMessage.success('地图初始化完成')
			})
	}, 2000)
}
//添加模型高度监听，不允许低于height=0

const observeModelHeight = () => {
	map.editor.propertyUpdateEvent.addEventListener((p1, p2) => {
		if (p1.id.startsWith('glWall_')) {
			EventBus.$emit('updataWallForm', p1)
		} else if (p1.id.startsWith('glPolygon_')) {
			EventBus.$emit('updataPolygonForm', p1)
		} else if (p1.id.startsWith('glPolyline_')) {
			EventBus.$emit('updatapolygonLineForm', p1)
		} else {
			if (p1.position.z < 0) {
				const position = CooGL.Vector3.clone(p1.position)
				position.z = 0
				map.editor.position = position
			}
		}
	})
}

onMounted(() => {
	nextTick(() => {
		initCooGis()
	})
})
</script>
<style lang="scss">
.map {
	position: absolute;
	left: 0;
	right: 0;
	bottom: 0;
	top: 0;
	height: 100%;
	overflow: hidden;
	width: 100%;
	display: flex;
}
.compo-status-bar-logo,
.compo-status-bar-mouseover {
	width: 0 !important;
	height: 0 !important;
	overflow: hidden;
}
#map {
	width: 100%;
	height: 100%;
}
.dialog-attr-wrap {
	position: relative;
	background: rgba(0, 0, 0, 0.6);
	color: #fff;
	padding: 12px;
	min-width: 300px;
	.dialog-header {
		position: relative;
		height: 30px;
		line-height: 30px;
		.dialog-close {
			position: absolute;
			font-size: 12px;
			right: 0px;
			top: 0px;
			cursor: pointer;
		}
	}
}
</style>

//preview
<!-- /**
 * @Date 2023-03-15 10:16:03 施磊鉴
 * @introduction 地图组件
 * @param {参数类型} 参数 参数说明
 * @return {返回类型说明}
 */ -->
<template>
	<div class="map">
		<div id="map"></div>
		<div id="map2"></div>
	</div>
</template>
<script setup>
import { nextTick, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { getMapById, getModels, getTemplateById } from '@/api/request.js'
import { setSaveJson } from '@/utils/jsonSetting'
import graphic from '@/utils/graphic.js'
import bindEvent from './eventTrigger'
import useStore from '@/store'
import { initHcFunction, BindEventClass } from '@/utils/mapUtils'
import { addServerData, addCustomLayer } from '@/utils/methods/sceneLayer.js'
import EventBus from '@/utils/bus'

const router = useRouter()
const store = useStore()
const initScene = () => {
	// window.globalData.scene.methods.skyBoxInit()
	// window.globalData.scene.methods.groundInit()
	// window.globalData.scene.methods.cameraInit()
	// window.globalData.scene.methods.globeInit()
	// map.clock.currentTime = CooGL.JulianDate.fromDate(
	//   new Date('2022-10-27T12:00:30')
	// )
	globalData.scene.methods.init()
}

const initLayer = () => {
	globalData.layer.methods.init()
}

const initServerData = () => {
	if (globalData.serverDataMap.length) {
		globalData.serverDataMap.forEach(item => {
			addServerData(Object.assign(item, { layerShow: true }), false)
		})
	}
}

const initCustomData = () => {
	if (globalData.customLayer.length) {
		globalData.customLayer.forEach(item => {
			addCustomLayer(Object.assign(item, { layerShow: true }), false)
		})
	}
}

const initGraphics = () => {
	Object.keys(globalData.graphicsParameters).forEach(id => {
		globalData.graphicMethods.addByGraphicParameters(globalData.graphicsParameters[id])
		const paramObj = globalData.graphicsParameters[id]
		if (paramObj.show === false) {
			setTimeout(() => {
				globalData.graphics[id].show = false
			}, 3000)
		}
	})
}

const initCooGis = async () => {
	// cooserver配置
	// const isDev = import.meta.env.DEV
	CooGL.CooServer.address = cooServerAddress
	CooGL.CooServer.defaultAccessToken = globalData.token
	const id = router.currentRoute.value.query?.id
	const tmp = router.currentRoute.value.query?.tmp
	let res
	if (tmp === 'true') {
		res = await getTemplateById(id)
	} else {
		res = await getMapById(id)
	}
	setSaveJson(res.data.data.content)

	// HTMLCanvasElement.prototype.toDataURL = () => {}
	// setTimeout(() => {
	//   HTMLCanvasElement.prototype.toDataURL = toDataURLFn
	// }, 10000)
	const mapConfig = {
		bottom: false,
		initializeAnimation: false,
		menu: false,
		defaultView: globalData.scene.methods.getCameraView(),
	}
	const map = new CooGL.EditorViewer('map', mapConfig)
	window.map = map
	// globalData.vueMount()
	globalData.graphicConfig.editor.enabled = false
	initScene()
	// initLayer()
	initGraphics()
	window.customPipeMap = {}
	bindEvent(store, EventBus)
	initHcFunction()
	// map.camera.controller.maximumZoomDistance = 30
	map.camera.controller.minimumZoomDistance = 5
	addCustomCode()
	const bindClass = new BindEventClass()
	window.bindClass = bindClass

	//加载基础图层服务数据
	initServerData()
	//加载自定义图层
	initCustomData()
}
const addCustomCode = () => {
	customCode()
}
const getData = () => {
	getModels().then(res => {
		window.groupObj = {}
		res.data.data.forEach(list => {
			for (let item of list.iconFileList) {
				window.groupObj[item.fileUrl] = item
			}
		})
	})
}

onMounted(() => {
	getData()
	nextTick(() => {
		initCooGis()
	})
})
</script>
<style lang="scss">
.map {
	position: absolute;
	left: 0;
	right: 0;
	bottom: 0;
	top: 0;
	height: 100%;
	overflow: hidden;
	width: 100%;
	display: flex;
}
#map {
	width: 100%;
	height: 100%;
}
</style>

//popup/index
<template>
	<div class="popup">
		<div v-if="mapCompleted">
			<Temp1 v-for="id in store.dialogList" :key="id" :id="id" />
			<!-- 独立单一点击数据弹窗 -->
			<Temp1 v-if="store.clickedDialogId" :id="store.clickedDialogId" />
		</div>
	</div>
	<!-- 点击显示的数据 -->
	<WarnDialog />
</template>

<script setup>
import useStore from '@/store'
import Temp1 from './temp1.vue'
import { getIotData, setIotData, getIotDetail } from '@/api/request.js'
import EventBus from '@/utils/bus'
import { LOOP_TIME } from '@/constants'

const store = useStore()
//轮询时间
let timeout = null
// 加载完成
const mapCompleted = ref(false)

globalData.popups = {}

// 左键地图监听事件，用来打开冒泡弹窗
let leftClick

const loop2 = async () => {
	const requestList = []
	for (const id of store.dialogList) {
		requestList.push(
			...globalData.graphicsParameters[id].data.keyMap.map(v => v.key).filter(v => v)
		)
	}
	if (requestList.length) {
		const arr = [...new Set(requestList)]
		if (arr.length) {
			im.send(
				{
					messageType: 'REALTIME_DATA_REQ',
					data: {
						items: arr,
						type: 1,
						timeout: 30, // 请求生效，会持续上报30分钟数据
						reportingFrequency: '0/2 * * * * ?',
					},
				},
				'33'
			)
		}
	}

	timeout = setTimeout(loop2, LOOP_TIME * 1000)
}
const loop = async () => {
	const requestList = []
	for (const id of store.dialogList) {
		requestList.push(...globalData.graphicsParameters[id].data.keyMap.map(v => v.key))
	}

	if (!requestList.length) return
	const arr = [...new Set(requestList)]
	const res = await getIotData({ propertyIdentifiers: arr })
	//FIXME:mock
	if (window.tempRealData) {
		// const num = Math.ceil(Math.random() * 5)
		res.data.data[0].dataValue = window.tempRealData
	}
	const data = res?.data?.data
	bindClass.handleRealData(data)
	for (const id of store.dialogList) {
		globalData.popups[id].updateData(data)
	}
	timeout = setTimeout(loop, LOOP_TIME * 1000)
}

const closeLoop = () => {
	if (timeout) {
		clearTimeout(timeout)
		timeout = null
	}
}

const addDialogStatus = () => {
	for (let i in globalData.graphicsParameters) {
		const { data, dialogOption, id } = globalData.graphicsParameters[i]
		if (
			dialogOption?.isConstantDialog &&
			dialogOption?.isDialogVisible &&
			data.dataTypeList?.includes('REALTIME_DATA')
		) {
			store.dialogList.add(id)
		}
	}
}

globalData.mapOnMounted(() => {
	mapCompleted.value = true
	addDialogStatus()
	setTimeout(loop2, LOOP_TIME * 1000)
})

const refreshDialogStatus = () => {
	store.dialogList.clear()
	nextTick(addDialogStatus)
}
EventBus.$on('refreshDialogStatus', refreshDialogStatus)
const addByType = parameters => {
	const id = parameters?.id
	if (!id) return
	const type = id.split('_')[0]
	switch (type) {
		case 'glText':
			break
		case 'glModel':
			store.dialogList.add(id)
			break
	}
}
// 添加
const graphicAddFn = data => {
	// lists.value[data.type].push(data.params)
	addByType(data.params)
}
// 删除
const graphicDeleteFn = data => {
	delete globalData.popups[data.id]
	popupList.value = popupList.value.filter(v => {
		return v.id !== data.id
	})
}

onBeforeUnmount(() => {
	if (timeout) {
		clearTimeout(timeout)
		timeout = null
	}
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

//modifiedvaluepop
<template>
	<el-dialog ref="dom" v-model="dialogShow" title="修改属性" width="20%">
		<div class="align-center">
			<span style="width: 100px">新值{{ inputValue }}</span>
			<el-switch v-if="option" v-model="inputValue" @change="setData" />
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
	console.warn()
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


//temp1
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
					<div style="width: 8rem" class="hc-marker-item-value">
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
		window.tempRealData = inputValue.value
		emits('close')
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
			valueType: obj.dataType,
			isScroll: v.isScroll || false,
			dataIdentifier: v.identifier,
		}
	})
	const tempValueList = list.value.map(v => +v.value)
	//极简模式，需要手动更新设备状态
	if (dialogStyle.value === 'custom2') {
		const item = list.value.find(v => v.value === 'true' || v.value === 'false')
		if (item) {
			deviceStatusStr.value = formatterObj[window.testStatus ?? item.value]
		}
	}
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
	deviceStatusStr.value = formatterObj[bool + '']
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
	updateScrollStatus()
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

