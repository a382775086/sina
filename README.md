import { getProgramMsg, getProgramMsg1,getProgramMsg2,getProgramMsg3 } from '@/api/request'
const store = useStore()
import initWs from '@/utils/ws.js'
const locale = ref(zhLocale)
initWs(store)
onMounted(async () => {
	const res = await getProgramMsg()//地址栏传入的 projectIdentifier
	const property = res.data.data[0].property
	const res1 = await getProgramMsg1(property)
	const pathName=res1.data.data['model.pathName']
	const res2 = await getProgramMsg2(pathName)
	const res3= await getProgramMsg3({
		filter: `status ne 2,pathName leftlike ${pathName}/`,
		models: res2.data.data.map((item) => item.identifier).join(","),
		orderBy: 'createTime desc',
		projectIdentifier: 'KELAN',//地址栏传入的 projectIdentifier
	})
	window.identifierList = res3.data.data
})
