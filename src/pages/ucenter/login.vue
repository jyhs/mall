<template >
  <view class="login">
    <view class="login-list">
      <img class="img" src="https://static.huanjiaohu.com/image/login_header.png">
      <view class="userinfo">
        <view class="userinfo-tel">
          <input maxlength="11" type="number" placeholder="请输入手机号" @input="bindInput">
          <button plain="true" open-type="getPhoneNumber" @getphonenumber="getPhoneNumber">自动填写</button>
        </view>
        <view class="userinfo-code">
          <input type="number" placeholder="短信验证码" @input="bindCode">
          <button
            plain="true"
            v-if="showtime===null"
            @click="getPhoneCode"
            :disabled="codeDisabled"
          >获取验证码</button>
          <div v-else class="captcha-button">{{showtime}}</div>
        </view>
        <view class="userinfo-confirm">
          <button
            v-if="canIUse"
            plain="true"
            :disabled="userInfo2.disabled"
            open-type="getUserInfo"
            @getuserinfo="onConfirm"
            @class="goLoginBtn"
          >进入小程序</button>
        </view>
      </view>
    </view>
  </view>
</template>

<script>
import wx from 'wx';
import api from '@/utils/api';

export default {
  data () {
    return {
      timeCounter: null,
      showtime: null,
      userInfo2: {
        disabled: true
      },
      userInfo: {},
      canIUse: wx.canIUse('button.open-type.getUserInfo')
    };
  },
  onShow () {
    let userInfo = wx.getStorageSync('userInfo');
    let token = wx.getStorageSync('token');
    let self = this;
    console.log('缓存中的个人信息userInfo', userInfo);
    console.log('缓存中的个人信息token', token);
    if (userInfo) {
      this.userInfo = userInfo;
    } else {
      this.userInfo = {};
    }
    wx.getSetting({
      success (res) {
        console.log('res', res);
        if (res.authSetting['scope.userInfo']) {
          wx.getUserInfo({
            success (res) {
              console.log('res.userInfo', res.userInfo);
              // this.userInfo = res.userInfo
              // 用户已经授权过
              self.userInfo = res.userInfo;
              console.log('用户已经授权过');
              wx.navigateTo({
                url: '/pages/ucenter/index'
              });
            }
          });
        } else {
          wx.removeStorageSync('token');
          wx.removeStorageSync('userInfo');
          console.log('用户还未授权过', res);
          // this.userInfo = {};
        }
      }
    });
  },
  methods: {
    countDownText (s) {
      this.showtime = `${s}s后重新获取`;
    },
    // 倒计时 60秒 不需要很精准
    countDown (times) {
      const self = this;
      // 时间间隔 1秒
      const interval = 1000;
      let count = 0;
      self.timeCounter = setTimeout(countDownStart, interval);
      function countDownStart () {
        if (self.timeCounter == null) {
          return false;
        }
        count++;
        self.countDownText(times - count + 1);
        if (count > times) {
          clearTimeout(self.timeCounter);
          self.showtime = null;
        } else {
          self.timeCounter = setTimeout(countDownStart, interval);
        }
      }
    },
    onConfirm (event) {
      if (wx.canIUse('button.open-type.getUserInfo')) {
      } else {
        console.log('请升级微信版本');
      }
      this.userInfo = event.mp.detail.userInfo;
      wx.setStorageSync('userInfo', this.userInfo);
      wx.setStorageSync('token', event.mp.detail.token);
      if (event.mp.detail.rawData) {
        console.log('12345');
        wx.switchTab({
          url: '/pages/ucenter/index'
        });
      } else {
        wx.showModal({
          title: '提示',
          content: '授权后才能进行后续操作',
          success: function (res) {
            if (res.confirm) {
              console.log('用户点击确定');
            } else {
              console.log('用户点击取消');
            }
          }
        });
        this.userInfo2 = {};
      }
      console.log('this.userInfo', this.userInfo);
    },
    // 点击登陆
    goLogin (event) {
      console.log('rawData', event.mp.detail.rawData);
      this.userInfo = event.mp.detail.userInfo;
    },
    getPhoneNumber (e) {
      console.log('111', e.mp.detail.encryptedData);
      console.log(e.mp.detail.iv);
      console.log('222', encodeURIComponent(e.mp.detail.encryptedData));
      wx.login({
        success: res => {
          console.log(res.code);
          wx.request({
            url: 'https://你的解密地址',
            data: {
              encryptedData: encodeURIComponent(e.mp.detail.encryptedData),
              iv: e.mp.detail.iv,
              code: res.code
            },
            method: 'GET', // OPTIONS, GET, HEAD, POST, PUT, DELETE, TRACE, CONNECT
            header: {
              'content-type': 'application/json'
            }, // 设置请求的 header
            success: function (res) {
              if (res.status === 1) {
                // 我后台设置的返回值为1是正确
                // 存入缓存即可
                wx.setStorageSync('phone', res.phone);
              }
            },
            fail: function (err) {
              console.log(err);
            }
          });
        }
      });
    },
    bindInput (e) {
      this.userInfo2.phoneNumber = e.mp.detail.value;
    },
    bindCode (e) {
      console.log('验证码错误', e.mp.detail.value.length);
      this.userInfo2.code = e.mp.detail.value;
      console.log('验证码错误2222', this.userInfo2.code);
      if (this.userInfo2.code.length === 4) {
        this.checkCode();
      }
    },
    async checkCode () {
      const res = await api.validateVerification({
        requestId: this.userInfo2.requestId,
        code: this.userInfo2.code
      });
      console.log('res', res);
      if (res.data) {
        this.userInfo2.disabled = false;
        console.log('验证成功');
      } else {
        wx.showToast({
          title: '验证码错误',
          icon: 'none',
          duration: 1000,
          mask: true
        });
      }
    },
    async getPhoneCode () {
      if (!this.userInfo2.phoneNumber) {
        wx.showToast({
          title: '请填写手机号',
          icon: 'none',
          duration: 1000,
          mask: true
        });
      } else {
        this.countDown(59);
        console.log('this.userInfo2.phoneNumber', this.userInfo2.phoneNumber);
        const res = await api.getVerification({
          phone: this.userInfo2.phoneNumber
        });
        this.userInfo2.requestId = res.requestId;
      }
    }
  },
  // 原生的分享功能
  onShareAppMessage: function () {
    return {
      title: 'xbyjShop',
      desc: '仿网易严选小程序商城',
      path: '/pages/ucenter/login'
    };
  }
};
</script>

<style>
page {
  background: #f4f4f4;
  min-height: 100%;
}

.container {
  background: #f4f4f4;
  min-height: 100%;
}

.login-list {
  width: 100%;
  height: 100vh;
  overflow: hidden;
  background: #fff;
  border-top: 1px solid #e1e1e1;
}

.userinfo-tel,
.userinfo-code {
  display: flex;
  justify-content: space-between;
  margin: 20px;
  border: solid 1px #d6d6d6;
  border-radius: 4px;
}
.userinfo-tel > input,
.userinfo-code > input{
  line-height:48px;
  height:48px;
  font-size:14px;
  justify-content:space-between;
  width:70%;
  padding-left:36px;
}
.userinfo-tel > input
{
      background: url(https://static.huanjiaohu.com/icon/phone.png);
      background-repeat: no-repeat;
      background-size: 8%;
      background-position: 10px;
}
.userinfo-code > input
{
      background: url(https://static.huanjiaohu.com/icon/validate.png);
      background-repeat: no-repeat;
      background-size: 8%;
      background-position: 10px;
}
.userinfo-tel button,
.userinfo-code button{
  border:none;
  color:#278cec;
  height:24px;
  margin:12px 0 0 0;
  align-items:center;
  padding:4px 0;
  border-left:solid 1px #d6d6d6;
  line-height:24px;
  padding:0;
  width:30%;
  font-size:14px;
  text-align:right;
  padding:0 12px;
  border-radius:0;
  flex-shrink:0;
}
.captcha-button{font-size:14px;line-height:48px;padding-right:10px;width:50%;text-align: right}
.userinfo-tel button{border:none;}
.login-list > image{
width:100%;
}

.userinfo-confirm{
  width:100%;
}

.userinfo-confirm button{
  border:none;
  margin:40px 20px 20px 20px;
  color:white;
  height:48px;
  line-height:48px;
  background: #26ab28;
}


</style>
