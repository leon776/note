<template>
<div class="mod-login">
	<div class="logo">
		<p class="title">小柚客服</p>
		<p>一切从这里开始</p>
	</div>
	<div class="form" >
		<div class="inner">
			<h2><img src="../assets/img/logo.png" alt=""></h2>
			<form v-on:submit="sub()" action="javascript:">
				<div class="input-group">
					<label class="item">
						<i class="fa fa-user" aria-hidden="true"></i>
						<input type="text" v-model="postData.username">
					</label>
					<label class="item">
						<i class="fa fa-lock" aria-hidden="true"></i>
						<input type="password" v-model="postData.password">
						<span class="btn" v-on:click="sub()">
							<i class="fa fa-long-arrow-right" v-show="!loading"></i>
							<i class="el-icon-loading" v-show="loading"></i>
						</span>
					</label>
					<div class="item is-noborder">
						<el-checkbox label="记住我" v-model="save"></el-checkbox>
					</div>
					<input type="submit" class="submit">
					<p class="err">{{errText}}</p>
					<!-- <div class="item is-noborder is-center">
						<el-button type="primary" :loading="loading" size="small" v-on:click="sub();" class="sub-btn">{{loading ? '正在登录' : '登录'}}</el-button>
					</div> -->
				</div>
			</form>
		</div>
	</div>
	<el-dialog
		title=""
		:visible.sync="dialogVisible"
		:show-close="false"
		:close-on-click-modal="false"
		top="28vh"
		width="60%">
	  	<div class="update-layer">
			<h2>{{currentMsg}}</h2>
			<el-progress :percentage="autoUpdatePercentage"></el-progress>
		</div>
	</el-dialog>
</div>
</template>

<script>
import Config from '../common/config';
const autoUpdater = Electron.remote.require('electron-updater').autoUpdater;
const log = Electron.remote.require("electron-log");
const dialog = Electron.remote.dialog;

export default {
	data () {
	    return {
	    	win: Electron.remote.getCurrentWindow(),
	        loading: false,
	        save: true,
	        dialogVisible: false,
	        autoUpdatePercentage: 0,
	        updateHandleInited: false,
	        message: {
		      error:'检查更新出错',
		      checking:'正在检查更新……',
		      updateAva:'发现新版本，是否立刻下载？',
		      updateDownload:'正在下载……',
		      updateNotAva:'现在使用的就是最新版本，不用更新',
		      updateEnd:'更新下载完毕，点击确定重启并安装',
		    },
		    currentMsg: '',
	        postData: {
	        	username: Util.getStore('loginData').username,
	        	//'qiaogqiang@163.com',
	        	password: Util.getStore('loginData').password,
	        },
	        errText: '',
	    }
  	},
	mounted: function() {
		//this.init();
		this.$nextTick(() => {
			if(!Config.DEBUG) this.initUpdateHandle();
		})
	},
	activated() {
		this.init();
	},
	methods: {
		init() {
			this.loading = false;
			this.errText = '';
			this.win.setMinimumSize(570, 340);
			this.win.setSize(570, 340);
			this.win.center();
			this.win.show();
		},
		sub() {
			this.loading = true;
			Util.http.post('/op/login', this.postData).then((res) => {
				Util.setStore('token', res.data);
				this.$store.dispatch('refreshToken', res.data);
				if(this.save) {
					Util.setStore('loginData', this.postData);
				}
				this.win.hide();
				this.win.setSize(960, 660);
				this.win.center();
				this.win.setResizable(true);
				this.win.setMinimumSize(930, 660);
				this.initOa()
			}).catch((e) => {
				this.loading = false;
				this.errText = e.data.message;
			});
			
			return false;
		},
		initOa() {
			Util.http.get('oaWechats').then(res => {
	            this.$store.commit('initOa', res.data[0])
				this.$router.push({ path: '/talk'});
	        });
		},
		initUpdateHandle() {
			if(this.updateHandleInited) return; //只搞一次
			this.updateHandleInited = true;
			autoUpdater.logger = log;
			autoUpdater.logger.transports.file.level = "info";
			autoUpdater.setFeedURL(Config.FEED);
			autoUpdater.autoDownload = false;
		    autoUpdater.on('error', error => {
		    	console.error(error)
		    	//this.sendUpdateMessage(message.error)
		    });
		    autoUpdater.on('checking-for-update', () => {
		    	this.sendUpdateMessage(this.message.checking)
		    });
		    autoUpdater.on('update-available', info => {
		    	dialog.showMessageBox({
		    		type: 'info',
				    title: '信息',
				    message: this.message.updateAva,
				    buttons: ['是', '否']
		    	}, index => {
		    		if(index === 0) {
		    			this.dialogVisible = true;
		    			autoUpdater.downloadUpdate();
		    			this.sendUpdateMessage(this.message.updateDownload);
		    		}
		    	});
		    	// if( confirm('发现新版本，是否立刻下载？') ) {
		    	// 	autoUpdater.downloadUpdate();
		    	// } else {
		    	// 	this.dialogVisible = false;
		    	// }
		    });
		    autoUpdater.on('update-not-available', info => {
		        this.sendUpdateMessage(this.message.updateNotAva)
		    });
		    
		    // 更新下载进度事件
		    autoUpdater.on('download-progress', progress => {
		    	this.autoUpdatePercentage = Math.round(progress.percent);
		    })

		    autoUpdater.on('update-downloaded', () => {
		    	this.sendUpdateMessage(this.message.updateEnd);
		    	dialog.showMessageBox({
		    		type: 'info',
				    title: '信息',
				    message: this.message.updateEnd,
				    buttons: ['确定']
		    	}, index => {
		    		autoUpdater.quitAndInstall();
		    	});
		    });
		    
		    //执行自动更新检查
		    console.log( autoUpdater.checkForUpdates() );
		},
		sendUpdateMessage(txt) {
			console.log(txt)
			this.currentMsg = txt;
		}
	}
}
</script>

<style scoped lang="scss">
	.mod-login{
		background: #2e93ee;height: 100%; position: relative;
		-webkit-app-region: drag;
		.update-layer{
			h2{
				font-size: 14px;text-align: center; margin-top: -28px;
			}
			.el-progress{
				//position: absolute;
				//width: 60%; left: 20%; top: 50%; margin-top: -9px;
			}
		}
	}
	.form{
		width: 64%;height: 100%;
		float: right;
		background: #fff;
		.inner{
			text-align: center; padding: 48px 70px 0;
			h2{
				font-size: 16px;
				img{
					width: 64px; height: 64px;
				}
			}
			.err{
				color: #FF4949;margin-top: 16px;
				height: 12px;line-height: 12px;text-align: center;
			}
			.input-group{
				text-align: left; margin-top: 24px;
				.item{
					display: block; position: relative;
					border-bottom: 1px solid #ccc; padding: 8px 0;line-height: 18px; height: 18px;
					&.is-noborder{border: none; margin-top: 8px;}
					&.is-center{text-align: center;}
					i,span,input{vertical-align: middle;}
				}
				.fa{
					color: #666;font-size: 15px; width: 12px;text-align: center;
					&.fa-long-arrow-right{
						overflow: hidden;
						&:before{margin-left: -3px;}
					}
				}
				.btn{
					width: 24px;height: 24px;float: right;text-align: right;position: absolute;
					right: 0;
					i{color: #20A0FF}
				}
				.sub-btn{width: 120px;}
			}
		}
		input{border:none; background: none;padding:0;font-size: 13px;height: 18px;padding-left: 8px;}
		.submit{opacity: 0;position: absolute;}

	}
	.logo{
		position: absolute;top: 60%;margin-top: -50px; left: 0; width: 36%;
		font-size: 12px;color: #fff;text-align: center; 
		// &:before{
		// 	display: block;background: url('../../assets/img/logo-white.png') no-repeat center;
		// 	background-size: auto 100%; content: ''; position: absolute;
		// 	width: 48px; height: 48px; left: 50%; margin-left: -24px;
		// 	top: -72px;
		// }
		.title{font-size: 20px;margin-bottom: 8px;}
	}
</style>
<style lang="scss">
	.mod-login .el-checkbox .el-checkbox__inner{width: 14px;height: 14px;}
	.mod-login .el-checkbox span{font-size: 12px!important;margin-top: 1px;}
	.mod-login .el-checkbox .el-checkbox__inner::after {
	    height: 6px;
	    left: 4px; 
	    top: 1px;
	    width: 3px;
	}
	.mod-login .el-checkbox__input{line-height: inherit;}
	/*定义滚动条高宽及背景 高宽分别对应横竖滚动条的尺寸*/  
	::-webkit-scrollbar  {  
	    width: 8px;  
	    height: 8px;  
	    background-color: #F5F5F5;  
	}  
	  
	/*定义滚动条轨道 内阴影+圆角*/  
	::-webkit-scrollbar-track  {  
	    
	    background-color: #F5F5F5;  
	}  
	  
	/*定义滑块 内阴影+圆角*/  
	::-webkit-scrollbar-thumb {  
	    border-radius: 10px;  
	    -webkit-box-shadow: inset 0 0 6px rgba(0,0,0,.3);  
	    background-color: #999;  
	}
</style>
