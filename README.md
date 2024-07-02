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
		requestList.push(...globalData.graphicsParameters[id].data.keyMap.map(v => v.key))
	}

	if (!requestList.length) return
	const arr = [...new Set(requestList)]
	if (!arr.length) return
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

globalData.mapOnMounted(() => {
	mapCompleted.value = true
	addDialogStatus()
	setTimeout(loop2, LOOP_TIME * 1000)
})

const addDialogStatus=()=>{
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

const refreshDialogStatus=()=>{
	store.dialogList.clear()
	nextTick(addDialogStatus)
}
EventBus.$on('refreshDialogStatus',refreshDialogStatus)
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
