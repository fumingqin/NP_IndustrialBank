<template>
	<view>
		
	</view>
</template>

<script>
	export default {
		data() {
			return {
				
			}
		},
		onShow:function(){
			this.aa()
		},
		methods: {
			aa:function(){
				uni.requestPayment({
					provider: 'wxpay',
					timeStamp: '1604906068',
					nonceStr: 'c915c1e7b43c4ef6900d2965d95be421',
					package: 'prepay_id=wx09151428275000d203677f49921fd90000',
					signType: 'RSA',
					paySign: 'rtdZcj0D/4Se6lZZqALZPvFf9ywnnDWY0xK4raP99q2Nh5COw+s5jhZ0qCVkAP94siOUnTzOXraroMqHJLdDaQ6oOG+4nze1r0dKn+9XYGxaOflLmhjWhKePm5Pu00xh4u8DglEOi4JWBbmzhzS3lZzkyC5fuvwH0Fz+XxieP/tp1LNtFx/h50CBCSKIACwbC8ONw6RwQzmmviilRVZEtNlsYR2k3yUGlw2fOzE5osIvS78afpV45UNizkKDkgrzEnjHgFZxWSaPG3uXVtTZSRtfNfqagOLa3E6zi8xiyVB1i69CtQMMeBJf7LVv6HZaKjg6afOJzm9ROEsTGOU8Zw==',
					
					success(res) {
						console.log(res)
						 
					},
					fail(err) {
						console.log(err)
					}
				});
			}
		}
	}
</script>

<style>

</style>
