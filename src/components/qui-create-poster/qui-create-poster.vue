<template>
	<view class="content" v-if="isShow" @click.stop="close">
		
		<canvas :style="{ width: canvasW + 'px', height: canvasH + 'px' }" canvas-id="my-canvas" class="isCan"></canvas>
		<view v-if="showImg">
			<img class="showImg" @longpress="longpress" :style="{ width: canvasW + 'px', height: canvasH + 'px' }" :src="tempImg"
			 style="display: block;margin: 0 auto;border-radius:6px" />
			 
		</view>
		<!-- #ifdef MP-WEIXIN -->
		<qui-button v-if="showImg" type="primary" size="max" @click="saveImg" :style="{width:canvasW+'px'}" style="margin-top: 15px;">
		  {{ i18n.t('share.savealbum') }}
		</qui-button>
		<!-- #endif -->
		<!-- #ifdef H5 -->
		<view v-if="showtip" class='tipToast'>长按保存图片！</view>
		<!-- #endif -->
	</view>
</template>

<script>
	import forums from '@/mixin/forums';
	export default {
		mixins: [forums],
		props: {
			headerImg: {
				type: String,
				default: ''
			},
			subTitle: {
				type: String,

			},
			username: {
				type: String,
			},
			threadId:{
				type: String,
			}
		},
		data() {
			return {
				logo: '[大文案]',
				tempImg: '',
				canvasW: 270,
				canvasH: 430,
				ctx: null,
				isShow: false,
				showImg:false,
				strList: [],
				row: 0,
				showtip: false,
				imgHeight: 180,
				imgWidth: 270,
				scale: 1,
				x: 0,
				y: 0,
				w: 0,
				h: 0,
				textY:0,//文字区域开始坐标
				lineheight:20,//行高
				linelength:17,//每行长度
				maxline:9,//最大行数
				jurisdiction:false,
				userweixincode:'',
				slitename:'分享自大文案',
				longpressText:'长按识别小程序码查看详情'
				
			}
		},

		methods: {
			// 点击关闭按钮
			close() {
				uni.hideLoading();
				this.isShow = false;
				this.tempImg = "";
			},

			//显示
			showCanvas() {
				this.strList = [];
				this.isShow = true
				this.__init();
				console.log(this.forums)
			},
			//初始化画布
			async __init() {

				uni.showLoading({
					title: '正在生成海报...',
					mask: true
				})
				this.ctx = uni.createCanvasContext('my-canvas', this)
				//设置填充颜色
				this.ctx.setFillStyle('#FFFFFF')
				//设置背景填充区域大小即位置
				this.ctx.fillRect(0, 0, this.canvasW, this.canvasH)
				//画布中显示的文字行数
				let linenum=0;
				// #ifdef H5
				let bottomH=0
				// #endif
				// #ifdef MP-WEIXIN
				let bottomH=30
				// #endif
				if (this.headerImg != '') {
					//获取标题图片
					let headerImg = await this.getImageInfo(this.headerImg);
					//图片截取(以短边为准，截取和设计稿中海报内图片相同比例，保证图片不变形，长边从中心开始截取)
					let ratio = this.imgWidth / this.imgHeight;//设计稿中海报内图片宽高比
					let ratioOrigin = headerImg.width / headerImg.height;//获取图片的宽高比
					if (ratio > ratioOrigin) {//图片扩大/缩小到和海报中图片同一宽度时，高度偏长,
						this.x = 0;//从图片的x位置开始绘制
						this.y = (headerImg.height - (headerImg.width / ratio)) / 2;//从图片Y轴的y位置开始绘制，y位置为高度差的1/2
						this.w = headerImg.width;//图片宽度
						this.h = headerImg.width / ratio;//根据比例计算图片高度
					} else if (ratio < ratioOrigin) {//图片偏宽
						this.x = (headerImg.width - (headerImg.height * ratio)) / 2;
						this.y = 0;
						this.w = headerImg.height * ratio;
						this.h = headerImg.height;
					} else {//图片比例相等
						this.x = 0;
						this.y = 0;
						this.w = headerImg.width;
						this.h = headerImg.height;
					}
					//绘制截取图片
					this.ctx.drawImage(headerImg.path, this.x, this.y, this.w, this.h, 0, 0, this.imgWidth, this.imgHeight);
					//参数说明(需要绘制的图片，X轴从图片的X位置开始，Y轴从图片的y位置开始，绘制图片宽度，绘制图片高度，X轴从画布的0开始，Y轴从画布的0开始，图片最终呈现在画布上的宽度(缩小或放大至该宽度)，图片最终呈现在画布上的高度(缩小或放大至该高度))
					
					//有图片下文字在剩余高度居中显示（限制最多8行，居中显示）
					// #ifdef MP-WEIXIN
					this.maxline=7;
					// #endif
					// #ifdef H5
					this.maxline=9;
					// #endif
					linenum=this.row>this.maxline?this.maxline:this.row;
					this.textY=((this.canvasH - this.imgHeight)-(linenum * this.lineheight)-bottomH)/2+this.imgHeight;
				} else {
					//无图片下文字在除底部的剩余高度居中（限制最多15行，居中显示）
					// #ifdef MP-WEIXIN
					this.maxline=15;
					// #endif
					// #ifdef H5
					this.maxline=17;
					// #endif
					linenum=this.row>this.maxline?this.maxline:this.row;
					this.textY=(this.canvasH-(linenum * this.lineheight)-bottomH)/2;
				}
				//绘制内容
				this.spliceStr(this.subTitle);
				this.ctx.setFontSize(12);
				this.ctx.setFillStyle('#666');
				this.ctx.textAlign = "center";
				this.strList.forEach((element, index) => {
					if(index==this.maxline-1&&element.length==this.linelength){
						//第10行满行则截取后6位显示...
						element=element.substring(0,this.linelength-3)+'...'
					}else if(index>this.maxline-1){
						return;
					}
					this.ctx.fillText(element, this.canvasW / 2,this.textY+this.lineheight*index)

				});
				// #ifdef MP-WEIXIN
					//小程序码绘制
					this.userweixincode= `${this.$u.host()}api/oauth/wechat/miniprogram/code?path=/pages/topic/index?id=${
          this.threadId
        }`//小程序码
					let codeImg=await this.getImageInfo(this.userweixincode)
					this.ctx.drawImage(codeImg.path, 0, 0, codeImg.height,codeImg.width, 15, this.canvasH-75,60,60);
					//长按文字绘制
					this.ctx.setFontSize(10);
					this.ctx.textAlign = "left";
					this.ctx.setFillStyle('#999');
					this.ctx.fillText(this.longpressText, 85, this.canvasH - 50)
					//大文案文字绘制
					this.ctx.setFontSize(10);
					this.ctx.textAlign = "left";
					this.ctx.setFillStyle('#999');
					this.ctx.fillText(this.slitename, 85, this.canvasH - 30)
					
				// #endif
				
				
				// #ifdef H5
				// 用户名称
				this.ctx.setFontSize(10);
				this.ctx.textAlign = "left";
				this.ctx.setFillStyle('#999');
				this.ctx.fillText(this.username, 15, this.canvasH - 15)
				
				//大文案名称
				this.ctx.setFontSize(10);
				this.ctx.textAlign = "right";
				this.ctx.setFillStyle('#999');
				this.ctx.fillText(this.logo, this.canvasW - 15, this.canvasH - 15)
				// #endif

				

				//延迟渲染
				setTimeout(() => {
					this.ctx.draw(true, () => {
						uni.hideLoading();
						this.creatImage()
					})
				}, 1000)
			},

			//截取字符串，按照br换号，截取
			spliceStr(str) {
				//   去掉话题span标签及内容 
				var spanReg = /<span[^>]*>.*?<\/span>/ig;
				str = str.replace(spanReg, '').replace(/\s+/g, "").replace(/<p>/g, '').replace(/<\/p>/g, ''); //去除空格及内容中的p标签
				if (str.indexOf('<br>') != -1) {
					let array = str.split('<br>');
					var reg = new RegExp("<br>", "g");
					array.forEach(element => {
						if (element.length > 17) {
							for (let x = 0; x <= Math.ceil(element.length / 17); x++) {
								if (element.substr(x * 17, 17) != "") {
									this.row = this.row + 1;
									this.strList.push(element.substr(x * 17, 17))
								}
							}
						} else {
							if (element != "") {
								this.strList.push(element);
								this.row = this.row + 1;
							}

						}
					});
				} else {
					let length = str.length,
						row = Math.ceil(length / 17) + 1;
					for (let i = 0; i < row; i++) {
						if (str.substr(i * 17, 17) != "") {
							this.row = this.row + 1;
							this.strList.push(str.substr(i * 17, 17))
						}
					}
				}
			},
			//获取图片
			getImageInfo(imgSrc) {
				return new Promise((resolve, reject) => {
					uni.getImageInfo({
						src: imgSrc,
						success: (image) => {
							resolve(image);
							console.log('获取图片成功', image)
						},
						fail: (err) => {
							reject(err);
							console.log('获取图片失败', err)
						}
					});
				});
			},

			//生成图片
			creatImage() {
				var that = this
				uni.canvasToTempFilePath({
					canvasId: 'my-canvas',
					quality: 1,
					fileType: "jpg",
					success: (res) => {
						console.log('生成图片' + res);
						that.tempImg = res.tempFilePath;
						setTimeout(() => {
							that.showImg=true;
							that.showtip = true;
							// that.stop();
							setTimeout(() => {
							 that.showtip = false;
							}, 2000);
						}, 400);
					},
					fail: (res) => {
						uni.showToast({
							title: '生成图片失败！请稍后再试'
						});
					},
				}, this);
			},
			// 保存图片
			saveImg() {
				uni.saveImageToPhotosAlbum({
					filePath: this.tempImg,
					success: function(res) {
						uni.showToast({
							title: '保存相册成功'
						});
					},
					fail(res) {
						if (res.errMsg == 'saveImageToPhotosAlbum:fail auth deny') {
							uni.showModal({
								title: '您需要授权相册权限',
								success(res) {
									if (res.confirm) {
										uni.openSetting({
											success(res) {},
											fail(res) {
												console.log(res);
											}
										});
									}
								}
							});
						}
					}
				});
			},
		}
	}
</script>

<style scoped lang="scss">
	.content {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: rgba(0, 0, 0, .4);
		z-index: 9999;
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
	}

	.isCan {
		border-radius: 10px;
		z-index: 999;
		position: fixed;
		left: 0;
		top: -99999px;
		margin: auto;
		background-size: 100%;
	}

	.tipToast {
		width: 350rpx;
		height: 80rpx;
		line-height: 80rpx;
		text-align: center;
		z-index: 100000;
		background-color: rgba(0,0,0,.7);
		color: #fff;
		position: fixed;
		top: 40%;
		left: 200rpx;
		font-size: 12px;
	}
</style>
