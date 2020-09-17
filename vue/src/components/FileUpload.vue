<template>
	<div>
		<input type="file" @change="handleFileChange" />
		<el-button @click="handleUpload">上传</el-button>
	</div>
</template>

<script>
const SIZE = 10 * 1024 * 1024 // 切片大小

export default {
	data: () => ({
		container: {
			file: null,
		},
		data: [],
	}),
	methods: {
		request({
			url,
			method = 'post',
			data,
			headers = {},
			// requestList
		}) {
			return new Promise((resolve) => {
				const xhr = new XMLHttpRequest()
				xhr.open(method, url)
				Object.keys(headers).forEach((key) =>
					xhr.setRequestHeader(key, headers[key])
				)
				xhr.send(data)
				xhr.onload = (e) => {
					resolve({
						data: e.target.response,
					})
				}
			}).catch((err) => {
				console.log('catch error')
				console.log(err)
			})
		},
		// 文件input
		handleFileChange(e) {
			const [file] = e.target.files
			if (!file) return
			Object.assign(this.$data, this.$options.data())
			this.container.file = file
		},
		// 生成文件切片
		createFileChunk(file, size = SIZE) {
			const fileChunkList = []
			let cur = 0
			while (cur < file.size) {
				fileChunkList.push({ file: file.slice(cur, cur + size) })
				cur += size
			}
			return fileChunkList
		},
		// 上传切片
		async uploadChunks() {
			const requestList = this.data
				.map(({ chunk, hash }) => {
					const formData = new FormData()
					formData.append('chunk', chunk)
					formData.append('hash', hash)
					formData.append('filename', this.container.file.name)
					return { formData }
				})
				.map(async ({ formData }) =>
					this.request({
						url: 'http://localhost:3000',
						data: formData,
					})
				)
			await Promise.all(requestList) // 并发切片
			// 合并切片
			await this.mergeRequest()
		},
		async mergeRequest() {
			await this.request({
				url: 'http://localhost:3000/merge',
				headers: {
					'content-type': 'application/json',
				},
				data: JSON.stringify({
					size: SIZE,
					filename: this.container.file.name,
				}),
			})
		},
		// 点击上传
		async handleUpload() {
			if (!this.container.file) return
			const fileChunkList = this.createFileChunk(this.container.file)
			console.log('fileChunkList', fileChunkList)
			this.data = fileChunkList.map(({ file }, index) => ({
				chunk: file,
				hash: this.container.file.name + '-' + index, // 文件名 + 数组下标
			}))
			await this.uploadChunks()
		},
	},
}
</script>
