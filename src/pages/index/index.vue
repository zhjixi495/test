<template>
	<div class="container">
		<view class='btn-group'>
			<button size='mini' @click="startConnect()">开始搜索</button>
			<button size='mini' @click="stopBluetoothDevicesDiscovery()">停止搜索</button>
		</view>
		<view class='list' v-if="devices.length > 0">
			<view class='li' v-for="(device, index) in devices">
				<text class='txt'>设备{{index + 1}}：{{device.name}}</text>
				<text class='connect' @click='startConnectDevices(device.deviceId)'>连接</text>
			</view>
		</view>
	</div>
</template>

<script>
import card from '@/components/card'

export default {
	data () {
		return {
			userInfo: {},
			devices: []
		}
	},
	created () {
		// 调用应用实例的方法获取全局数据
	},
	methods: {
		// 开始
		startConnect() {
			wx.showLoading({
				title: '开启蓝牙适配'
			})
			// 开启蓝牙适配器
			wx.openBluetoothAdapter({
				success: res => {
					console.log('初始化蓝牙适配器')
					console.log(res)
					this.getBluetoothAdapterState()
				},
				fail: err => {
					wx.showToast({
						title: '蓝牙初始化失败',
						duration: 2000
					})
					setTimeout(function () {
						wx.hideToast()
					}, 2000)
				}
			})
			wx.onBluetoothAdapterStateChange(res => {
				console.log(res)
				let available = res.available
				if (available) {
					this.getBluetoothAdapterState()
				}
			})
		},
		// 获取本机蓝牙适配器状态
		getBluetoothAdapterState() {
			wx.getBluetoothAdapterState({
				success: res => {
					let available = res.available
					let discovering = res.discovering
					if (!available) {
						wx.showToast({
							title: '设备无法开启蓝牙连接',
							duration: 2000
						})
						setTimeout(function () {
							wx.hideToast()
						}, 2000)
					} else {
						if (!discovering) {
							this.startBluetoothdeviceDiscovery()
							this.getConnectedBluetoothDevices()
						}
					}
				}
			})
		},
		// 开始搜索蓝牙设备
		startBluetoothdeviceDiscovery () {
			wx.showLoading({
				title: '蓝牙搜索'
			})
			wx.startBluetoothDevicesDiscovery({
				services: [],
				allowDuplicatesKey: false,
				success: res => {
					console.log(res)
					if (!res.isDiscovering) {
						this.getBluetoothAdapterState()
					} else {
						this.onBluetoothDeviceFound()
					}
				},
				fail: err => {
					console.log(err)
				}
			})
		},
		// 获取已配对的蓝牙设备
		getConnectedBluetoothDevices() {
			wx.getConnectedBluetoothDevices({
				// services: [that.serviceId],
				services: [],
				success: res => {
					console.log("获取处于连接状态的设备", res['devices'])
					this.devices = Array.from(new Set(this.devices.concat(res['devices'])))
					if (this.devices.length > 0) {
						wx.hideLoading()
					}
					let devices = res['devices'], flag = false, index = 0, conDevList = []
					console.log(devices)
					devices.forEach((value, index, array) => {
						if (value['name'].indexOf('FeiZhi') != -1) {
							// 如果存在包含FeiZhi字段的设备
							flag = true
							index += 1
							conDevList.push(value['deviceId'])
							that.deviceId = value['deviceId']
							return
						}
					})
					if (flag) {
						this.connectDeviceIndex = 0
						this.loopConnect(conDevList)
					} else {
						if (!this.getConnectedTimer) {
							this.getConnectedTimer = setTimeout(() => {
								this.getConnectedBluetoothDevices()
							}, 5000)
						}
					}
				},
				fail: err => {
					if (!this.getConnectedTimer) {
						this.getConnectedTimer = setTimeout(() => {
							this.getConnectedBluetoothDevices()
						}, 5000)
					}
				}
			})
		},
		// 开启蓝牙搜索功能失败
		onBluetoothDeviceFound() {
			console.log('onBluetoothDeviceFound')
			wx.onBluetoothDeviceFound(res => {
				console.log('new device list has founded')
				console.log(res['devices'])
				this.devices = Array.from(new Set(this.devices.concat(res['devices'])))
				if (this.devices.length > 0) {
					wx.hideLoading()
				}
			})
		},
		// 开始连接要配对的设备
		startConnectDevices(deviceId) {
			clearTimeout(this.getConnectedTimer)
			this.getConnectedTimer = null
			// clearTimeout(this.discoveryDevicesTimer)
			this.stopBluetoothDevicesDiscovery()
			this.isConnectting = true
			wx.createBLEConnection({
				deviceId: deviceId,
				success: res => {
					if (res.errCode == 0) {
						setTimeout(() => {
							this.getService(deviceId)
						}, 5000)
					}
				},
				fail: function (err) {
					console.log('连接失败：', err)
				},
				complete: function () {
					console.log('complete connect devices')
					this.isConnectting = false
				}
			})
		},
		// 连接成功后根据deiviceId获取设备的所有服务
		getService(deviceId) {
			// 监听蓝牙连接
			wx.onBLEConnectionStateChange(res => {
				console.log(res)
			});
			// 获取蓝牙设备service值
			wx.getBLEDeviceServices({
				deviceId: deviceId,
				success: res => {
					this.getCharacter(deviceId, res.services)
				}
			})
		},
		// 读取服务的特征值
		getCharacter(deviceId, services) {
			services.forEach((value, index, array) => {
				if (value == this.serviceId) {
					this.serviceId = array[index]
				}
			})
			wx.getBLEDeviceCharacteristics({
				deviceId: deviceId,
				serviceId: this.serviceId,
				success: (res) => {
					this.writeBLECharacteristicValue(deviceId, this.serviceId, this.characterId_write)
					this.openNotifyService(deviceId, this.serviceId, this.characterId_read)
				},
				fail: (err) => {
					console.log(err)
				},
				complete: () => {
					console.log('complete')
				}
			})
		},
		// 停止蓝牙搜索
		stopBluetoothDevicesDiscovery() {
			wx.stopBluetoothDevicesDiscovery({
				success: (res) => {
					console.log(res)
					wx.hideLoading()
				}
			})
		}
	},
	components: {
		card
	}
}
</script>

<style scoped>
.btn-group {
	display: flex;
	width: 100%
}
.btn-group button {
	width: 120px
}
.list {
	width: 80%;
	margin: 20px auto;
	border: 1px solid #ddd;
	padding: 0 10px;
	font-size: 14px;
	background-color: #f6f6f6
}
.list .li {
	display: flex;
	height: 50px;
	line-height: 50px;
	border-bottom: 1px solid #ddd;
}
.list .li:last-child {
	border-bottom: none
}
.list .li .txt {
	flex: 1
}
.list .li .connect {
	flex: 0 0 60px;
	text-align: right;
	color: blue;
}
</style>
