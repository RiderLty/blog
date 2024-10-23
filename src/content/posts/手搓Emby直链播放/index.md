---
title: 手搓Emby直链播放
published: 2024-10-21
description: 'emby + nginx 实现直链播放'
image: './cover.jpg'
tags: [emby,nginx,njs,nas]
category: 'NAS'
draft: false 
lang: ''
---

组了台NAS，但是说是网络附加存储，其实小型服务器更准确些。

只配了一块2T的固态用于存储系统数据，电影番剧都是存网盘，毕竟一块24T的红盘都够开15年网盘会员的了。

而且在外使用的时候，家宽的小水管上传根本不够看4K的。

所以这里只要手搓一点点代码就可以实现直链播放。

## 原理部分

* 媒体文件都存储在网盘，本地使用strm文件内存储对应的媒体文件的URL。

* strm文件内的媒体文件URL可以被Emby正常访问。

* nginx反向代理Emby服务器，添加strm地址转换接口，并使用njs处理视频接口。

* 当访问URL时，通过njs获取真实地址并重定向。

## 需要注意的点

* 在Emby的网络设置中，将局域网IP与端口改为nginx反向代理后的地址与端口，否则会自动跳转原始服务器。

* 如果不能直链播放，请检查是否正在播放转码后的视频。

## 实现过程

```emby.conf```
```nginx
js_shared_dict_zone zone=jsEmbyCacheDict:10M timeout=2h evict;
server {
    set $EMBY_SERVER_URL "http://192.168.3.3:7096";
    js_import emby from /etc/nginx/conf.d/emby.js;
    listen 80;
    listen 8096;
    server_name emby.nas.local;
    js_fetch_verify off;

    location ~ /(socket|embywebsocket) {
        proxy_pass $EMBY_SERVER_URL;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Protocol $scheme;
        proxy_set_header X-Forwarded-Host $http_host;
    }

    location ~ ^(.*)/proxy(/.*)$ {
        internal;
        rewrite ^(.*)/proxy(/.*)$ $1$2 break;
        proxy_pass $EMBY_SERVER_URL;
    }

    location ~ ^(.*)/strm(/.*)$ {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        js_content emby.strmHandler;
    }

    location ~* /videos/(.*)/Subtitles {
        js_content emby.apiHandler;
    }

    location ~* /videos/(.*)/(stream|original) {
        js_content emby.apiHandler;
    }

    location ~* /Items/([^/]+)/Download {
        js_content emby.apiHandler;
    }

    location ~* /Sync/JobItems/(.*)/File {
        js_content emby.apiHandler;
    }

    location / {
        proxy_pass $EMBY_SERVER_URL;
    }
}
```
nginx的配置文件，添加了一个```/strm/*```路由，用于处理strm文件的链接

视频文件将会由njs的apiHandler函数处理

```emby.js```
```javascript
import qs from 'querystring';

const debug = (msg) => {
    ngx.log(ngx.ERR, `DEBUG:${msg}\n`);
}

const EMBY_SERVER_URL = "http://192.168.3.3:7096"
const EMBY_API_KEY = "34fb345c***********8ad9390b"
const WBEDAV_AUTHORIZATION = "Basic MTU***********************************mM3Y="
const URL_HEADER = "http://emby.nas.local/strm/"
const ALIST_URL = "http://alist.nas.local/d/"

const cacheGet = (key) => {
    return ngx.shared.jsEmbyCacheDict.get(key);
}

const cacheSet = async (key, value) => {
    if (!key || !value) {
        debug("!key || !value  RETURN");
        return;
    }
    const dict = ngx.shared["jsEmbyCacheDict"];
    dict.delete(key)
    // dict.add(key, value , 1000);
    dict.add(key, value , 1000 * 60 * 2);
}

const get123PanDirectUrl = async (path) => {
    const pan123Path = `https://webdav-18********59.pd1.123pan.cn/webdav${path}`
    debug(`123pan获取URL中 : ${path}\n ${encodeURI(pan123Path)}`)
    const resp = await ngx.fetch(encodeURI(pan123Path), {
        headers: {
            Authorization: WBEDAV_AUTHORIZATION,
            Range: 'bytes=0-0'
        }
    })
    debug(`123pan获取URL成功 ${path} ==302==> ${resp.headers.Location}`)
    return resp.headers.Location
}

const getAlistDirectUrl = async (path) => {
    const alistUrl = `${ALIST_URL}${path}`
    debug(`alist获取URL中 ${path} ${encodeURI(alistUrl)}`)
    const resp = await ngx.fetch(encodeURI(alistUrl), {
        headers: {
            Range: 'bytes=0-0'
        }
    })
    debug(`alist获取URL成功 ${path} ==302==> ${resp.headers.Location}`)
    return resp.headers.Location
}

const strm2realUrl = async (relativeUrl) => {
    // relativeUrl , 完整的STRM地址去掉URL_HEADER的部分
    // 例如 /123pan/asda/adasd/asda , /xiaoya/asdasd/asdasd/asd
    debug(`strm转真实url:${relativeUrl}`)
    const cachedUrl = cacheGet(relativeUrl)
    if (!cachedUrl) {
        debug(`缓存MISS !`)
        if (relativeUrl.indexOf("/123pan/") === 0) {
            const url = await get123PanDirectUrl(relativeUrl.slice(7))
            cacheSet(relativeUrl, url)
            return url
        } else if (relativeUrl.indexOf("/alist/") === 0) {
            const url = await getXiaoYaDirectUrl(relativeUrl.slice(7))
            cacheSet(relativeUrl, url)
            return url
        } else {
            return null
        }
    } else {
        debug(`缓存命中 ${relativeUrl} ==302==> ${cachedUrl}`)
        return cachedUrl
    }
}

const strmHandler = async (r) => {
    const url = await strm2realUrl(r.uri.replace("/strm/", "/"))
    r.return(302, url)
}


const apiHandler = async (r) => {
    const proxy = `/proxy${r.uri}?1=1&${qs.stringify(r.args)}`
    let match = r.uri.toLowerCase().match(/videos\/(.*)\/(stream|original)/)
    if (match === null) {
        match = r.uri.toLowerCase().match(/items\/([^/]+)\/download/)
    }
    if (match === null) {
        match = r.uri.toLowerCase().match(/sync\/jobitems\/(.*)\/file/)
    }
    if (match === null) {
        debug(`match == null [${r.uri}]`)
        r.internalRedirect(proxy)
    } else {
        const itemInfo = await (await ngx.fetch(`${EMBY_SERVER_URL}/Items?Ids=${match[1]}&Fields=Path,MediaSources&Limit=1&api_key=${EMBY_API_KEY}`)).json()
        const media = itemInfo["Items"][0]["MediaSources"][0]
        if (media["Protocol"] == "Http" && media["Path"].indexOf(URL_HEADER) === 0) {
            const url = await strm2realUrl(media["Path"].replace(URL_HEADER, "/"))
            r.return(302, url)
        } else {
            debug(`本地文件 ${media["Path"]}`)
            r.internalRedirect(proxy)
        }
    }

}

export default { apiHandler, strmHandler };
```
njs文件，实现了具体的功能。

```apiHandler```接管请求，向emby服务器请求信息，如果是视频且类型为http，则将strm文件内链接获取到302的地址返回，否则使用内部路由```/proxy/```来反向代理原始服务器。

以一个strm文件为例：

```Predestination.2014.Bluray.1080p.DTS-HD.x264-Grym.strm```

在emby详情页选择下载，原始URL为：

```http://emby.nas.local/emby/Items/22176/Download?api_key=34fb345c***********8ad9390b&mediaSourceId=mediasource_22176```

```apiHandler```请求emby的Items接口：

```http://emby.nas.local/Items?Ids=22176&Fields=Path,MediaSources&Limit=1&api_key=34fb345c***********8ad9390b```

取得以下数据：

```json
{
    "Items": [
        {
            "Name": "前目的地",
            "ServerId": "c065f00f3a8844d6b86b5becf9531624",
            "Id": "22176",
            "Container": "mkv",
            "MediaSources": [
                {
                    "Chapters": [],
                    "Protocol": "Http",
                    "Id": "mediasource_22176",
                    "Path": "http://emby.nas.local/strm/123pan/电影/Predestination.2014.Bluray.1080p.DTS-HD.x264-Grym/Predestination.2014.Bluray.1080p.DTS-HD.x264-Grym.mkv",
                    "Type": "Default",
                    "Container": "mkv",
                    "Size": 11989856916,
                    "Name": "前目的地",
                    "IsRemote": true,
                    "HasMixedProtocols": false,
                    "RunTimeTicks": 58688650000,
                    "SupportsTranscoding": true,
                    "SupportsDirectStream": true,
                    "SupportsDirectPlay": true,
                    "IsInfiniteStream": false,
                    "RequiresOpening": false,
                    "RequiresClosing": false,
                    "RequiresLooping": false,
                    "SupportsProbing": false,
                    "MediaStreams": [
                        {
                            "Codec": "h264",
                            "Language": "eng",
                            "TimeBase": "1/1000",
                            "VideoRange": "SDR",
                            "DisplayTitle": "1080p H264",
                            "DisplayLanguage": "English",
                            "NalLengthSize": "4",
                            "IsInterlaced": false,
                            "BitRate": 16343680,
                            "BitDepth": 8,
                            "RefFrames": 1,
                            "IsDefault": true,
                            "IsForced": false,
                            "IsHearingImpaired": false,
                            "Height": 1080,
                            "Width": 1920,
                            "AverageFrameRate": 23.976025,
                            "RealFrameRate": 23.976025,
                            "Profile": "High",
                            "Type": "Video",
                            "AspectRatio": "16:9",
                            "Index": 0,
                            "IsExternal": false,
                            "IsTextSubtitleStream": false,
                            "SupportsExternalStream": false,
                            "Protocol": "File",
                            "PixelFormat": "yuv420p",
                            "Level": 41,
                            "IsAnamorphic": false,
                            "ExtendedVideoType": "None",
                            "ExtendedVideoSubType": "None",
                            "ExtendedVideoSubTypeDescription": "None",
                            "AttachmentSize": 0
                        },
                        {
                            "Codec": "dts",
                            "Language": "eng",
                            "TimeBase": "1/1000",
                            "DisplayTitle": "English DTS-HD MA 5.1 (默认)",
                            "DisplayLanguage": "English",
                            "IsInterlaced": false,
                            "ChannelLayout": "5.1",
                            "BitDepth": 16,
                            "Channels": 6,
                            "SampleRate": 48000,
                            "IsDefault": true,
                            "IsForced": false,
                            "IsHearingImpaired": false,
                            "Profile": "DTS-HD MA",
                            "Type": "Audio",
                            "Index": 1,
                            "IsExternal": false,
                            "IsTextSubtitleStream": false,
                            "SupportsExternalStream": false,
                            "Protocol": "File",
                            "ExtendedVideoType": "None",
                            "ExtendedVideoSubType": "None",
                            "ExtendedVideoSubTypeDescription": "None",
                            "AttachmentSize": 0
                        },
                        {
                            "Codec": "ass",
                            "Language": "zh-CN",
                            "DisplayTitle": "Chinese Simplified (ASS)",
                            "DisplayLanguage": "Chinese Simplified",
                            "IsInterlaced": false,
                            "IsDefault": false,
                            "IsForced": false,
                            "IsHearingImpaired": false,
                            "Type": "Subtitle",
                            "Index": 21,
                            "IsExternal": true,
                            "IsTextSubtitleStream": true,
                            "SupportsExternalStream": true,
                            "Path": "/mnt/user/storage/Media/EmbyMedia/123pan/电影/Predestination.2014.Bluray.1080p.DTS-HD.x264-Grym/Predestination.2014.Bluray.1080p.DTS-HD.x264-Grym.chs.ass",
                            "Protocol": "File",
                            "ExtendedVideoType": "None",
                            "ExtendedVideoSubType": "None",
                            "ExtendedVideoSubTypeDescription": "None",
                            "AttachmentSize": 0
                        }
                    ],
                    "Formats": [],
                    "Bitrate": 16343680,
                    "RequiredHttpHeaders": {},
                    "AddApiKeyToDirectStreamUrl": false,
                    "ReadAtNativeFramerate": false,
                    "ItemId": "22176"
                }
            ],
            "Path": "/mnt/user/storage/Media/EmbyMedia/123pan/电影/Predestination.2014.Bluray.1080p.DTS-HD.x264-Grym/Predestination.2014.Bluray.1080p.DTS-HD.x264-Grym.strm",
            "RunTimeTicks": 58688650000,
            "Size": 11989856916,
            "Bitrate": 16343680,
            "IsFolder": false,
            "Type": "Movie",
            "ImageTags": {
                "Primary": "00db47f51b81c2680f1e3d7986faf9e6",
                "Logo": "2b74dcda7c3ad35227937fa0c2d7ec68",
                "Art": "69818ff598c06b17eb6acff1a23ef787",
                "Banner": "6f36fa4b027ffd4d10b9bcf2acd39370",
                "Thumb": "bdc5c80bcd149d5a2d9867cb3903b4f6"
            },
            "BackdropImageTags": [
                "cacd9ee28056210e53b46016fc895370"
            ],
            "MediaType": "Video"
        }
    ],
    "TotalRecordCount": 1
}
```
Protocol等于Http，则Path是strm文件内的url。

去除```URL_HEADER```部分，根据开头判断这是一个123盘的文件链接，绝对地址为：

```/电影/Predestination.2014.Bluray.1080p.DTS-HD.x264-Grym/Predestination.2014.Bluray.1080p.DTS-HD.x264-Grym.mkv```

使用```get123PanDirectUrl```函数发送请求，在headers中添加```Range: 'bytes=0-0'```来获取302后的地址。

将获取到的地址使用```r.return(302, url)```返回，客户端在点击下载时，会跳转直链。alist同理。

自此，通过nginx访问反向代理后的emby，即可实现直链播放。

## 扩展

[创建strm映射的脚本](https://github.com/RiderLty/rcloneStrm/blob/main/createStrm.py)

::github{repo="RiderLty/rcloneStrm"}