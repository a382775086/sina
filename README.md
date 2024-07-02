import IM from 'webtools-instant-messaging'

// 前后端服务部署到一起可使用window.location.host
const host = '10.44.218.94'
// 可根据实际情况获取token
const token = 'YWUxM2U4MjItZjQxZi00NjcyLWE3MTktN2U5MGQzMGI3MTZk'
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
								window.globalData.popups[id].updateData(res.data)
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
