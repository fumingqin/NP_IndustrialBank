<template>
	<view class="myView">
		<!-- 头部显示信息的view -->
		<view class="topContentInfo">
			<view class="topHead">
				<view>
					<view class="topStation">
						<view class="marginR" style="font-size: 36rpx;">{{startStation}}</view>
						<!-- <view class="marginR">→</view>
						<view>{{endStation}}</view> -->
					</view>
					<view class="topStation topTime">
						<view class="marginR">首 {{firstLastTimeArray[0]}}</view>
						<view class="marginR">末 {{firstLastTimeArray[1]}}</view>
						<!-- <view style="margin-left: 30rpx; color: #FFFFFF; marborder-bottom: 1rpx solid #EEEEEE;white-space: nowrap;width: 300rpx; overflow: hidden;text-overflow: ellipsis;">票价：{{price}}</view> -->
					</view>
				</view>
				<image class="turnImage" src="../../static/Bus/turnBusdirection.png" @click="turnImageClick"></image>
			</view>
			
			<view style="margin-top: 20rpx; margin-left: 30rpx; color: #FFFFFF; marborder-bottom: 1rpx solid #EEEEEE;">票价：{{price}}</view>
			<!-- 时刻表弹框 -->
			<view class="timeClass">
				<u-popup v-model="isShow" mode="center" border-radius="14" width="600rpx">
					<view class="timeClass_text timeClass_title">最近发班时刻</view>
					<block v-for="(item,index) in timeArray" :key="index">
						<view class="timeClass_text timeClass_time">{{item.foreStartTime}}</view>
					</block>
				</u-popup>
			</view>
			<view class="timeButton" @click="timeButtonClick">最近发班时刻</view>
			
			<!-- 暂时没有数据，先隐藏 -->
			<view class="topBottom" v-if="false">
				<view class="topBottomLeft">
					<view class="topBottomText">下一站/3分600米</view>
					<view class="topBottomText">预计到达</view>
				</view>
				<view style="width: 50%;">
					<view class="topBottomText">10:00</view>
					<view class="topBottomText">预计起点发车</view>
				</view>
			</view>
		</view>
		
		<!-- 时间轴 -->
		<scroll-view scroll-x="true" class="busTimeStep">
			<BusTimeSteps :list="stationList" :current="stationInfoArray.ChangStation" :direction="direction" :carLocationArray="carLocationArray"></BusTimeSteps>
		</scroll-view>
		
		<view class="bottomView">
			<!-- <view @click="getBusLocationOntime">公交实时位置</view> -->
			<view style="width: 100%;" @click="turnDirection">横竖转换</view>
		</view>
		
	</view>
</template>

<script>
	var _self;
	import BusTimeSteps from '../../commponents/BUS/BUS.vue';
	export default {
		components:{
			BusTimeSteps
		},
		data() {
			return {
				startStation:'',
				endStation:'',
				price:'1元',
				direction:'row',//默认横向排列
				isClickTurnButton:false,
				stationInfoArray:[],//接收上一个页面传过来的数据
				firstLastTimeArray:[],//时间
				lineDirection:'',//公交上下行方向0:上行 1:下行
				lineID:'',//线路ID
				stationList:[],//站点数组
				carLocationArray:[],//车辆位置数组
				serverTime:'',//服务器时间
				markStationIndex:'',//选择的站点的下标
				isShow:false,//是否显示弹框
				timeArray:[],//发班时间表数组
				timer:null,
				lastPage:'',//记录上一个页面是从哪里进来的
			}
		},
		onLoad(option) {
			_self = this;
			_self.lastPage = '';
			_self.stationInfoArray = JSON.parse(decodeURIComponent(option.lineRoute));
			// console.log(_self.stationInfoArray)
			//从上个页面获取该条线路是上行还是下行
			if(_self.stationInfoArray.LineRouteDirection == '上行' || _self.stationInfoArray.LineRouteDirection == 0){
				_self.lineDirection = 0;
			}else {
				_self.lineDirection = 1;
			};
			//获取服务器时间
			_self.getServerTime();
			//如果上一个页面是从SearchDetail或者BusQuery跳转过来的，因为里面有时间跟线路ID所以不用请求getBusLineInfoByStationName接口，直接赋值
			//如果是从搜索页、搜索详情页、首页进来的话有传线路ID，就不需要再去请求getBusLineInfoByStationName接口
			if(option.lastPage == 'SearchDetail' || option.lastPage == 'BusQuery'){
				uni.setNavigationBarTitle({
					title:_self.stationInfoArray.lineName
				})
				_self.lastPage = 'BusQuery';
				//取到时间，转成数组
				_self.firstLastTimeArray = _self.stationInfoArray.firstLastTime.split('-');
				//取出线路ID
				_self.lineID = _self.stationInfoArray.lineID;
				//获取显示的站点名称（起点-终点）
				_self.startStation = _self.stationInfoArray.StartName + '→' + _self.stationInfoArray.EndName;
				//请求：根据线路查询改线路的所有站点信息，这个接口需要用到线路ID，线路方向
				_self.getStationByLine();
				//请求：根据线路查询车辆实时位置信息，这个接口需要用到线路ID，线路方向
				_self.getBusLocationByStation();
				
			}else {
				_self.lastPage = 'RoutePlan';
				//设置导航栏标题为线路名称
				uni.setNavigationBarTitle({
					title: _self.getLine(_self.stationInfoArray.LineRoute)
				})
				var station = _self.getStartEndStation(_self.stationInfoArray.LineRoute)
				_self.startStation = station[0].substring(1,station[0].length)
				//如果上一个页面是从RoutePlan页面进来的需要先去获取时间跟线路ID
				//根据站点查询线路信息（时间、线路ID）
				_self.getBusLineInfoByStationName();
			}
			//设置定时器，每十秒刷新一次公交位置信息
			_self.setBusTimeInterval();
		},
		onUnload:function(){
			//清楚定时器
			clearInterval(_self.timer);
			_self.timer = null;
		},
		methods: {
//-------------------------------------------功能方法模块开始-------------------------------------------
			
			
			//-------------------------------------------设置定时器，10秒获取一次公交位置信息-------------------------------------------
			setBusTimeInterval:function(){
				_self.timer = setInterval(()=>{
					_self.getBusLocationByStation();
				},10000)
			},
			//-------------------------------------------获取车辆实时位置-------------------------------------------
			getBusLocationOntime:function(){
				_self.getBusLocationByStation();
			},
			//-------------------------------------------调换公交行驶方向-------------------------------------------
			turnImageClick:function(){
				//更换显示的站点名称,调换起点跟终点的位置
				var stationArray = _self.startStation.split('→')
				_self.startStation = stationArray[1] + '→' + stationArray[0]
				
				if(_self.lastPage == 'RoutePlan'){
					//更换线路方向
					if(_self.lineDirection == 0){
						_self.lineDirection = 1;
						_self.getBusLineInfoByStationName();
					}else if (_self.lineDirection == 1){
						_self.lineDirection = 0;
						_self.getBusLineInfoByStationName();
					}
				}else {
					//更换线路方向
					if(_self.lineDirection == 0){
						_self.lineDirection = 1;
						_self.getStationByLine();
						_self.getBusLocationByStation();
					}else if (_self.lineDirection == 1){
						_self.lineDirection = 0;
						_self.getStationByLine();
						_self.getBusLocationByStation();
					}
				}
			},
			//-------------------------------------------调换线路显示的方向-------------------------------------------
			turnDirection:function(){
				_self.isClickTurnButton = !_self.isClickTurnButton;
				if(_self.isClickTurnButton) {
					_self.direction = 'column'
				}else {
					_self.direction = 'row'
				}
			},
			//-------------------------------------------截取线路名称-------------------------------------------
			getLine:function(lineRoute){
				return lineRoute.substring(0,lineRoute.indexOf('('));
			},
			//-------------------------------------------截取站点名称-------------------------------------------
			getStartStation:function(lineRoute){
				var subStr = lineRoute.match(/(\(\S*)(?=→)/)
				return subStr[1].substring(1,subStr[1].length)
			},
			//-------------------------------------------截取首站和末站名称-------------------------------------------
			getStartEndStation:function(lineRoute){
				var subStr = lineRoute.match(/(\(\S*)(?=\))/)
				return subStr
			},
			//-------------------------------------------截取时间-------------------------------------------
			getScheduleTime:function(date){
				return date.split(' ')
			},
			//-------------------------------------------点击时刻表-------------------------------------------
			timeButtonClick:function(){
				_self.isShow = !_self.isShow;
				_self.getTimeScheduleInfo();
			},
//-------------------------------------------功能方法模块结束-------------------------------------------
			
			
			
			
//-------------------------------------------请求方法模块开始-------------------------------------------
			
			
			//----------------------------------根据站点查询线路信息（时间、线路ID）----------------------------------
			getBusLineInfoByStationName:function(){
				uni.request({
					url:_self.$Bus.BusInterface.getBusLineInfoByStationName.Url,
					method:_self.$Bus.BusInterface.getBusLineInfoByStationName.method,
					data:{
						stationName :  _self.getStartStation(_self.stationInfoArray.LineRoute),
						Encryption  :  _self.$Bus.BusInterface.publicCode.encryption
					},
					success(res) {
						console.log('请求线路信息成功',res)
						if(res.data.status){
							//截取出线路方案的站点名称
							var stationName = _self.getLine(_self.stationInfoArray.LineRoute)
							//遍历数组，取出与当前站点相对应的时间
							for(var i = 0; i < res.data.data.length; i++) {
								//取出线路名称相同且线路方向一致的数据
								if(stationName == res.data.data[i].lineName && _self.lineDirection == res.data.data[i].lineDirection){
									//取出区间价格
									_self.price = res.data.data[i].PriceDescribe
									//取到时间，转成数组
									_self.firstLastTimeArray = res.data.data[i].firstLastTime.split('-')
									//取出线路ID
									_self.lineID = res.data.data[i].lineID
									//请求：根据线路查询改线路的所有站点信息，这个接口需要用到线路ID，线路方向
									_self.getStationByLine();
									//请求：根据线路查询车辆实时位置信息，这个接口需要用到线路ID，线路方向
									_self.getBusLocationByStation();
								}
							}
						}else {
							uni.showToast({
								title:res.data.msg,
								icon:'none'
							});
						}
					},
					fail(res) {
						console.log('请求失败',res)
					}
				})
			},
			//----------------------------------根据线路查询该线路的所有站点信息----------------------------------
			getStationByLine:function(){
				uni.showLoading({
					title:'加载中...'
				})
				uni.request({
					url: _self.$Bus.BusInterface.getBusLineStationInfoByLineIdDirection.Url,
					method: _self.$Bus.BusInterface.getBusLineStationInfoByLineIdDirection.method,
					data:{
						lineId     : _self.lineID,
						direction  : _self.lineDirection,
						Encryption : _self.$Bus.BusInterface.publicCode.encryption
					},
					success(res) {
						uni.hideLoading()
						// console.log('站点信息',res)
						if(res.data.status){
							_self.stationList = res.data.data
							//筛选出目标站点的下标，要让目标站点滚动到屏幕中心
							for(var i = 0; i < res.data.data.length; i++){
								if(_self.stationInfoArray.ChangStation == res.data.data[i].stationName){
									_self.markStationIndex = i;
								}
							}
						}else {
							uni.showToast({
								title:res.data.msg,
								icon:'none'
							});
						}
					},
					fail(res) {
						uni.hideLoading()
						console.log('请求失败',res)
					}
				})
			},
			//----------------------------------根据线路编号和运行方向获取某条线路即将到站的车辆信息----------------------------------
			getBusLocationByStation:function(){
				uni.request({
					url: _self.$Bus.BusInterface.getBusArriveLeaveStationInfoByLineIdDirection.Url,
					method: _self.$Bus.BusInterface.getBusArriveLeaveStationInfoByLineIdDirection.method,
					data:{
						lineId      : _self.lineID,
						direction   : _self.lineDirection,
						Encryption  : _self.$Bus.BusInterface.publicCode.encryption
					},
					success(res) {
						// console.log('请求实时到站信息成功',res)
						if(res.data.status){
							for(var i = 0; i < res.data.data.length; i++){
								//车辆时时到站时间戳
								var arriveTime = new Date(res.data.data[i].generationTime).getTime()
								//用服务器时间戳减去车辆到站的时间戳
								var date = _self.serverTime - arriveTime
								//得到的时间差是分钟（车辆到站与服务器的时间差），如果时间超过30分钟就不显示
								var miun = date / (60*60)
								if(miun >= 30){
									res.data.data.splice(i,1)
								}
							}
							//这里的方法不需要，不要管
							// var array = [];
							// for(var i = 0; i < res.data.data.length; i++){
							// 	for(var j = 0; j < _self.stationList.length; j++){
							// 		if(res.data.data[i].stationIndex == j){
							// 			var item = {
							// 				"lineID": _self.stationList[j-1].lineID,
							// 				"stationID": _self.stationList[j-1].stationID,
							// 				"stationName": _self.stationList[j-1].stationName,
							// 				"direction": _self.stationList[j-1].direction,
							// 				"stationIndex": _self.stationList[j-1].stationIndex,
							// 				"lon": _self.stationList[j-1].lon,
							// 				"lat": _self.stationList[j-1].lat,
							// 				"carIndex" : res.data.data[i].stationIndex
							// 			}
							// 			_self.stationList.splice(j-1,1)
										
							// 			_self.stationList.splice(j-1,1,item)
							// 		}
							// 	}
							// }
							
							_self.carLocationArray = res.data.data
							console.log('请求实时到站信息成功',_self.carLocationArray)
						}else {
							if(res.data.msg != "无数据！"){
								uni.showToast({
									title:res.data.msg,
									icon:'none'
								});
							}
						}
					},
					fail(res) {
						console.log('请求实时到站信息失败',res)
					}
				})
			},
			//----------------------------------获取最近发班时刻表----------------------------------
			getTimeScheduleInfo:function(){
				uni.showLoading({
					title:'加载中...'
				})
				uni.request({
					url:_self.$Bus.BusInterface.getBusLineScheduleInfoByLineIdDirection.Url,
					method:_self.$Bus.BusInterface.getBusLineScheduleInfoByLineIdDirection.method,
					data:{
						lineId    : _self.lineID,
						direction : _self.direction,
						Encryption: _self.$Bus.BusInterface.publicCode.encryption
					},
					success(res) {
						uni.hideLoading()
						_self.timeArray = [];
						if(res.data.status){
							for(var i = 0; i < res.data.data.length; i++){
								var timeArray = {
									foreStartTime : _self.getScheduleTime(res.data.data[i].foreStartTime)[1]
								} 
								_self.timeArray.push(timeArray)
							}
						}else {
							uni.showToast({
								title:res.data.msg,
								icon:'none'
							})
						}
					},
					fail(res) {
						uni.hideLoading()
						console.log(res)
					}
				})
			},
			//----------------------------------获取服务器时间----------------------------------
			getServerTime:function(){
				uni.request({
					url: _self.$Bus.BusInterface.getServerTime.Url,
					method: _self.$Bus.BusInterface.getServerTime.method,
					data:{
						Encryption  : _self.$Bus.BusInterface.publicCode.encryption
					},
					success(res) {
						if(res.data.status){
							_self.serverTime = new Date(res.data.data).getTime()
						}else {
							uni.showToast({
								title:res.data.msg,
								icon:'none'
							});
						}
					},
					fail(res) {
						console.log('请求实时到站信息失败',res)
					}
				})
			},
			
			
//-------------------------------------------请求方法模块结束-------------------------------------------
		}
	}
</script>

<style lang="scss">
	$u-itemtMarginLeft:20rpx;
	$u-contentMarginLeft:30rpx;
	page,
	.myView {
		flex-direction: column;
		width: 100%;
		height: 100%;
		background: #F1F1F1;
	}
	
	.topContentInfo{
		padding-top: 30rpx;
		padding-bottom: 30rpx;
		margin-top: 20rpx;
		margin-left: $u-itemtMarginLeft;
		margin-right: $u-itemtMarginLeft;
		border-radius: 12rpx;
		background-color: #35C762;
	}
	.topHead {
		margin-left: $u-contentMarginLeft;
		margin-right: $u-contentMarginLeft;
		display: flex;
		align-items: center;
		justify-content: space-between;
		border-bottom: 1rpx solid #EEEEEE;
	}
	.topStation{
		display: flex;
		align-items: center;
		color: #FFFFFF;
		font-size: 30rpx;
	}
	.topTime{
		margin-top: 20rpx;
		font-size: 26rpx;
		margin-bottom: 30rpx;
	}
	.marginR {
		margin-right: 20rpx;
	}
	.turnImage{
		height: 43rpx;
		width: 52rpx;
		margin-right: 40rpx;
	}
	.timeClass{
		text-align: center;;
		.timeClass_text{
			justify-content: center;
			display: flex;
			align-items: center;
		}
		.timeClass_title{
			height: 80rpx;
			font-size: 36rpx;
			font-weight: 900;
		}
		.timeClass_time{
			height: 60rpx;
			font-size: 30rpx;
			font-weight: 300;
		}
	}
	.timeButton{
		margin-left: $u-contentMarginLeft;
		margin-top: $u-contentMarginLeft;
		font-size: 30rpx;
		font-weight: 600;
		width: 210rpx;
		border-radius: 8rpx;
		padding: 5rpx 10rpx;
		color: #FFFFFF;
		border: 1rpx solid #FFFFFF;
	}
	.topBottom{
		margin-left: $u-contentMarginLeft;
		margin-right: $u-contentMarginLeft;
		margin-top: 30rpx;
		display: flex;
		align-items: center;
		justify-content: space-between;
	}
	.topBottomLeft{
		width: 50%;
		border-right: 1rpx solid #FFFFFF;
	}
	.topBottomText{
		text-align: center;
		color: #FFFFFF;
		font-size: 28rpx;
		margin-bottom: 20rpx;
	}
	.busTimeStep{
		margin-top: 20rpx;
		margin-left: $u-itemtMarginLeft;
		margin-right: $u-itemtMarginLeft;
		width: 710rpx;
		border-radius: 12rpx;
		background-color: #FFFFFF;
		// padding-top: 50rpx;
	}
	.bottomView{
		position: fixed;
		bottom: 0;
		height: 100rpx;
		width: 100%;
		display: flex;
		align-items: center;
		justify-content: space-between;
		background-color: #FFFFFF;
		text-align: center;
	}
	.bottomView>view:first-child{
		width: 50%;
		font-size: 30rpx;
	}
	.bottomView>view:last-child{
		width: 50%;
		font-size: 30rpx;
	}
</style>
