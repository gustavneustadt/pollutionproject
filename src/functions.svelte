<script context="module">
	import * as d3 from "d3"
	export function sumArrayOfObjects(group, template = null) {
		let templateCopy = { ...template}
		var objectBuffer = {}
		var templateBuffer = {}
		if(template) {
			let keys = Object.keys(template)
			let values = Object.values(template)
			let filteredIndexes = values.map((value, i) => {
				if(value instanceof Object) {
					return i
				}
			}).filter(i => i != undefined)
			
			filteredIndexes.forEach(index => {
				templateBuffer[keys[index]] = values[index]
			})
		}
		
		let reduced = group.reduce((acc, curr) => {
			let objectKeys = Object.keys(curr)
			let objectValues = Object.values(curr)
			
			let objectValueTypes = objectValues.map(val => typeof val)
			
			objectKeys.forEach((key, i) => {
				switch(objectValueTypes[i]) {
					case "number":
						if(template && acc[key] == undefined) {
							break;
						}
						if(acc[key] == undefined) {
							acc[key] = objectValues[i]
						} else {
							acc[key] += objectValues[i]
						}
						break;
					case "object":
						if(!template && templateBuffer[key] != undefined) {
							break;
						}
						if(template && templateBuffer[key] == undefined) {
							break;
						}
						if(objectBuffer[key] == undefined) {
							objectBuffer[key] = [objectValues[i]]
						} else {
							objectBuffer[key].push(objectValues[i])
						}
						break;
					default:
						return	
				}
			})
			return acc
		}, templateCopy ?? {})
	
		Object.keys(objectBuffer).forEach(objectKey => {
			let object = {}
			object[objectKey] = sumArrayOfObjects(objectBuffer[objectKey], templateBuffer[objectKey])
			Object.assign(reduced, object)
		})
		
		return reduced
	}
</script>

