---
layout: post
title: uniapp开发微信小程序支付功能
tags: 技术
---
uniapp开发微信小程序支付<br />
1、申请接入微信支付<br />

打开微信公众平台申请接入微信支付:https://mp.weixin.qq.com/<br />

2、调用统一下单和微信支付接口<br />
```javascript
uni.login({
	success: (e) => {
		uni.request({
			url: '',
			data: {
				openId: this.$store.state.openId,
				attach: this.attachList.join('&'),
				orderNote: this.stakeName,
				finalCost: this.totalCost
			},
			method: 'POST',
			header: {
				'content-type': 'application/json'
			},
			success: (res) => {
				if (res.statusCode !== 200) {
					uni.showModal({
						content: "支付失败，请重试！",
						showCancel: false
					});
					return;
				}
				if (res.data.code === 0) {
					console.log("得到接口prepay_id", res.data.data);
					let paymentData = res.data.data;
					uni.requestPayment({
						timeStamp: paymentData.timeStamp,
						nonceStr: paymentData.nonceStr,
						package: paymentData.package,
						signType: 'MD5',
						paySign: paymentData.paySign,
						success: (res) => {
							uni.showToast({
								title: "支付成功!"
							});
						},
						fail: (res) => {
							uni.showModal({
								content: "支付失败",
								showCancel: false
							})
						},
						complete: () => {
							this.loading = false;
						}
					})
				} else {
					uni.showModal({
						content: res.data.msg,
						showCancel: false
					})
					this.loading = false;
				}
			},
			fail: (e) => {
				console.log("fail", e);
				this.loading = false;
				uni.showModal({
					content: "支付失败,原因为: " + e.errMsg,
					showCancel: false
				})
			}
		})

	},
	fail: (e) => {
		console.log("fail", e);
		this.loading = false;
		uni.showModal({
			content: "支付失败,原因为: " + e.errMsg,
			showCancel: false
		})
	}
})
```
