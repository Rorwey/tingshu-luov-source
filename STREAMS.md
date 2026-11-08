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

## 9. 全国广播覆盖（v11：950 台 / 31 个省级行政区 / 251 个地市）`[已接入]`

v11 换了数据来源，不再靠第三方清单东拼西凑，直接用蜻蜓 FM 的区域频道目录：

```text
GET https://rapi.qingting.fm/categories/{分类id}/channels?page=N
# 13 个分类：资讯433 音乐442 交通429 经济439 文艺432 都市441 体育430
#            双语431 综合440 生活438 旅游435 曲艺436 方言434
```

每页固定返回 20 条（`limit` 参数无效），翻到空为止；每条自带 `region`（省）与 `city`（市），
所以能按「省 · 市」精确归位，不必靠台名猜省份。共抓到 **1079 个频道**。

直播地址有两套形态，都免签名、不带会过期的参数：

| 频道 id 段 | 直播地址 | 说明 |
| --- | --- | --- |
| 老频道（3~7 位，如 `336`） | `https://ls.qingting.fm/live/{id}.m3u8` | 直接可播 |
| 2024 年后新建（`20xxx` 段） | `https://ls.qingting.fm/live/{id}/64k.m3u8` | 路径多一层 `/64k`，数据表里标识写作 `20198/64k` |

筛选漏斗：

| 环节 | 数量 |
| --- | --- |
| 分类目录频道 | 1079 |
| 排除「网络台」（无省市归属） | -46 |
| 旧 id 补漏（清单里有、分类目录没收录） | +130 候选 |
| 三道门通过（主列表 200 → 分片/子列表 200 → 8 秒 seq 增长） | 1021 + 52 |
| 与央广 `local` 接口同台 | -93 |
| 与湖南/江西硬编码台重复 | -9 |
| 同台 / 同城同频重复 | -30 |
| **入表合计** | **950**（蜻蜓 941 + 央广 CDN 9） |

逐省统计：

| 省/区 | 台数 | 地市数 | 覆盖地市（前 8） |
| --- | --- | --- | --- |
| 北京 | 10 | 1 | 北京市 |
| 天津 | 2 | 1 | 天津市 |
| 河北 | 70 | 11 | 保定市、唐山市、廊坊市、张家口市、承德市、沧州市、石家庄市、秦皇岛市… |
| 山西 | 36 | 7 | 吕梁市、大同市、太原市、晋城市、运城市、长治市、阳泉市 |
| 内蒙古 | 25 | 7 | 乌海市、包头市、呼和浩特市、巴彦淖尔市、赤峰市、鄂尔多斯市、阿拉善盟 |
| 辽宁 | 29 | 9 | 丹东市、大连市、朝阳市、本溪市、沈阳市、葫芦岛市、辽阳市、锦州市… |
| 吉林 | 29 | 7 | 吉林市、四平市、延边朝鲜族自治州、松原市、白山市、通化市、长春市 |
| 黑龙江 | 19 | 5 | 哈尔滨市、大兴安岭地区、大庆市、牡丹江市、鸡西市 |
| 上海 | 11 | 1 | 上海市 |
| 江苏 | 76 | 13 | 南京市、南通市、宿迁市、常州市、徐州市、扬州市、无锡市、淮安市… |
| 浙江 | 64 | 10 | 台州市、嘉兴市、宁波市、杭州市、温州市、湖州市、绍兴市、舟山市… |
| 安徽 | 38 | 13 | 亳州市、合肥市、安庆市、宣城市、宿州市、池州市、淮北市、滁州市… |
| 福建 | 27 | 7 | 三明市、南平市、厦门市、泉州市、漳州市、福州市、龙岩市 |
| 江西 | 16 | 8 | 九江市、南昌市、抚州市、新余市、景德镇市、萍乡市、赣州市、鹰潭市 |
| 山东 | 92 | 15 | 东营市、临沂市、威海市、德州市、日照市、枣庄市、济南市、济宁市… |
| 河南 | 68 | 20 | 三门峡市、信阳市、南阳市、周口市、商丘市、安阳市、巩义、平顶山市… |
| 湖北 | 43 | 13 | 十堰市、咸宁市、孝感市、宜昌市、恩施土家族苗族自治州、武汉市、荆州市、荆门市… |
| 湖南 | 36 | 10 | 娄底市、岳阳市、常德市、怀化市、湘潭市、益阳市、衡阳市、邵阳市… |
| 广东 | 52 | 18 | 中山市、云浮市、广州市、惠州市、揭阳市、梅州市、汕头市、江门市… |
| 广西 | 18 | 8 | 北海市、南宁市、柳州市、桂林市、玉林市、贵港市、贺州市、钦州市 |
| 海南 | 11 | 3 | 三亚市、海口市、琼海 |
| 重庆 | 11 | 1 | 重庆市 |
| 四川 | 61 | 19 | 乐山市、内江市、凉山彝族自治州、南充市、宜宾市、巴中市、广元市、广安市… |
| 贵州 | 19 | 8 | 六盘水市、安顺市、毕节地区、贵阳市、遵义市、铜仁地区、黔东南苗族侗族自治州、黔西南布依族苗族自治州 |
| 云南 | 27 | 12 | 保山市、大理白族自治州、德宏傣族景颇族自治州、怒江傈僳族自治州、思茅市、文山壮族苗族自治州、昆明市、昭通市… |
| 西藏 | 2 | 1 | 拉萨市 |
| 陕西 | 12 | 5 | 咸阳市、安康市、宝鸡市、渭南市、西安市 |
| 甘肃 | 4 | 4 | 天水市、定西市、张掖市、酒泉市 |
| 青海 | 4 | 1 | 西宁市 |
| 宁夏 | 2 | 2 | 石嘴山市、银川市 |
| 新疆 | 36 | 11 | 乌鲁木齐市、伊犁哈萨克自治州、博尔塔拉蒙古自治州、哈密地区、图木舒克市、塔城地区、巴音郭楞蒙古自治州、昌吉回族自治州… |

几个要点：

- **天津**：央广 `local/list` 只列 30 个省、根本没有天津，所以单开一个「天津」分类（id `tj`），内容全走数据表。
- **台名规则**：台名里没有城市名就补 `城市·` 前缀（「保定·曲阳融媒FM90.4」）；自治州按民族名截断（「红河哈尼族彝族自治州」→「红河」）；同城同频只留一条。
- **每台独立台标**：优先用蜻蜓 `pic.qtfm.cn` 的频道封面（约 900 张），缺失回退省默认图。
- **数据源短板**：蜻蜓对西藏（1 个频道）、宁夏（3）、青海（4）、甘肃（7）、天津（2）收录本身很少，
  这几地台数少是源的限制，不是筛选过严。
- **数据表分块**：整表约 100KB，超过 JVM 单字符串常量 65535 字节上限（编译报 `UTF8 string too large`），
  `RadioData.kt` 已按 52KB 拆成 `C1`/`C2`，由 `TABLE` 在运行时合并。

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

---

## 11. 电视直播源（v12：央视 + 全国各省卫视伴音）`[已接入]`

电视源「LocalTV」在 v12 重构为与广播源同构的分级：菜单首项「央视」，其后 31 个省级行政区（含天津，与广播同序）。

### 11.1 全国各省卫视伴音（央广 CDN 卫星电视纯音频版）

来源：公开 IPTV 清单 `guozu616/iptv` 的「电视伴音」表。本质是各省级卫视的**纯音频版**，
走央广 CDN `https://satellitepull.cnr.cn/live/{code}/playlist.m3u8`（与广播源同域名，免签名）。
master 列表返回 `#EXT-X-STREAM-INF`（BANDWIDTH ~220k，确认纯音频），其子列表经 `100ycdn.com`
下发并带 `wsSession/wsIPSercert` 动态签名；播放器先取 master 再取带签名子列表，链路与已上线的湖南/江西卫视伴音一致。

v12 实测（2026-11-08，本开发环境）：**31 个省级卫视伴音 code 全部 HTTP 200 + 合法 m3u8**；
抽 8 省（北京/内蒙古/江苏/湖北/广东/四川/新疆/西藏）做 8 秒双快照，`#EXT-X-MEDIA-SEQUENCE` 稳定 +1（真·直播）。

| 省/市 | 卫视伴音（名称标「（伴音）」） | code | 备注 |
| --- | --- | --- | --- |
| 北京 | 北京卫视 | `wxbtv` | |
| 天津 | 天津卫视 | `wxtjws` | |
| 河北 | 河北卫视 | `wxhebws` | |
| 山西 | 山西卫视 | `wxssxws` | |
| 内蒙古 | 内蒙古卫视 | `wx32nmgws` | |
| 辽宁 | 辽宁卫视 | `wxlnws` | |
| 吉林 | 吉林卫视 | `wxjlws` | 另含城市卫视「延边卫视 `wxybws`」 |
| 黑龙江 | 黑龙江卫视 | `wx32hljws` | |
| 上海 | 东方卫视 | `wx32dfws` | 上海无本省卫视，由东方卫视覆盖 |
| 江苏 | 江苏卫视 | `wx32jsws` | |
| 浙江 | 浙江卫视 | `wxzjws` | |
| 安徽 | 安徽卫视 | `wxahws` | |
| 福建 | 东南卫视 | `wx32fjws` | 福建无本省卫视，由东南卫视覆盖 |
| 江西 | 江西卫视 | `wx32jiangxws` | 另含南昌广电 3 个官方视频流（见 11.3） |
| 山东 | 山东卫视 | `wxsdws` | |
| 河南 | 河南卫视 | `wxhnws` | |
| 湖北 | 湖北卫视 | `wx32hubws` | |
| 湖南 | 湖南卫视 | `wx32hunws` | |
| 广东 | 广东卫视 | `wxgdws` | 另含城市卫视「深圳卫视 `wxszws`」 |
| 海南 | 海南卫视 | `wxhainlyws` | |
| 重庆 | 重庆卫视 | `wxcqws` | |
| 四川 | 四川卫视 | `wxscws` | |
| 贵州 | 贵州卫视 | `wx32gzws` | |
| 西藏 | 西藏卫视 | `wxxzws` | |
| 陕西 | 陕西卫视 | `wxsxxws` | |
| 甘肃 | 甘肃卫视 | `wxgsws` | |
| 青海 | 青海卫视 | `wx32qhws` | |
| 宁夏 | 宁夏卫视 | `wxnxws` | |
| 新疆 | 新疆卫视 | `wxxjws` | |
| 广西 | （无） | — | 卫视伴音 CDN 上查无对应 code，tab 暂空 |
| 云南 | （无） | — | 卫视伴音 CDN 上查无对应 code，tab 暂空 |

封面：卫视伴音复用广播源各省已验证 200 的台标（`RadioData.provinceCovers`）作兜底；
南昌视频流用官网 `nctv.net.cn/images/ngxw*.jpg`。

### 11.2 央视分类（CCTV 音频版，名称标「⚠（未验证）」）

央视官网 `tv.cctv.com` 视频链路为 **ChinaDRM 加密**（ExoPlayer 无法直解），在播视频流不可接。
公开清单里存在 CCTV1–17、CCTV新科动漫、中国教育 CETV1–4 的**音频版**
（`myalicdn.com` / `wscdns.com` 的 `BR=audio` / `audio.m3u8`）。

**重要限制**：这些音频 CDN（cctv.v.myalicdn.com、cctv5cncc.v.wscdns.com、wstvalic.v.myalicdn.com）
在**本开发环境 DNS 解析失败、无法连通**，故 v12 仅按用户选择「收录并标未验证」——
**在你国内真机上可能可播，不可用请直接在 App 内删除对应条目**。央视音频 CDN 不可达属网络环境限制，非代码问题。

### 11.3 江西省内官方视频流（不标注「伴音」）

南昌广播电视台 3 个频道来自官网 `https://www.nctv.net.cn/live?channel=2` 的 data-url：

| 名称 | URL（裸，播放时由 StreamSigner 注入 txSecret/txTime） |
| --- | --- |
| 南昌新闻综合频道 | `https://live.nctv.net.cn/live/nctv1.m3u8` |
| 南昌文旅频道 | `https://live.nctv.net.cn/live/nctv2.m3u8` |
| 南昌资讯频道 | `https://live.nctv.net.cn/live/nctv3.m3u8` |

裸地址 403，播放时由 `StreamSigner.resolve()` 反查官网注入最新签名（约 30 分钟滚动）。
视频流在纯音频 App 中只取声音；若个别机型无法解码，可改用官网 `only-audio=1` 伴音版。

### 11.4 为什么省级官网真视频流不接入

v7/v9 已实测：江西广电官网电视鉴权链虽逆出，但播放 CDN（网宿 `MC_VCLOUD_LIVE`）一律 403，
按开源算法（`linux-jin/TvBox`、恩山论坛）复现 36 种组合仍 403；湖南 vzan、多数省台官网视频流同样
403 / 需 token。故电视以「卫视伴音（纯音频）」为唯一可稳定播的形态。

