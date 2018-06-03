<template>
	<div class="container">
		<!-- <view class='btn-group'>
			<button size='mini' @click="startConnect()">开始搜索</button>
			<button size='mini' @click="stopBluetoothDevicesDiscovery()">停止搜索</button>
		</view> -->
		<!-- <view class='list' v-if="devices.length > 0">
			<card 
				v-for="(device, index) in devices" 
				:key="device.deviceId" 
				:device="device" 
				:index="index" 
				:isConnectingDevice="isConnectingDevice" 
				@startConnectDevices="startConnectDevices">
			</card>
		</view> -->
		<view></view>
	</div>
</template>

<script>
import card from '@/components/card'
import Bluetooth from '../../utils/ble'

export default {
	data () {
		return {
			userInfo: {},
			devices: [],
			isConnectingDevice: '',
			serviceId: ''
		}
	},
	created () {
		// 调用应用实例的方法获取全局数据
		// this.startConnect()
		const bluebooth = new Bluetooth({   // configOptions 参考下方的API
			debug: true,
			keepAlive: true,    // 保持持续链接状态
			// 必须配置 `connectOptions` 中的 `deviceName` 和 `services` 以匹配你想匹配的蓝牙设备
			connectOptions: {
				interval: 0,
				services: [
					// "00006000-0000-1000-8000-00805F9B34FB",
					// "00005752-0000-1000-8000-00805F9B34FB",
					// "0000422D-0000-1000-8000-00805F9B34FB",
					// "0000454C-0000-1000-8000-00805F9B34FB"
					// "6E400001-b5A3-F393-E0A9-E50E24DCCA9E"
				], // your device services array
				allowDuplicatesKey: false,
				// deviceName: 'TD_7B74', // device name
				deviceName: '', // device name
				// characteristicId: ''
			},
			onConnect: function () {
				// 如果 keepAlive 为`true`的话，需要自己手动在 sendData 成功后执行 `return this.trigger('success', true)` 以触发 `finish` 状态以进入关闭蓝牙连接和蓝牙适配器操作
				this.sendData('01').then(res => this.sendData('02')).then(res => this.sendData('03')).then(res => this.trigger('success'))
			},
			onNotify: function (value) {
				// 收到蓝牙设备传过来的值
				console.log(value)
			}
		})

		bluebooth.start();
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
					setTimeout(() => {
						wx.hideToast()
					}, 2000)
				}
			})
			wx.onBluetoothAdapterStateChange(res => {
				if (res.available) {
					this.getBluetoothAdapterState()
				}
			})
		},
		// 获取本机蓝牙适配器状态
		getBluetoothAdapterState() {
			wx.getBluetoothAdapterState({
				success: res => {
					if (!res.available) {
						wx.showToast({
							title: '设备无法开启蓝牙连接',
							duration: 2000
						})
						setTimeout(() => {
							wx.hideToast()
						}, 2000)
					} else {
						if (!res.discovering) {
							this.startBluetoothdeviceDiscovery()
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
				// services: ['6e400001-b5a3-f393-e0a9-e50e24dcca9e'],
				allowDuplicatesKey: false,
				success: res => {
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
		// 监听寻找到新设备的事件
		onBluetoothDeviceFound() {
			console.log('onBluetoothDeviceFound')
			wx.onBluetoothDeviceFound(res => {
				console.log('发现新的设备', res['devices'])
				this.devices = Array.from(new Set(this.devices.concat(res['devices'])))
				if (this.devices.length > 0) {
					wx.hideLoading()
				}
			})
		},
		// 开始连接要配对的设备
		startConnectDevices(deviceId) {
			this.stopBluetoothDevicesDiscovery()
			this.isConnectingDevice = deviceId
			wx.createBLEConnection({
				deviceId: deviceId,
				success: res => {
					console.log('开始连接要配对的设备', res)
					if (res.errCode == 0) {
						setTimeout(() => {
							this.isConnectingDevice = ''
							this.getService(deviceId)
						}, 3000)
					}
				},
				fail: err => {
					this.isConnectingDevice = ''
					console.log('配对失败：', err)
					wx.showToast({
						title: '配对失败',
						duration: 2000
					})
					setTimeout(function () {
						wx.hideToast()
					}, 2000)
				},
				complete: function () {
					console.log('complete connect devices')
				}
			})
		},
		// 连接成功后根据deiviceId获取设备的所有服务
		getService() {
			let deviceId = 'F2:13:AF:B8:7B:74'
			// 监听蓝牙连接(重连)
			// wx.onBLEConnectionStateChange(res => {
			// 	this.isConnectingDevice = ''
			// 	console.log(`device ${res.deviceId} state has changed, connected: ${res.connected}`)
			// 	wx.showToast({
			// 		title: '已断开',
			// 		duration: 2000
			// 	})
			// 	setTimeout(() => {
			// 		wx.hideToast()
			// 	}, 2000)
			// });
			// 获取蓝牙设备service值
			wx.getBLEDeviceServices({
				deviceId: deviceId,
				success: res => {
					wx.showToast({
						title: '找到指定设备!',
						duration: 2000
					})
					setTimeout(() => {
						wx.hideToast()
					}, 2000)
					console.log('获取蓝牙设备service值', res)
					this.getCharacter(deviceId, res.services)
				},
				fail: res => {
					wx.showToast({
						title: '找不到指定设备!',
						duration: 1000
					})
					setTimeout(() => {
						wx.hideToast()
					}, 1000)
				},
				complete: res => {
					if (res.errMsg != 0) {
						this.getService()
					}
					console.log(res)
				}
			})
		},
		// 读取服务的特征值
		getCharacter(deviceId, services) {
			this.serviceId = services[0].uuid
			wx.getBLEDeviceCharacteristics({
				deviceId: deviceId,
				serviceId: this.serviceId,
				success: res => {
					console.log('获取蓝牙设备特征值', res)
					let characteristicIdRead = ''
					if (res['characteristics'][0].properties.read) {
						characteristicIdRead = res['characteristics'][0].uuid
					}
					this.readBLECharacteristicValue(deviceId, this.serviceId, characteristicIdRead)
					// this.writeBLECharacteristicValue(deviceId, this.serviceId, this.characterId_write)
					// this.openNotifyService(deviceId, this.serviceId, this.characterId_read)
				},
				fail: err => {
					console.log(err)
				},
				complete: () => {
					console.log('complete')
				}
			})
		},
		// 
		readBLECharacteristicValue(deviceId, serviceId, characteristicId) {
			wx.readBLECharacteristicValue({
				deviceId: deviceId, 
				serviceId: serviceId, 
				characteristicId: characteristicId,
				success: res => {
					console.log('readBLECharacteristicValue:', res)
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

<style lang="stylus" scoped>
.btn-group
	display flex
	width 100%
	button
		width 120px
.list
	width 80%
	margin 20px auto
	border 1px solid #ddd
	padding 0 10px
	font-size 14px
	background-color #f6f6f6
</style>
