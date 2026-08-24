# 微信授权登录接入说明

本次新增：小程序端与 App 端的微信授权登录，登录时强制获取已验证手机号，可选获取头像/昵称，登录成功后请求一次定位权限并静默获取位置。

## 新增/修改的前端文件

- `types/auth.uts`：新增 `WechatAppLoginResult` 类型
- `types/location.uts`：新增 `LocationInfo` 类型
- `api/auth.uts`：新增 `wechatMiniLogin`、`wechatAppLogin`、`bindWechatPhone`；`sendVerificationCode` 增加可选 `scene` 参数
- `api/upload.uts`：新增头像上传 `uploadAvatar`
- `stores/location.uts`：定位结果本地存储
- `utils/location.uts`：定位权限申请 + 静默获取位置（小程序走 `authorize`，App 端 `getLocation` 自动弹系统权限）
- `pages/auth/wechat.uvue`：微信登录页（小程序 = 头像/昵称可选 + 手机号快捷验证按钮必选；App = 微信授权按钮）
- `pages/auth/bind-phone.uvue`：App 端微信登录后若未绑定手机号，走短信验证码补充绑定
- `pages/auth/login.uvue`：新增"微信一键登录"入口；手机号/邮箱登录成功后也会触发一次定位
- `pages.json`：注册上述两个新页面
- `manifest.json`：mp-weixin 增加 `requiredPrivateInfos: ["getLocation"]`；app 增加 `uni-oauth.weixin` 占位配置；app-ios 增加 `Geolocation` 模块与定位隐私描述占位

## 需要后端新增的接口契约

所有接口沿用现有 `{ code, message, data }` 响应包装（见 `services/http.uts`）。

**POST `/rc/auth/wechat-mini-login`**（小程序）
请求：`{ code, phone_code, nickname, avatar }`（`code` 为 `wx.login` 登录凭证，`phone_code` 为 `getPhoneNumber` 按钮返回的手机号凭证，`nickname`/`avatar` 可为 null）
后端需用 `code` 换取 openid/session_key（`auth.code2Session`），用 `phone_code` 调用微信 `phonenumber.getPhoneNumber` 接口换取已验证手机号。
响应：与现有登录接口一致的 `LoginResult`：`{ token_type, access_token, user }`

**POST `/rc/auth/wechat-app-login`**（App）
请求：`{ code, nickname, avatar }`（`code` 为 App 端 `uni.login({provider:'weixin', onlyAuthorize:true})` 返回的授权码，走微信开放平台 OAuth2 换取 openid/unionid；`nickname`/`avatar` 为微信侧返回的资料，可能为空）
App 端微信 SDK 无法像小程序一样直接给出已验证手机号。若该微信身份（按 unionid/openid 匹配）尚未绑定手机号，返回：
`{ status: "need_phone", pending_token, token_type: null, access_token: null, user: null }`
若已绑定手机号（历史账号或已有映射），直接返回：
`{ status: "ok", token_type, access_token, user }`

**POST `/rc/auth/wechat-bind-phone`**（App 补充绑定）
请求：`{ pending_token, phone, code }`（`code` 为短信验证码，通过 `POST /rc/auth/send-verification-code` 加 `scene: "bind"` 获取）
后端校验 `pending_token` 对应的微信身份 + 短信验证码，创建/关联账号并绑定手机号。
响应：`LoginResult`

**POST `/rc/upload/avatar`**
`multipart/form-data`，文件字段名 `file`。用于把小程序 `chooseAvatar` 拿到的本地临时头像上传为可长期访问的 URL。
响应：`{ url }`

## 需要在微信后台/HBuilderX 中配置的凭证与权限

1. **小程序端**
   - 手机号快速验证（`getPhoneNumber`）需要小程序完成微信认证且具备相应权限，个别类目/未认证小程序可能拿不到该按钮权限，需在 [mp.weixin.qq.com](https://mp.weixin.qq.com) 后台确认。
   - 使用 `wx.getLocation` 前需在小程序后台"设置-隐私"里完成"用户隐私保护指引"配置，声明会收集地理位置信息，否则接口会报隐私协议未同意的错误。
   - `manifest.json` 中 mp-weixin 的 `appid`（`wxca11d49980c96626`）已存在，无需改动。

2. **App 端**
   - 需要在微信开放平台（open.weixin.qq.com，与小程序后台是两个不同的账号体系）单独创建移动应用并提交审核，拿到 App 专属的 AppID / AppSecret（`manifest.json` 中 `app.distribute.modules.uni-oauth.weixin` 的 `appid`/`appsecret` 目前是占位符，需要替换成真实值）。
   - iOS 端还需在开放平台配置 Universal Links，并填入 `UniversalLinks` 字段；Android 端需要在开放平台登记应用签名（release 签名）与包名。
   - 建议在 HBuilderX 可视化 manifest 编辑器里的"App 模块配置 → OAuth(登录鉴权) → 微信登录"里核对一遍这些字段是否被正确识别，不同 HBuilderX 版本对该节点的字段名可能略有出入。

3. **定位权限**
   - 小程序：`manifest.json` 已声明 `scope.userLocation` 权限描述及 `requiredPrivateInfos`。
   - Android：已启用 `uni-location` 模块，调用 `uni.getLocation` 会自动弹出系统权限申请。
   - iOS：已在 `app-ios.distribute` 加入 `Geolocation` 模块与 `NSLocationWhenInUseUsageDescription` 占位文案，可按需修改措辞。

## 尚待你确认/在真机联调时留意的点

- App 端 `uni.login({ provider: 'weixin', onlyAuthorize: true })` 返回的授权码字段名在不同平台/HBuilderX 版本上可能是顶层 `code` 或嵌套在 `authResult.code`，`pages/auth/wechat.uvue` 里的 `extractAppLoginCode` 做了兼容处理，但建议真机联调时打印一次原始返回值确认。
- App 端 `uni.getUserInfo({ provider: 'weixin' })` 自 2022 年后微信收紧了隐私权限，新授权很可能只返回默认昵称/头像甚至为空，代码已做静默降级（拿不到就不填，不影响登录）。
- `platformConfig.json` 当前只勾选了 `APP-ANDROID` 作为运行目标，若要联调 iOS 微信登录，需要把 `APP-IOS` 也加入 `targets`。
