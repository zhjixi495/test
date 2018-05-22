<template>
	<view class='li'>
		<text class='txt'>设备{{index + 1}}：{{device.name ? device.name : device.deviceId}}</text>
		<text class='connecting' v-if="isConnectingDevice == device.deviceId">正在配对...</text>
		<button size='mini' type="primary" class='connect' @click='startConnectDevices()' v-else>连接</button>
	</view>
</template>

<script>
export default {
	props: {
		device: {
			type: Object
		},
		index: {
			type: Number
		},
		isConnectingDevice: {
			type: String,
			default: ''
		}
	},
	data() {
		return {
			isConnecting: ''
		}
	},
	methods: {
		startConnectDevices() {
			if (this.isConnectingDevice) return
			this.$emit('startConnectDevices', this.device.deviceId)
		}
	}
}
</script>

<style lang="stylus" scoped>
.li
	display flex
	height 50px
	line-height 50px
	border-bottom 1px solid #ddd
	align-items center
	&:last-child
		border-bottom none
	.txt
		flex 1
	.connect
		flex 0 0 60px
		height 30px
		text-align right
	.connecting
		flex 0 0 80px
		text-align right
		color #ccc
</style>
