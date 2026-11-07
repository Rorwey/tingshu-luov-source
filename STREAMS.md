# 直播源与封面清单（核验手册）

本文件是 `sources_by_luov.jar` 所用直播流、封面图 URL 的**可复制清单**，用于后续快速接入与复核。
核验时间：2026-11-07 ~ 11-08（北京时间）。App 侧订阅入口见根目录 `external_sources.json`。

## 0. 核验口径（复现时请照此执行）

一条流判定为「可用」必须同时满足：

1. 主播放列表 HTTP 200，且 body 以 `#EXTM3U` 开头（排除伪装成 200 的错误 JSON / HTML）；
2. 若有子播放列表（`#EXT-X-STREAM-INF`），跟随第一档子列表也 200；
3. 取第一个 ts 分片，HTTP 200；
4. 主/子列表中**不含 `#EXT-X-KEY`**（本 App 的 `AudioUrlDirectExtractor` 无法解 AES / ChinaDRM）；
5. **活性检测**：间隔 8~9 秒取两次 `#EXT-X-MEDIA-SEQUENCE`，序号必须滚动（排除 CDN 静态缓存壳）。

标记含义：`[已接入]` 已写进 jar；`[备选]` 实测可用但未纳入订阅源；`[不可用]` 已实测排除。
「需签名」= 裸 URL 直接 403，必须播放时动态注入鉴权参数，详见第 1 节。

---

## 1. 动态签名机制（订阅源是"动态"的）

订阅源不是静态清单，而是 Kotlin 编译的 dex-jar，App 加载 `SourceEntry` 后可执行任意代码。
因此签名**不硬编码进 jar、也不写进收藏记录**：Book 里存稳定裸路径，点播放时由
`getBookDetailInfo()` 调用 `StreamSigner.resolve()` 实时联网反查，注入最新参数。

| 站点 | 反查入口 | 注入参数 | 备注 |
| --- | --- | --- | --- |
| 央广国家台 16 套 | `POST https://apppc.cnr.cn/national`，body `{}` → `data.categories[].detail[].other_info11[0].url` | `key=<md5>&time=<unix秒>` | 接口返回 2 个内容相同的分类，取其一即可；`time` 为生成时刻，实测 12 分钟前的旧签名仍 200 |
| 南昌广电 5 条 | `GET https://www.nctv.net.cn/live?channel=2`，正则 `data-url="([^"]+)"[^>]*?data-title="([^"]+)"` | `txSecret=<md5>&txTime=<hex>` | 约 30 分钟滚动；广播频率额外带 `only-audio=1` |
| 国广 CRI 3 套 | 无需反查 | 无 | `sk.cri.cn` 裸链实测 200，直接透传 |

**v9 加固**：列表页拿到签名即 `StreamSigner.remember()` 预热进程内缓存；播放时每次仍联网刷新，
若接口/官网偶发失败则用缓存里最近一次成功的签名兜底，不再退化成 403 裸链。

**「纯本地按时间算签名」不可行**：央广 `key` 为服务端私有算法，用 4 组实测数据尝试
`md5(path+time)`、`md5(time+path)`、`md5(path+time+cnr)`、`md5(time)` 等组合全部不命中，
必须走一次网络反查（只在点播放时发生）。

---

## 2. 封面图 URL（全部实测 HTTP 200，image 格式）

| 用途 | URL |
| --- | --- |
| 央广 / 湖南新闻类 | `https://cnvod.cnr.cn/audio2017/ondemand/media/1100/20170614/1497425171192.jpg` |
| 央广 / 湖南潇湘类 | `https://cnvod.cnr.cn/audio2017/ondemand/img/1100/20191217/1576570410135.png` |
| 央广 / 湖南音乐类 | `https://cnvod.cnr.cn/audio2017/ondemand/img/1100/20200413/1586764109369.jpg` |
| 江西新闻类 | `https://cnvod.cnr.cn/audio2017/ondemand/media/1100/20170703/1499072543804.jpg` |
| 江西交通类 | `https://cnvod.cnr.cn/audio2016/default/2014/8/18/1408354161968.jpg` |
| 江西音乐类 | `https://cnvod.cnr.cn/audio2016/default/2014/8/18/1408354248154.jpg` |
| 南昌新闻综合频率 FM91.7 | `https://www.nctv.net.cn/images/pl/917.jpg` |
| 南昌交通广播 FM95.1 | `https://www.nctv.net.cn/images/pl/951.jpg` |
| 南昌新闻综合频道 | `https://www.nctv.net.cn/images/ngxw1.jpg` |
| 南昌文旅频道 | `https://www.nctv.net.cn/images/ngxw2.jpg` |
| 南昌资讯频道 | `https://www.nctv.net.cn/images/ngxw3.jpg` |
| 湖南卫视 | `https://parco-zh.github.io/demo/HNWS.png` |
| 江西卫视 | `https://www.tvchinese.net/uploads/tv/jiangxi.jpg` |
| 央视 / 央视频道（备用台标） | `https://www.tvchinese.net/uploads/tv/cctv1.jpg`、`.../cctvnews.jpg` |

央广接口返回的各省台封面走 `other_info6`，缺失回退 `other_info24`（同属 `cnvod.cnr.cn`）。
未找到九江交通广播官方台标，当前用「江西交通类」封面占位，可随时替换。

---

## 3. 央广 / 国广 国家广播 19 套 `[已接入]`

分类菜单第一项「央广」。下表为 Book 中保存的**稳定裸路径**；标「需签名」的必须经
第 1 节反查后才可播放（裸链 403）。接口返回的 2 个分类内容完全相同，v9 已按裸 URL 去重，
列表由 38 条恢复为 19 条。

| # | 台名 | 稳定裸 URL | 签名 |
| --- | --- | --- | --- |
| 1 | 中国之声 | `https://ngcdn001.cnr.cn/live/zgzs/index.m3u8` | 裸链可播 |
| 2 | 经济之声 | `https://ngcdn002.cnr.cn/live/jjzs/index.m3u8` | 裸链可播 |
| 3 | 音乐之声 | `https://ngcdn003.cnr.cn/live/yyzs/index.m3u8` | 需签名 |
| 4 | 经典音乐广播 | `https://ngcdn004.cnr.cn/live/dszs/index.m3u8` | 需签名 |
| 5 | 台海之声 | `https://ngcdn005.cnr.cn/live/zhzs/index.m3u8` | 需签名 |
| 6 | 神州之声 | `https://ngcdn006.cnr.cn/live/szzs/index.m3u8` | 需签名 |
| 7 | 大湾区之声 | `https://ngcdn007.cnr.cn/live/hxzs/index.m3u8` | 需签名 |
| 8 | 民族之声 | `https://ngcdn009.cnr.cn/live/mzzs/index.m3u8` | 需签名 |
| 9 | 文艺之声 | `https://ngcdn010.cnr.cn/live/wyzs/index.m3u8` | 需签名 |
| 10 | 老年之声 | `https://ngcdn011.cnr.cn/live/lnzs/index.m3u8` | 需签名 |
| 11 | 藏语广播 | `https://ngcdn012.cnr.cn/live/zygb/index.m3u8` | 需签名 |
| 12 | 维语广播 | `https://ngcdn013.cnr.cn/live/wygb/index.m3u8` | 需签名 |
| 13 | 香港之声 | `https://ngcdn008.cnr.cn/live/xgzs/index.m3u8` | 需签名 |
| 14 | 中国交通广播 | `https://ngcdn016.cnr.cn/live/gsgljtgb/index.m3u8` | 需签名 |
| 15 | 中国乡村之声 | `https://ngcdn017.cnr.cn/live/xczs/index.m3u8` | 需签名 |
| 16 | 哈语广播 | `https://ngcdn025.cnr.cn/live/hygb/index.m3u8` | 需签名 |
| 17 | 环球资讯广播 | `https://sk.cri.cn/905.m3u8` | 裸链可播 |
| 18 | 南海之声 | `https://sk.cri.cn/nhzs.m3u8` | 裸链可播 |
| 19 | 英语资讯广播 CGTN Radio | `https://sk.cri.cn/am846.m3u8` | 裸链可播 |

地方广播 30 省由 `POST https://apppc.cnr.cn/local` body `{"id":"<省id>"}` 动态拉取，
流多为 `https://satellitepull.cnr.cn/live/<code>/playlist.m3u8`，无需签名。

---

## 4. 江西 省级 / 市级广播 `[已接入]`

分类「江西」(省 id `3236`)，顺序：省级 → 市级按车牌字母序（赣A 南昌 → 赣B 赣州 → 赣G 九江）。

### 4.1 省级 — 江西广电官网自营直连（6 频率，官网 `jxntv.cn/live/radio`）

| 台名 | URL |
| --- | --- |
| 江西故事广播 FM96.9 | `https://fm-live-pull.jxtvcn.com.cn/live-jxrtv/fm969.m3u8` |
| 江西旅游广播 FM97.4 | `https://fm-live-pull.jxtvcn.com.cn/live-jxrtv/fm974.m3u8` |
| 江西财经广播 FM99.2 | `https://fm-live-pull.jxtvcn.com.cn/live-jxrtv/fm992.m3u8` |
| 江西音乐广播 FM103.4 | `https://fm-live-pull.jxtvcn.com.cn/live-jxrtv/fm1034.m3u8` |
| 江西新闻广播 FM104.4 | `https://fm-live-pull.jxtvcn.com.cn/live-jxrtv/fm1044.m3u8` |
| 江西交通广播 FM105.4 | `https://fm-live-pull.jxtvcn.com.cn/live-jxrtv/fm1054.m3u8` |

### 4.2 省级 — 央广 CDN 备用线路（v9 新增，免签名、活性 LIVE）

| 台名 | URL | 活性（seq 双快照） |
| --- | --- | --- |
| 江西综合新闻广播（央广线路） | `https://satellitepull.cnr.cn/live/wx32jiangxxwgb/playlist.m3u8` | 1794067380 → 1794067434 |
| 江西信息交通广播（央广线路） | `https://satellitepull.cnr.cn/live/wx32jiangxjtgb/playlist.m3u8` | 1794067383 → 1794067437 |
| 江西音乐广播（央广线路） | `https://satellitepull.cnr.cn/live/wx32jiangxyygb/playlist.m3u8` | 1794067386 → 1794067440 |

### 4.3 市级

| 台名 | URL | 签名 |
| --- | --- | --- |
| 南昌新闻综合频率 FM91.7 | `https://live.nctv.net.cn/live/fm917.m3u8?only-audio=1` | 需签名（官网） |
| 南昌交通广播 FM95.1 | `https://live.nctv.net.cn/live/fm951.m3u8?only-audio=1` | 需签名（官网） |
| 南昌交通音乐广播（备用） | `https://ls.qingting.fm/live/1804.m3u8` | 无 |
| 赣州音乐广播 | `https://ls.qingting.fm/live/4942.m3u8` | 无 |
| 九江交通广播 FM88.4/88.9 | `https://ls.qingting.fm/live/2785094.m3u8` | 无，v9 新增，seq 10901524 → 10901538 |

---

## 5. 湖南 扩展台 `[已接入]`（分类「湖南」id `3238`）

省级：`https://satellitepull.cnr.cn/live/wx32hunjtgb/playlist.m3u8`（交通）、
`wx32hunjjgb`（经济）、`wx32955`（金鹰之声）、`https://ls.qingting.fm/live/4979.m3u8`（音乐之声）。
市级（湘A → 湘B）：长沙新闻广播 `.../live/4877.m3u8`、长沙城市之音 `4237`、
长沙品味音乐广播 `4930`、长沙交通广播 `3967`、株洲交通广播 `3971`（前缀均为
`https://ls.qingting.fm`）。

v10 另补衡阳（交通 `5079921`、新闻 `5079970`）、岳阳（交通 `88931`、新闻综合 `88933`）、
郴州（综合 `76765`、音乐交通 `86747`）；全国其他省份的补充台见第 9 节。

---

## 6. 电视源 `[已接入]`（子源「央视/地方卫视」）

命名规则：**伴音才标注「（伴音）」，真视频流不标注**。

| 分类 | 显示名 | URL | 签名 |
| --- | --- | --- | --- |
| 卫视伴音 | 湖南卫视（伴音） | `https://satellitepull.cnr.cn/live/wx32hunws/playlist.m3u8` | 无 |
| 卫视伴音 | 江西卫视（伴音） | `https://satellitepull.cnr.cn/live/wx32jiangxws/playlist.m3u8` | 无 |
| 南昌电视 | 南昌新闻综合频道 | `https://live.nctv.net.cn/live/nctv1.m3u8` | 需签名（官网） |
| 南昌电视 | 南昌文旅频道 | `https://live.nctv.net.cn/live/nctv2.m3u8` | 需签名（官网） |
| 南昌电视 | 南昌资讯频道 | `https://live.nctv.net.cn/live/nctv3.m3u8` | 需签名（官网） |

南昌官网 `www.nctv.net.cn/live?channel=1..8` 实测均返回同样 5 条（`data-url` + `data-title` + `data-img`），
无更多频道可挖。

---

## 7. 备选：实测可用但**未接入**的央视系流 `[备选]`

来源为 GitHub 聚合清单 `iptv-org/iptv`(`streams/cn.m3u`)、`fanmingming/live` 及官方 CDN。
以下均通过第 0 节全部 5 项核验（无 `#EXT-X-KEY`、seq 实时滚动），但未写进 jar：
央视官网 `tv.cctv.com` 现为 ChinaDRM 加密流，按既有约定「需要加密就算了」放弃；
下列多为第三方转播 / CGTN 国际频道，非官方直连，稳定性无保证，且对纯音频 App 价值有限。
若日后要启用，直接加进 `LocalTV.kt` 的 `stations` 列表即可（真视频流 `audioOnly=false`、不标注）。

| 频道 | URL | 活性 |
| --- | --- | --- |
| CCTV-1（voc.com.cn 转播） | `https://liveplay-srs.voc.com.cn/hls/tv/134_180adf.m3u8` | LIVE 8129 → 8140 |
| CGTN 英语 | `https://0472.org/hls/cgtn.m3u8` | LIVE |
| CGTN 纪录 | `https://0472.org/hls/cgtnd.m3u8` | LIVE |
| CGTN 阿语 | `https://0472.org/hls/cgtna.m3u8` | LIVE |
| CGTN 西语 | `https://0472.org/hls/cgtnx.m3u8` | LIVE |
| CGTN Global Biz | `https://fastlive.cctvplus.com/out/v1/2b3d8a0c805d437f9dc85b764c2ac599/index.m3u8` | LIVE |
| China Travel | `https://fastlive.cctvplus.com/out/v1/ca6f9297b7314a63959435028af287fc/index.m3u8` | LIVE |
| Discovering China | `https://fastlive.cctvplus.com/out/v1/8442c3bfd4c64a3984389328f6087879/index.m3u8` | LIVE |

`https://0472.org/hls/cgtnf.m3u8`（法语）、`cgtne.m3u8`（俄语）取不到 seq，标 UNKNOWN，不列入。

---

## 8. 已实测排除 `[不可用]`（避免重复踩坑）

| 对象 | 结论 |
| --- | --- |
| 江西广电官网电视 8 频道 | 官网 SPA `www.jxntv.cn/live` 的取流链已完整逆出：`POST https://cdnauth.jxgdw.com/liveauth/pc` body `{t,stream,uuid}` → `{t,token}`，再拼 `https://yun-live.jxtvcn.com.cn/live-jxtv/<stream>?source=pc&t=&token=&uuid=`。鉴权接口 200 正常返回，但播放 CDN（网宿 `MC_VCLOUD_LIVE`）一律 403。 |
| 上述 403 的开源算法复现 | 已按 GitHub `linux-jin/TvBox` 的「江西电视台PC官网代理源码」及恩山论坛算法逐一试过：`Authorization=md5(t+stream+etag+密钥)`（3 个密钥）、`etag` 随机 8 位、`t` 10 位/15 位、去掉 `uuid`、只带 `token`、换 `jsp-tv-live.jxtvcn.com.cn` 域名、旧接口 `app.jxntv.cn/Qiniu/liveauth/getPCAuth.php`（已下线）——共 36 种组合全部 403。判断为 CDN 侧 IP / 风控校验，非参数可解，按约定放弃。频道表：`tv_jxtv1/2/3_hd/5/6/7/8.m3u8`、`tv_taoci.m3u8`。 |
| 江西卫视 / 江西各频道 IPTV 流 | `39.134.115.163:8080`、`112.25.48.68`、`183.207.248.71`、`183.207.249.11`、`223.110.245.170`、`223.110.254.199:6610`、`117.169.120.140:8080`、`125.210.152.18:9090`、`111.75.179.195:30767`（抚州公共）均为运营商内网 IP，公网实测全部超时；`play-live-hls.jxtvcn.com.cn/live-city/tv_jxtv*.m3u8` 全部 403。 |
| 央广 GitHub 备用流 | `live.xmcdn.com/live/<id>/64.m3u8` 返回 HTTP 200 但 body 是 `{"ret":2011,"msg":"电台流获取失败"}`；`ls.qingting.fm/live/386|388|4985.m3u8` 全 404。故国家台只能走官网动态签名。 |
| 74.91.26.218:82 CCTV 1-17 | 302 跳 `192.187.123.50` 带 `tm`+`key` 动态签名，部分频道返回「频道 cctv5 没有配置任何服务器」，不采纳。 |
| 湖南年代音乐台 | `wx32hunlygb` 404，源码 `deadCodes` 已过滤。 |
| 江西都市 / 民生 / 农村广播 | `wx32jiangxdsgb` / `wx32jiangxmsgb` / `wx32jiangxncgb` 连不上；官网 `fm1019` 民生广播同样连不上，与 `jxgdw.cn` 停播公告吻合。 |
| 湘潭广电、喜马拉雅金鹰955 | 前者分片时间戳冻结在 20250830、seq 2619 不动；后者 200 但 body 为错误 JSON。 |

---

## 9. 全国补充台数据表（v10 新增，25 省 106 台）`[已接入]`

央广官方接口 `apppc.cnr.cn/local` 每省只返回 1~12 个台，且**没有任何地市级电台**（`local/list` 只列 30 个省；`city`、`local/city`、`city/list` 等端点均返回 500）。
v10 因此新增 `RadioData.kt` 数据表，按省补齐接口未覆盖的省级频率与地市级电台，**每条都过三道门**：主列表 200 → 首个分片 / 子列表 200 → 间隔 8 秒两次快照 `MEDIA-SEQUENCE` 必须增长。不含 `#EXT-X-KEY`，不带会过期的鉴权参数。

来源代号：`Q` = 蜻蜓直播 `https://ls.qingting.fm/live/{标识}.m3u8`；`C` = 央广 CDN `https://satellitepull.cnr.cn/live/{标识}/playlist.m3u8`。

| 省 | 编号 | 补充台（来源:标识或频率） | 台数 | 说明 |
| --- | --- | --- | --- | --- |
| 北京 | `3226` | 北京交通广播 FM103.9、北京好音乐 FM95.9、北京文艺广播 FM87.6、北京音乐广播 FM97.4 | 4 | 接口只有新闻广播/京津冀之声 2 个台，补交通/文艺/音乐/好音乐 |
| 河北 | `3229` | 保定交通广播 FM104.8、保定新闻广播 FM93.7、保定经典964汽车音乐广播、河北农民广播 AM558、河北故事广播 FM107.9、河北新闻广播 FM104.3、河北旅游广播 AM603、河北私家车广播 FM90.7 | 8 | 接口已 6 台，补保定 3 个地市台 + 故事/农民/旅游/私家车 |
| 山西 | `3231` | 太原交通广播 FM107、太原新闻广播 FM91.2、太原私家车Radio FM104.4、太原音乐广播 FM102.6、长治交通文艺广播 FM94.9、长治新闻综合广播(幸福广播) FM94.3、阳泉交通广播、阳泉新闻综合广播 | 8 | 接口只有综合广播 1 个台，补太原、长治、阳泉地市台 |
| 内蒙古 | `3249` | 呼和浩特交通广播 FM107.4、呼和浩特新闻综合广播 FM92.9 | 2 | 补呼和浩特 |
| 辽宁 | `3245` | 沈阳新闻广播 FM104.5、沈阳都市广播 FM92.1、沈阳音乐广播 路上好朋友 FM98.6 | 3 | 补沈阳 |
| 黑龙江 | `3246` | 龙广青苹果之声 FM104.6 | 1 | 接口已 9 台，仅补青苹果之声 |
| 上海 | `3228` | 东广新闻台 FM90.9、第一财经广播（C:wx32dycjgb） | 2 | 接口只有东方广播/新闻广播，补东广新闻台与浦江之声 |
| 江苏 | `3232` | 南通交通广播 FM92.9、南通新闻广播 FM97.0、常州交通广播 FM90、无锡交通广播 FM106.9、无锡新闻广播 FM93.7 | 5 | 补常州、南通、无锡 |
| 浙江 | `3233` | 宁波交通广播 FM93.9、宁波动感105 FM105.2、温州交通广播 FM103.9、温州新闻广播 FM94.9、温州私家车音乐广播 FM100.3、温州经济生活广播 FM88.8、温州绿色之声 FM93.8、湖州交通文艺广播 FM98.5、湖州经济广播 FM103.5、湖州综合广播 湖州之声 FM05 | 10 | 补宁波、温州、湖州 |
| 福建 | `3235` | 福建私家车广播 FM98.7 | 1 | 接口已 5 台，补福建私家车 |
| 山东 | `3237` | 山东女主播电台 FM97.5、山东新闻广播 FM95、山东生活广播 MyFM FM105、山东经典音乐广播（C:wxsdshxxgb）、济南故事广播 FM104.3、济南私家车广播 FM93.6、济南经济广播 FM90.9、潍坊新闻广播 FM100.2、潍坊私家车广播 FM93.3、潍坊音乐优生活 FM90.8、阳信人民广播电台 FM103.4、青岛交通广播 FM89.7、青岛故事广播 FM95.2、青岛新闻广播 FM107.6、青岛西海岸城市生活广播 FM92.6 | 15 | 补济南、青岛、潍坊地市台与阳信县台 |
| 河南 | `3230` | 郑州新闻广播 FM98.6、郑州汽车广播 FM91.2、郑州活力944、郑州私家车 FM91.8、郑州经典广播 FM107.9 | 5 | 补郑州 5 个频率 |
| 湖北 | `3239` | 楚天交通广播（C:wx32hubctjtgb）、襄阳交通广播 FM89.0、襄阳音乐广播 | 3 |  |
| 湖南 | `3238` | 岳阳交通广播 FM106.1、岳阳新闻综合广播、衡阳交通广播 FM101.8、衡阳新闻广播 FM98.9、郴州综合广播 FM99.2、郴州音乐交通广播 FM102.8 | 6 | 接口只有 3 个台；长沙/株洲见第 5 节，本表补衡阳/岳阳/郴州 |
| 广东 | `3240` | 广东优悦广播(南粤) FM105.7、广州 MYFM 88.0 (都市生活)、广州交通电台 FM106.1、广州新闻电台 FM96.2、广州汽车音乐电台 FM102.7、广州花都广播 FM100.5、梅州新闻广播 FM94.8、深圳快乐1062(交通广播)、深圳飞扬音乐971、珠海电台交通音乐875、珠海电台先锋951、鹤山电台104.7 | 12 | 补广州、深圳、珠海、梅州、鹤山等地市台 |
| 广西 | `3250` | 南宁交通音乐广播 FM107.4、广西私家车930 FM93.0 | 2 | 接口已 6 台，补南宁交通音乐 + 广西私家车930 |
| 海南 | `3241` | 海南国际旅游之声 FM103.8、海南民生广播 FM101 | 2 | 接口已 3 台，补国际旅游之声 + 民生广播 |
| 重庆 | `6836` | 万盛旅游交通广播 FM92.2 | 1 | 接口已 6 台，补万盛旅游交通 |
| 四川 | `3242` | 广安交通旅游广播 | 1 | 接口已 4 台，补广安交通旅游 |
| 贵州 | `3243` | 贵州电台交通广播 FM95.2、黔西南金州之声 FM107.9 | 2 | 接口已 6 台（无交通台），补贵州交通 + 黔西南金州之声 |
| 云南 | `3251` | 云南国际广播（C:wxynsegb）、保山综合广播 FM98.7、昆明汽车广播 FM95.4、昆明资讯频率、昆明阳光广播 | 5 | 补昆明 4 个频率 + 保山 + 云南国际（C） |
| 西藏 | `6843` | 拉萨人民广播电台 FM91.4 | 1 | 接口已 5 台，补拉萨人民广播电台 |
| 陕西 | `3247` | 咸阳城市之声 FM100.7、西安交通广播 FM104.3、西安新闻广播 FM95.0、西安音乐广播 FM93.1 | 4 | 补西安、咸阳地市台 |
| 青海 | `3253` | 西宁交通频率、西宁新闻频率 | 2 | 接口已 4 台，补西宁交通/新闻 |
| 宁夏 | `3252` | 宁夏交通广播 FM98.4 | 1 | 接口只有新闻/音乐，补宁夏交通广播 |

> 另有 4 个省级台走央广 CDN（表中 `C` 列）：第一财经广播 `wx32dycjgb`（上海）、楚天交通广播 `wx32hubctjtgb`（湖北）、山东经典音乐广播 `wxsdshxxgb`（山东）、云南国际广播 `wxynsegb`（云南）。

> 已排除的同台别名（与接口是同一频率的别称，避免重复）：龙广交通/新闻广播、浙江动听968/财富广播、广东城市之声SunFM、贵州新闻综合、青海经济、新疆蒙古语、广西北部湾之声/女主播/新闻910/音乐台。

---

## 10. 快速核验命令

```bash
# 1) 主列表
curl -sk -m 12 -o /dev/null -w '%{http_code}\n' '<URL>'
# 2) 看是否有加密、是否有子列表、seq 起始值
curl -sk -m 12 '<URL>' | head -20
# 3) 8 秒后重取，比对 #EXT-X-MEDIA-SEQUENCE 是否滚动
curl -sk -m 12 '<URL>' | grep -E 'MEDIA-SEQUENCE|EXT-X-KEY'
```

带签名的流不要直接用上表裸 URL 判定，先取签名：

```bash
# 央广国家台：拿到 19 套带签名 URL
curl -sk -X POST 'https://apppc.cnr.cn/national' -H 'Content-Type: application/json' -d '{}'
# 南昌广电：抓 5 条 data-url
curl -sk 'https://www.nctv.net.cn/live?channel=2' | grep -o 'data-url="[^"]*"'
```

本仓库另附的 Python 核验脚本思路见第 0 节；活性检测务必隔 8 秒取双快照，
仅凭 HTTP 200 会把 CDN 静态缓存壳误判为可用。
