# 《我的听书》订阅源合集

因为《我的听书》APP 已停止维护。本仓库汇总多个《我的听书》自定义源 jar 包与对应的订阅入口 `external_sources.json`，方便一键订阅导入或离线使用。仅供自用。

**导入方式（订阅）**：在 App 的「自定义源管理 → 右上角添加订阅」中，填入下方对应源的 raw 地址即可。

**导入方式（离线）**：把 jar 直接复制到手机 `Android/data/com.github.eprendre.tingshu/files/jars/` 目录，App 启动时会自动加载（无需 json）。

---

## 一、本仓库唯一自维护源：luov（B 站 + 广播 / 电视直播）

`sources_by_luov.jar` 是**本仓库唯一新编写、长期维护**的源，相对 B 站官方内容做了**兼容性保留 + 功能新增**：

- **订阅标题**：本订阅在 App「源管理」页显示的标题为 **「订阅源：听书+直播源」**
- **哔哩（新）（B 站子源）** —— App 源列表中的显示名即为「哔哩（新）」。
  - 保留并兼容原有能力：搜索、分类（6 个标签）、合集视频连播；
  - 相比于旧的官方源，**新增「合集」连播**：支持 B 站 `ugc_season` 合集（原 App 只能抓单集、分P，无法合集连播），自动展开合集内每一集、以及每集合的每个分 P，生成可连续播放的条目。

> ⚠️ **重要：请勿同时启用原作者的官方「哔哩哔哩」源。** 本「哔哩（新）」是官方旧源的**超集**（旧功能全保留 + 新增合集连播），且两者使用完全相同的 B 站搜索接口与域名。若同时启用，App 在全局搜索时会对同一关键词向 B 站发两条重复请求，触发风控限流 / 结果归属竞态，表现为「其中一个哔哩源搜索空白、时好时坏」。请在本仓库导入的 `sources_by_eprendre` 等第三方源中，**关闭或删除官方旧「哔哩哔哩」源**，仅保留本「哔哩（新）」即可。

- **央广 / 地方广播**：国家广播 19 套（央广 16 套 + 国广 CRI 3 套）+ 地方广播 31 个省级行政区（含央广接口没有的天津），统一收进同一个源——分类菜单中「央广」为第一个分类，其后依次各省/区/直辖市，共 31 个分类；每个台都带自己的台标。并新增了地市覆盖。


导入地址：

```text
https://raw.githubusercontent.com/Rorwey/tingshu-luov-source/main/external_sources.json
```

文件说明：

- `external_sources.json`：luov 源的订阅入口（当前 version = 11，版本号变化会触发 App 重新拉取）。
- `sources_by_luov.jar`：luov 源解析包。
- `STREAMS.md`：**直播源与封面清单 / 核验手册**——所有已接入流的 URL、封面图、签名方式、活性检测结果，以及备选（实测可用但未接入）与已排除流的清单和原因、可复制的核验命令。

> 参考与致谢：本 luov 源基于原作者 **eprendre** 的《我的听书》框架与示例编写（自定义源项目 `CustomSources`、源编写逻辑均来源于此），借助AI协助。参考项目地址：https://github.com/eprendre/tingshu 。本仓库仅在此框架基础上新增合集连播与广播 / 电视直播能力，著作权尊重原框架作者。
>
> 广播 / 电视直链另参考了 GitHub 公开电台清单（`kilvn/iptv`、`hcm82/iptvradio`、`jiqiao/tv`、`iptv-org/iptv`、`fanmingming/live` 等）、江西广电鉴权的公开实现（`linux-jin/TvBox` 江西电视台 PC 官网代理源码、恩山无线论坛相关讨论）与各地广电官网（`www.cnr.cn/gbzb/`、`jxntv.cn`、`www.jxgdw.cn`、`www.nctv.net.cn`）。所有收录的流均**逐条实测**：主列表 → 子列表 → ts 分片 HTTP 200、无 `#EXT-X-KEY` 加密、播放列表 8 秒双快照实时滚动；静态缓存（如湘潭广电）、伪装成 200 的错误响应、停播 / 404 流一律排除。需要鉴权的官方流不硬编码签名，改为播放时动态获取。完整清单见 `STREAMS.md`。

---

## 二、其余三个第三方源（原作者已停止维护）

以下三个 jar 为**第三方 / 原作者**提供，**本仓库仅收录备份**，其原始地址如下：

| 源 | 说明 | 原始地址（不一定是官方）  |
| --- | --- | --- |
| ting29 | 听书网合集（29听书 / 恋听 / 乐听 / 麒麟 / 275 / 有听 / 听13 / 爱听书） | https://github.com/liaozhu913/tingshu-ting29-source |
| eprendre | 原作者 eprendre 维护的多站听书 / 电台合集（哔哩哔哩、懒人听书、酷我、CCTV 等 33 个源） | http://ywbk.cc/306.html |
| shun | 原作者生态下的第三方整合听书源 | http://ywbk.cc/306.html |

对应订阅入口（**在 App「自定义源管理 → 添加订阅」中直接填入下方 raw 地址即可**）：

| 源 | 订阅地址（复制到 App 订阅框） | 本地文件（jar + json） |
| --- | --- | --- |
| ting29 | `https://raw.githubusercontent.com/Rorwey/tingshu-luov-source/main/external_sources_ting29.json` | `sources_by_ting29.jar` + `external_sources_ting29.json` |
| eprendre | `https://raw.githubusercontent.com/Rorwey/tingshu-luov-source/main/external_sources_eprendre.json` | `sources_by_eprendre.jar` + `external_sources_eprendre.json` |
| shun | `https://raw.githubusercontent.com/Rorwey/tingshu-luov-source/main/external_sources_shun.json` | `sources_by_shun.jar` + `external_sources_shun.json` |

> 说明：eprendre / shun 原作者项目已**停止维护**（源码大部分失效），本仓库仅作存档与离线备用；ting29 仍由其作者维护，本仓库仅做备份。

---

## 三、App 获取与参考来源

本仓库的源需配合《我的听书》App（包名 `com.github.eprendre.tingshu`）使用。本仓库所使用的 App 安装包获取自：https://pan.lanzoux.com/b873905 （官方托管地址）。

自定义源编写所参考的原作者框架与示例，见第一节「参考与致谢」：https://github.com/eprendre/tingshu 。

---

## 四、免责申明

1. 本仓库所有内容**仅供个人学习、研究与非商业的自用目的**，请勿用于任何商业用途或大规模分发。
2. 第三方 jar 包（ting29 / eprendre / shun）的著作权归**各自原作者**所有，本仓库仅做备份收录与离线导入便利，**不主张任何权利**，亦不承担其维护责任；如原作者要求移除，将立即下架。
3. 本仓库自维护的 luov 源所聚合的媒体内容（哔哩哔哩视频，中央广播电视总台 / 央广 / 国广、江西 / 湖南 / 南昌等各地广播电视台的直播与伴音流）其**版权均归原作者 / 原始平台（B 站、中央广播电视总台及各地广电等）所有**；本源仅提供内容索引与播放链路解析，**不存储、不转码、不重新分发任何受版权保护的内容**，亦不构成对任一平台版权的侵犯。
4. 使用本仓库任何源即表示你已知晓并同意上述条款；因使用本仓库内容产生的任何后果由使用者自行承担。
