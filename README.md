//获取⼯程信息
export const getProgramMsg = (identifier = 'KELAN') =>
	request.get(
		`/huichuan/api/model/instances?pagination=0&objectType=IOT_PROJECT&$filter=identifier eq ${identifier}`
	)

//获取⼯程信息
export const getProgramMsg1 = property =>
	request.get(
		`/huichuan/api/model/instances/${property}?accessMethod=foreign_key&relationProperties=name,identifier,pathName`
	)

//
export const getProgramMsg2 = property =>
	request.get(
		`/huichuan/api/model/instances?objectType=IOT_MODEL&$filter=pathName leftlike 
${property}&pagination=0&$orderBy=createTime desc`
	)
//
export const getProgramMsg3 = data =>
	request.post(`/huichuan/api/thing-model/instances/models`, data)
