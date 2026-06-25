# Userscripts

## Siempre Activadas

Para instalar Userscripts:  
[ViolentMonkey](https://addons.mozilla.org/en-US/firefox/addon/violentmonkey/)<br><br>

Pagetual:  
[Website](https://pagetual.hoothin.com/)
[Greasyfork](https://greasyfork.org/en/scripts/438684-pagetual)
[Github](https://github.com/hoothin/UserScripts)<br><br>

4chan X:  
[Greasyfork](https://greasyfork.org/es/scripts/7750-4chan-x)
[Github](https://github.com/ccd0/4chan-x/)<br><br>

TwitchAdSolutions:  
[Github](https://github.com/pixeltris/TwitchAdSolutions)
<details>
<summary>
Archive (37.0.0)
</summary>

```js
// ==UserScript==
// @name         TwitchAdSolutions (vaft)
// @namespace    https://github.com/pixeltris/TwitchAdSolutions
// @version      37.0.0
// @description  Multiple solutions for blocking Twitch ads (vaft)
// @updateURL    https://github.com/pixeltris/TwitchAdSolutions/raw/master/vaft/vaft.user.js
// @downloadURL  https://github.com/pixeltris/TwitchAdSolutions/raw/master/vaft/vaft.user.js
// @author       https://github.com/cleanlock/VideoAdBlockForTwitch#credits
// @match        *://*.twitch.tv/*
// @run-at       document-start
// @inject-into  page
// @grant        none
// ==/UserScript==
(function() {
    'use strict';
    const ourTwitchAdSolutionsVersion = 24;// Used to prevent conflicts with outdated versions of the scripts
    if (typeof window.twitchAdSolutionsVersion !== 'undefined' && window.twitchAdSolutionsVersion >= ourTwitchAdSolutionsVersion) {
        console.log("skipping vaft as there's another script active. ourVersion:" + ourTwitchAdSolutionsVersion + " activeVersion:" + window.twitchAdSolutionsVersion);
        window.twitchAdSolutionsVersion = ourTwitchAdSolutionsVersion;
        return;
    }
    window.twitchAdSolutionsVersion = ourTwitchAdSolutionsVersion;
    function declareOptions(scope) {
        scope.AdSignifier = 'stitched';
        scope.ClientID = 'kimne78kx3ncx6brgo4mv6wki5h1ko';
        scope.BackupPlayerTypes = [
            'embed',//Source
            'popout',//Source
            'autoplay',//360p
            //'picture-by-picture-CACHED'//360p (-CACHED is an internal suffix and is removed)
        ];
        scope.FallbackPlayerType = 'embed';
        scope.ForceAccessTokenPlayerType = 'popout';
        scope.SkipPlayerReloadOnHevc = false;// If true this will skip player reload on streams which have 2k/4k quality (if you enable this and you use the 2k/4k quality setting you'll get error #4000 / #3000 / spinning wheel on chrome based browsers)
        scope.AlwaysReloadPlayerOnAd = false;// Always pause/play when entering/leaving ads
        scope.ReloadPlayerAfterAd = true;// After the ad finishes do a player reload instead of pause/play
        scope.PlayerReloadMinimalRequestsTime = 1500;
        scope.PlayerReloadMinimalRequestsPlayerIndex = 2;//autoplay
        scope.HasTriggeredPlayerReload = false;
        scope.StreamInfos = [];
        scope.StreamInfosByUrl = [];
        scope.GQLDeviceID = null;
        scope.ClientVersion = null;
        scope.ClientSession = null;
        scope.ClientIntegrityHeader = null;
        scope.AuthorizationHeader = undefined;
        scope.SimulatedAdsDepth = 0;
        scope.PlayerBufferingFix = true;// If true this will pause/play the player when it gets stuck buffering
        scope.PlayerBufferingDelay = 600;// How often should we check the player state (in milliseconds)
        scope.PlayerBufferingSameStateCount = 3;// How many times of seeing the same player state until we trigger pause/play (it will only trigger it one time until the player state changes again)
        scope.PlayerBufferingDangerZone = 1;// The buffering time left (in seconds) when we should ignore the players playback position in the player state check
        scope.PlayerBufferingDoPlayerReload = false;// If true this will do a player reload instead of pause/play (player reloading is better at fixing the playback issues but it takes slightly longer)
        scope.PlayerBufferingMinRepeatDelay = 8000;// Minimum delay (in milliseconds) between each pause/play (this is to avoid over pressing pause/play when there are genuine buffering problems)
        scope.PlayerBufferingPrerollCheckEnabled = false;// Enable this if you're getting an immediate pause/play/reload as you open a stream (which is causing the stream to take longer to load). One problem with this being true is that it can cause the player to get stuck in some instances requiring the user to press pause/play
        scope.PlayerBufferingPrerollCheckOffset = 5;// How far the stream need to move before doing the buffering mitigation (depends on PlayerBufferingPrerollCheckEnabled being true)
        scope.V2API = false;
        scope.IsAdStrippingEnabled = true;
        scope.AdSegmentCache = new Map();
        scope.AllSegmentsAreAdSegments = false;
    }
    let isActivelyStrippingAds = false;
    let localStorageHookFailed = false;
    const twitchWorkers = [];
    const workerStringConflicts = [
        'twitch',
        'isVariantA'// TwitchNoSub
    ];
    const workerStringAllow = [];
    const workerStringReinsert = [
        'isVariantA',// TwitchNoSub (prior to (0.9))
        'besuper/',// TwitchNoSub (0.9)
        '${patch_url}'// TwitchNoSub (0.9.1)
    ];
    function getCleanWorker(worker) {
        let root = null;
        let parent = null;
        let proto = worker;
        while (proto) {
            const workerString = proto.toString();
            if (workerStringConflicts.some((x) => workerString.includes(x)) && !workerStringAllow.some((x) => workerString.includes(x))) {
                if (parent !== null) {
                    Object.setPrototypeOf(parent, Object.getPrototypeOf(proto));
                }
            } else {
                if (root === null) {
                    root = proto;
                }
                parent = proto;
            }
            proto = Object.getPrototypeOf(proto);
        }
        return root;
    }
    function getWorkersForReinsert(worker) {
        const result = [];
        let proto = worker;
        while (proto) {
            const workerString = proto.toString();
            if (workerStringReinsert.some((x) => workerString.includes(x))) {
                result.push(proto);
            } else {
            }
            proto = Object.getPrototypeOf(proto);
        }
        return result;
    }
    function reinsertWorkers(worker, reinsert) {
        let parent = worker;
        for (let i = 0; i < reinsert.length; i++) {
            Object.setPrototypeOf(reinsert[i], parent);
            parent = reinsert[i];
        }
        return parent;
    }
    function isValidWorker(worker) {
        const workerString = worker.toString();
        return !workerStringConflicts.some((x) => workerString.includes(x))
            || workerStringAllow.some((x) => workerString.includes(x))
            || workerStringReinsert.some((x) => workerString.includes(x));
    }
    function hookWindowWorker() {
        const reinsert = getWorkersForReinsert(window.Worker);
        const newWorker = class Worker extends getCleanWorker(window.Worker) {
            constructor(twitchBlobUrl, options) {
                let isTwitchWorker = false;
                try {
                    isTwitchWorker = new URL(twitchBlobUrl).origin.endsWith('.twitch.tv');
                } catch {}
                if (!isTwitchWorker) {
                    super(twitchBlobUrl, options);
                    return;
                }
                const newBlobStr = `
                    const pendingFetchRequests = new Map();
                    ${stripAdSegments.toString()}
                    ${getStreamUrlForResolution.toString()}
                    ${processM3U8.toString()}
                    ${hookWorkerFetch.toString()}
                    ${declareOptions.toString()}
                    ${getAccessToken.toString()}
                    ${gqlRequest.toString()}
                    ${parseAttributes.toString()}
                    ${getWasmWorkerJs.toString()}
                    ${getServerTimeFromM3u8.toString()}
                    ${replaceServerTimeInM3u8.toString()}
                    const workerString = getWasmWorkerJs('${twitchBlobUrl.replaceAll("'", "%27")}');
                    declareOptions(self);
                    GQLDeviceID = ${GQLDeviceID ? "'" + GQLDeviceID + "'" : null};
                    AuthorizationHeader = ${AuthorizationHeader ? "'" + AuthorizationHeader + "'" : undefined};
                    ClientIntegrityHeader = ${ClientIntegrityHeader ? "'" + ClientIntegrityHeader + "'" : null};
                    ClientVersion = ${ClientVersion ? "'" + ClientVersion + "'" : null};
                    ClientSession = ${ClientSession ? "'" + ClientSession + "'" : null};
                    self.addEventListener('message', function(e) {
                        if (e.data.key == 'UpdateClientVersion') {
                            ClientVersion = e.data.value;
                        } else if (e.data.key == 'UpdateClientSession') {
                            ClientSession = e.data.value;
                        } else if (e.data.key == 'UpdateClientId') {
                            ClientID = e.data.value;
                        } else if (e.data.key == 'UpdateDeviceId') {
                            GQLDeviceID = e.data.value;
                        } else if (e.data.key == 'UpdateClientIntegrityHeader') {
                            ClientIntegrityHeader = e.data.value;
                        } else if (e.data.key == 'UpdateAuthorizationHeader') {
                            AuthorizationHeader = e.data.value;
                        } else if (e.data.key == 'FetchResponse') {
                            const responseData = e.data.value;
                            if (pendingFetchRequests.has(responseData.id)) {
                                const { resolve, reject } = pendingFetchRequests.get(responseData.id);
                                pendingFetchRequests.delete(responseData.id);
                                if (responseData.error) {
                                    reject(new Error(responseData.error));
                                } else {
                                    // Create a Response object from the response data
                                    const response = new Response(responseData.body, {
                                        status: responseData.status,
                                        statusText: responseData.statusText,
                                        headers: responseData.headers
                                    });
                                    resolve(response);
                                }
                            }
                        } else if (e.data.key == 'TriggeredPlayerReload') {
                            HasTriggeredPlayerReload = true;
                        } else if (e.data.key == 'SimulateAds') {
                            SimulatedAdsDepth = e.data.value;
                            console.log('SimulatedAdsDepth: ' + SimulatedAdsDepth);
                        } else if (e.data.key == 'AllSegmentsAreAdSegments') {
                            AllSegmentsAreAdSegments = !AllSegmentsAreAdSegments;
                            console.log('AllSegmentsAreAdSegments: ' + AllSegmentsAreAdSegments);
                        }
                    });
                    hookWorkerFetch();
                    eval(workerString);
                `;
                super(URL.createObjectURL(new Blob([newBlobStr])), options);
                twitchWorkers.push(this);
                this.addEventListener('message', (e) => {
                    if (e.data.key == 'UpdateAdBlockBanner') {
                        updateAdblockBanner(e.data);
                    } else if (e.data.key == 'PauseResumePlayer') {
                        doTwitchPlayerTask(true, false);
                    } else if (e.data.key == 'ReloadPlayer') {
                        doTwitchPlayerTask(false, true);
                    }
                });
                this.addEventListener('message', async event => {
                    if (event.data.key == 'FetchRequest') {
                        const fetchRequest = event.data.value;
                        const responseData = await handleWorkerFetchRequest(fetchRequest);
                        this.postMessage({
                            key: 'FetchResponse',
                            value: responseData
                        });
                    }
                });
            }
        };
        let workerInstance = reinsertWorkers(newWorker, reinsert);
        Object.defineProperty(window, 'Worker', {
            get: function() {
                return workerInstance;
            },
            set: function(value) {
                if (isValidWorker(value)) {
                    workerInstance = value;
                } else {
                    console.log('Attempt to set twitch worker denied');
                }
            }
        });
    }
    function getWasmWorkerJs(twitchBlobUrl) {
        const req = new XMLHttpRequest();
        req.open('GET', twitchBlobUrl, false);
        req.overrideMimeType("text/javascript");
        req.send();
        return req.responseText;
    }
    function hookWorkerFetch() {
        console.log('hookWorkerFetch (vaft)');
        const realFetch = fetch;
        fetch = async function(url, options) {
            if (typeof url === 'string') {
                if (AdSegmentCache.has(url)) {
                    return new Promise(function(resolve, reject) {
                        const send = function() {
                            return realFetch('data:video/mp4;base64,AAAAKGZ0eXBtcDQyAAAAAWlzb21tcDQyZGFzaGF2YzFpc282aGxzZgAABEltb292AAAAbG12aGQAAAAAAAAAAAAAAAAAAYagAAAAAAABAAABAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAAABqHRyYWsAAABcdGtoZAAAAAMAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAEAAAAAAQAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAURtZGlhAAAAIG1kaGQAAAAAAAAAAAAAAAAAALuAAAAAAFXEAAAAAAAtaGRscgAAAAAAAAAAc291bgAAAAAAAAAAAAAAAFNvdW5kSGFuZGxlcgAAAADvbWluZgAAABBzbWhkAAAAAAAAAAAAAAAkZGluZgAAABxkcmVmAAAAAAAAAAEAAAAMdXJsIAAAAAEAAACzc3RibAAAAGdzdHNkAAAAAAAAAAEAAABXbXA0YQAAAAAAAAABAAAAAAAAAAAAAgAQAAAAALuAAAAAAAAzZXNkcwAAAAADgICAIgABAASAgIAUQBUAAAAAAAAAAAAAAAWAgIACEZAGgICAAQIAAAAQc3R0cwAAAAAAAAAAAAAAEHN0c2MAAAAAAAAAAAAAABRzdHN6AAAAAAAAAAAAAAAAAAAAEHN0Y28AAAAAAAAAAAAAAeV0cmFrAAAAXHRraGQAAAADAAAAAAAAAAAAAAACAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAAAABAAAAAAoAAAAFoAAAAAAGBbWRpYQAAACBtZGhkAAAAAAAAAAAAAAAAAA9CQAAAAABVxAAAAAAALWhkbHIAAAAAAAAAAHZpZGUAAAAAAAAAAAAAAABWaWRlb0hhbmRsZXIAAAABLG1pbmYAAAAUdm1oZAAAAAEAAAAAAAAAAAAAACRkaW5mAAAAHGRyZWYAAAAAAAAAAQAAAAx1cmwgAAAAAQAAAOxzdGJsAAAAoHN0c2QAAAAAAAAAAQAAAJBhdmMxAAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAoABaABIAAAASAAAAAAAAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGP//AAAAOmF2Y0MBTUAe/+EAI2dNQB6WUoFAX/LgLUBAQFAAAD6AAA6mDgAAHoQAA9CW7y4KAQAEaOuPIAAAABBzdHRzAAAAAAAAAAAAAAAQc3RzYwAAAAAAAAAAAAAAFHN0c3oAAAAAAAAAAAAAAAAAAAAQc3RjbwAAAAAAAAAAAAAASG12ZXgAAAAgdHJleAAAAAAAAAABAAAAAQAAAC4AAAAAAoAAAAAAACB0cmV4AAAAAAAAAAIAAAABAACCNQAAAAACQAAA', options).then(function(response) {
                                resolve(response);
                            })['catch'](function(err) {
                                reject(err);
                            });
                        };
                        send();
                    });
                }
                url = url.trimEnd();
                if (url.endsWith('m3u8')) {
                    return new Promise(function(resolve, reject) {
                        const processAfter = async function(response) {
                            if (response.status === 200) {
                                resolve(new Response(await processM3U8(url, await response.text(), realFetch)));
                            } else {
                                resolve(response);
                            }
                        };
                        const send = function() {
                            return realFetch(url, options).then(function(response) {
                                processAfter(response);
                            })['catch'](function(err) {
                                reject(err);
                            });
                        };
                        send();
                    });
                } else if (url.includes('/channel/hls/') && !url.includes('picture-by-picture')) {
                    V2API = url.includes('/api/v2/');
                    const channelName = (new URL(url)).pathname.match(/([^\/]+)(?=\.\w+$)/)[0];
                    if (ForceAccessTokenPlayerType) {
                        // parent_domains is used to determine if the player is embeded and stripping it gets rid of fake ads
                        const tempUrl = new URL(url);
                        tempUrl.searchParams.delete('parent_domains');
                        url = tempUrl.toString();
                    }
                    return new Promise(function(resolve, reject) {
                        const processAfter = async function(response) {
                            if (response.status == 200) {
                                const encodingsM3u8 = await response.text();
                                const serverTime = getServerTimeFromM3u8(encodingsM3u8);
                                let streamInfo = StreamInfos[channelName];
                                if (streamInfo != null && streamInfo.EncodingsM3U8 != null && (await realFetch(streamInfo.EncodingsM3U8.match(/^https:.*\.m3u8$/m)[0])).status !== 200) {
                                    // The cached encodings are dead (the stream probably restarted)
                                    streamInfo = null;
                                }
                                if (streamInfo == null || streamInfo.EncodingsM3U8 == null) {
                                    StreamInfos[channelName] = streamInfo = {
                                        ChannelName: channelName,
                                        IsShowingAd: false,
                                        LastPlayerReload: 0,
                                        EncodingsM3U8: encodingsM3u8,
                                        ModifiedM3U8: null,
                                        IsUsingModifiedM3U8: false,
                                        UsherParams: (new URL(url)).search,
                                        RequestedAds: new Set(),
                                        Urls: [],// xxx.m3u8 -> { Resolution: "284x160", FrameRate: 30.0 }
                                        ResolutionList: [],
                                        BackupEncodingsM3U8Cache: [],
                                        ActiveBackupPlayerType: null,
                                        IsMidroll: false,
                                        IsStrippingAdSegments: false,
                                        NumStrippedAdSegments: 0
                                    };
                                    const lines = encodingsM3u8.replaceAll('\r', '').split('\n');
                                    for (let i = 0; i < lines.length - 1; i++) {
                                        if (lines[i].startsWith('#EXT-X-STREAM-INF') && lines[i + 1].includes('.m3u8')) {
                                            const attributes = parseAttributes(lines[i]);
                                            const resolution = attributes['RESOLUTION'];
                                            if (resolution) {
                                                const resolutionInfo = {
                                                    Resolution: resolution,
                                                    FrameRate: attributes['FRAME-RATE'],
                                                    Codecs: attributes['CODECS'],
                                                    Url: lines[i + 1]
                                                };
                                                streamInfo.Urls[lines[i + 1]] = resolutionInfo;
                                                streamInfo.ResolutionList.push(resolutionInfo);
                                            }
                                            StreamInfosByUrl[lines[i + 1]] = streamInfo;
                                        }
                                    }
                                    const nonHevcResolutionList = streamInfo.ResolutionList.filter((element) => element.Codecs.startsWith('avc') || element.Codecs.startsWith('av0'));
                                    if (AlwaysReloadPlayerOnAd || (nonHevcResolutionList.length > 0 && streamInfo.ResolutionList.some((element) => element.Codecs.startsWith('hev') || element.Codecs.startsWith('hvc')) && !SkipPlayerReloadOnHevc)) {
                                        if (nonHevcResolutionList.length > 0) {
                                            for (let i = 0; i < lines.length - 1; i++) {
                                                if (lines[i].startsWith('#EXT-X-STREAM-INF')) {
                                                    const resSettings = parseAttributes(lines[i].substring(lines[i].indexOf(':') + 1));
                                                    const codecsKey = 'CODECS';
                                                    if (resSettings[codecsKey].startsWith('hev') || resSettings[codecsKey].startsWith('hvc')) {
                                                        const oldResolution = resSettings['RESOLUTION'];
                                                        const [targetWidth, targetHeight] = oldResolution.split('x').map(Number);
                                                        const newResolutionInfo = nonHevcResolutionList.sort((a, b) => {
                                                            // TODO: Take into account 'Frame-Rate' when sorting (i.e. 1080p60 vs 1080p30)
                                                            const [streamWidthA, streamHeightA] = a.Resolution.split('x').map(Number);
                                                            const [streamWidthB, streamHeightB] = b.Resolution.split('x').map(Number);
                                                            return Math.abs((streamWidthA * streamHeightA) - (targetWidth * targetHeight)) - Math.abs((streamWidthB * streamHeightB) - (targetWidth * targetHeight));
                                                        })[0];
                                                        console.log('ModifiedM3U8 swap ' + resSettings[codecsKey] + ' to ' + newResolutionInfo.Codecs + ' oldRes:' + oldResolution + ' newRes:' + newResolutionInfo.Resolution);
                                                        lines[i] = lines[i].replace(/CODECS="[^"]+"/, `CODECS="${newResolutionInfo.Codecs}"`);
                                                        lines[i + 1] = newResolutionInfo.Url + ' '.repeat(i + 1);// The stream doesn't load unless each url line is unique
                                                    }
                                                }
                                            }
                                        }
                                        if (nonHevcResolutionList.length > 0 || AlwaysReloadPlayerOnAd) {
                                            streamInfo.ModifiedM3U8 = lines.join('\n');
                                        }
                                    }
                                }
                                streamInfo.LastPlayerReload = Date.now();
                                resolve(new Response(replaceServerTimeInM3u8(streamInfo.IsUsingModifiedM3U8 ? streamInfo.ModifiedM3U8 : streamInfo.EncodingsM3U8, serverTime)));
                            } else {
                                resolve(response);
                            }
                        };
                        const send = function() {
                            return realFetch(url, options).then(function(response) {
                                processAfter(response);
                            })['catch'](function(err) {
                                reject(err);
                            });
                        };
                        send();
                    });
                }
            }
            return realFetch.apply(this, arguments);
        };
    }
    function getServerTimeFromM3u8(encodingsM3u8) {
        if (V2API) {
            const matches = encodingsM3u8.match(/#EXT-X-SESSION-DATA:DATA-ID="SERVER-TIME",VALUE="([^"]+)"/);
            return matches.length > 1 ? matches[1] : null;
        }
        const matches = encodingsM3u8.match('SERVER-TIME="([0-9.]+)"');
        return matches.length > 1 ? matches[1] : null;
    }
    function replaceServerTimeInM3u8(encodingsM3u8, newServerTime) {
        if (V2API) {
            return newServerTime ? encodingsM3u8.replace(/(#EXT-X-SESSION-DATA:DATA-ID="SERVER-TIME",VALUE=")[^"]+(")/, `$1${newServerTime}$2`) : encodingsM3u8;
        }
        return newServerTime ? encodingsM3u8.replace(new RegExp('(SERVER-TIME=")[0-9.]+"'), `SERVER-TIME="${newServerTime}"`) : encodingsM3u8;
    }
    function stripAdSegments(textStr, stripAllSegments, streamInfo) {
        let hasStrippedAdSegments = false;
        const lines = textStr.replaceAll('\r', '').split('\n');
        const newAdUrl = 'https://twitch.tv';
        for (let i = 0; i < lines.length; i++) {
            let line = lines[i];
            // Remove tracking urls which appear in the overlay UI
            line = line
                .replaceAll(/(X-TV-TWITCH-AD-URL=")(?:[^"]*)(")/g, `$1${newAdUrl}$2`)
                .replaceAll(/(X-TV-TWITCH-AD-CLICK-TRACKING-URL=")(?:[^"]*)(")/g, `$1${newAdUrl}$2`);
            if (i < lines.length - 1 && line.startsWith('#EXTINF') && (!line.includes(',live') || stripAllSegments || AllSegmentsAreAdSegments)) {
                const segmentUrl = lines[i + 1];
                if (!AdSegmentCache.has(segmentUrl)) {
                    streamInfo.NumStrippedAdSegments++;
                }
                AdSegmentCache.set(segmentUrl, Date.now());
                hasStrippedAdSegments = true;
            }
            if (line.includes(AdSignifier)) {
                hasStrippedAdSegments = true;
            }
        }
        if (hasStrippedAdSegments) {
            for (let i = 0; i < lines.length; i++) {
                // No low latency during ads (otherwise it's possible for the player to prefetch and display ad segments)
                if (lines[i].startsWith('#EXT-X-TWITCH-PREFETCH:')) {
                    lines[i] = '';
                }
            }
        } else {
            streamInfo.NumStrippedAdSegments = 0;
        }
        streamInfo.IsStrippingAdSegments = hasStrippedAdSegments;
        AdSegmentCache.forEach((value, key, map) => {
            if (value < Date.now() - 120000) {
                map.delete(key);
            }
        });
        return lines.join('\n');
    }
    function getStreamUrlForResolution(encodingsM3u8, resolutionInfo) {
        const encodingsLines = encodingsM3u8.replaceAll('\r', '').split('\n');
        const [targetWidth, targetHeight] = resolutionInfo.Resolution.split('x').map(Number);
        let matchedResolutionUrl = null;
        let matchedFrameRate = false;
        let closestResolutionUrl = null;
        let closestResolutionDifference = Infinity;
        for (let i = 0; i < encodingsLines.length - 1; i++) {
            if (encodingsLines[i].startsWith('#EXT-X-STREAM-INF') && encodingsLines[i + 1].includes('.m3u8')) {
                const attributes = parseAttributes(encodingsLines[i]);
                const resolution = attributes['RESOLUTION'];
                const frameRate = attributes['FRAME-RATE'];
                if (resolution) {
                    if (resolution == resolutionInfo.Resolution && (!matchedResolutionUrl || (!matchedFrameRate && frameRate == resolutionInfo.FrameRate))) {
                        matchedResolutionUrl = encodingsLines[i + 1];
                        matchedFrameRate = frameRate == resolutionInfo.FrameRate;
                        if (matchedFrameRate) {
                            return matchedResolutionUrl;
                        }
                    }
                    const [width, height] = resolution.split('x').map(Number);
                    const difference = Math.abs((width * height) - (targetWidth * targetHeight));
                    if (difference < closestResolutionDifference) {
                        closestResolutionUrl = encodingsLines[i + 1];
                        closestResolutionDifference = difference;
                    }
                }
            }
        }
        return closestResolutionUrl;
    }
    async function processM3U8(url, textStr, realFetch) {
        const streamInfo = StreamInfosByUrl[url];
        if (!streamInfo) {
            return textStr;
        }
        if (HasTriggeredPlayerReload) {
            HasTriggeredPlayerReload = false;
            streamInfo.LastPlayerReload = Date.now();
        }
        const haveAdTags = textStr.includes(AdSignifier) || SimulatedAdsDepth > 0;
        if (haveAdTags) {
            streamInfo.IsMidroll = textStr.includes('"MIDROLL"') || textStr.includes('"midroll"');
            if (!streamInfo.IsShowingAd) {
                streamInfo.IsShowingAd = true;
                postMessage({
                    key: 'UpdateAdBlockBanner',
                    isMidroll: streamInfo.IsMidroll,
                    hasAds: streamInfo.IsShowingAd,
                    isStrippingAdSegments: false
                });
            }
            if (!streamInfo.IsMidroll) {
                const lines = textStr.replaceAll('\r', '').split('\n');
                for (let i = 0; i < lines.length; i++) {
                    const line = lines[i];
                    if (line.startsWith('#EXTINF') && lines.length > i + 1) {
                        if (!line.includes(',live') && !streamInfo.RequestedAds.has(lines[i + 1])) {
                            // Only request one .ts file per .m3u8 request to avoid making too many requests
                            //console.log('Fetch ad .ts file');
                            streamInfo.RequestedAds.add(lines[i + 1]);
                            fetch(lines[i + 1]).then((response)=>{response.blob()});
                            break;
                        }
                    }
                }
            }
            const currentResolution = streamInfo.Urls[url];
            if (!currentResolution) {
                console.log('Ads will leak due to missing resolution info for ' + url);
                return textStr;
            }
            const isHevc = currentResolution.Codecs.startsWith('hev') || currentResolution.Codecs.startsWith('hvc');
            if (((isHevc && !SkipPlayerReloadOnHevc) || AlwaysReloadPlayerOnAd) && streamInfo.ModifiedM3U8 && !streamInfo.IsUsingModifiedM3U8) {
                streamInfo.IsUsingModifiedM3U8 = true;
                streamInfo.LastPlayerReload = Date.now();
                postMessage({
                    key: 'ReloadPlayer'
                });
            }
            let backupPlayerType = null;
            let backupM3u8 = null;
            let fallbackM3u8 = null;
            let startIndex = 0;
            let isDoingMinimalRequests = false;
            if (streamInfo.LastPlayerReload > Date.now() - PlayerReloadMinimalRequestsTime) {
                // When doing player reload there are a lot of requests which causes the backup stream to load in slow. Briefly prefer using a single version to prevent long delays
                startIndex = PlayerReloadMinimalRequestsPlayerIndex;
                isDoingMinimalRequests = true;
            }
            for (let playerTypeIndex = startIndex; !backupM3u8 && playerTypeIndex < BackupPlayerTypes.length; playerTypeIndex++) {
                const playerType = BackupPlayerTypes[playerTypeIndex];
                const realPlayerType = playerType.replace('-CACHED', '');
                const isFullyCachedPlayerType = playerType != realPlayerType;
                for (let i = 0; i < 2; i++) {
                    // This caches the m3u8 if it doesn't have ads. If the already existing cache has ads it fetches a new version (second loop)
                    let isFreshM3u8 = false;
                    let encodingsM3u8 = streamInfo.BackupEncodingsM3U8Cache[playerType];
                    if (!encodingsM3u8) {
                        isFreshM3u8 = true;
                        try {
                            const accessTokenResponse = await getAccessToken(streamInfo.ChannelName, realPlayerType);
                            if (accessTokenResponse.status === 200) {
                                const accessToken = await accessTokenResponse.json();
                                const urlInfo = new URL('https://usher.ttvnw.net/api/' + (V2API ? 'v2/' : '') + 'channel/hls/' + streamInfo.ChannelName + '.m3u8' + streamInfo.UsherParams);
                                urlInfo.searchParams.set('sig', accessToken.data.streamPlaybackAccessToken.signature);
                                urlInfo.searchParams.set('token', accessToken.data.streamPlaybackAccessToken.value);
                                const encodingsM3u8Response = await realFetch(urlInfo.href);
                                if (encodingsM3u8Response.status === 200) {
                                    encodingsM3u8 = streamInfo.BackupEncodingsM3U8Cache[playerType] = await encodingsM3u8Response.text();
                                }
                            }
                        } catch (err) {}
                    }
                    if (encodingsM3u8) {
                        try {
                            const streamM3u8Url = getStreamUrlForResolution(encodingsM3u8, currentResolution);
                            const streamM3u8Response = await realFetch(streamM3u8Url);
                            if (streamM3u8Response.status == 200) {
                                const m3u8Text = await streamM3u8Response.text();
                                if (m3u8Text) {
                                    if (playerType == FallbackPlayerType) {
                                        fallbackM3u8 = m3u8Text;
                                    }
                                    if ((!m3u8Text.includes(AdSignifier) && (SimulatedAdsDepth == 0 || playerTypeIndex >= SimulatedAdsDepth - 1)) || (!fallbackM3u8 && playerTypeIndex >= BackupPlayerTypes.length - 1)) {
                                        backupPlayerType = playerType;
                                        backupM3u8 = m3u8Text;
                                        break;
                                    }
                                    if (isFullyCachedPlayerType) {
                                        break;
                                    }
                                    if (isDoingMinimalRequests) {
                                        backupPlayerType = playerType;
                                        backupM3u8 = m3u8Text;
                                        break;
                                    }
                                }
                            }
                        } catch (err) {}
                    }
                    streamInfo.BackupEncodingsM3U8Cache[playerType] = null;
                    if (isFreshM3u8) {
                        break;
                    }
                }
            }
            if (!backupM3u8 && fallbackM3u8) {
                backupPlayerType = FallbackPlayerType;
                backupM3u8 = fallbackM3u8;
            }
            if (backupM3u8) {
                textStr = backupM3u8;
                if (streamInfo.ActiveBackupPlayerType != backupPlayerType) {
                    streamInfo.ActiveBackupPlayerType = backupPlayerType;
                    console.log(`Blocking${(streamInfo.IsMidroll ? ' midroll ' : ' ')}ads (${backupPlayerType})`);
                }
            }
            // TODO: Improve hevc stripping. It should always strip when there is a codec mismatch (both ways)
            const stripHevc = isHevc && streamInfo.ModifiedM3U8;
            if (IsAdStrippingEnabled || stripHevc) {
                textStr = stripAdSegments(textStr, stripHevc, streamInfo);
            }
        } else if (streamInfo.IsShowingAd) {
            console.log('Finished blocking ads');
            streamInfo.IsShowingAd = false;
            streamInfo.IsStrippingAdSegments = false;
            streamInfo.NumStrippedAdSegments = 0;
            streamInfo.ActiveBackupPlayerType = null;
            if (streamInfo.IsUsingModifiedM3U8 || ReloadPlayerAfterAd) {
                streamInfo.IsUsingModifiedM3U8 = false;
                streamInfo.LastPlayerReload = Date.now();
                postMessage({
                    key: 'ReloadPlayer'
                });
            } else {
                postMessage({
                    key: 'PauseResumePlayer'
                });
            }
        }
        postMessage({
            key: 'UpdateAdBlockBanner',
            isMidroll: streamInfo.IsMidroll,
            hasAds: streamInfo.IsShowingAd,
            isStrippingAdSegments: streamInfo.IsStrippingAdSegments,
            numStrippedAdSegments: streamInfo.NumStrippedAdSegments
        });
        return textStr;
    }
    function parseAttributes(str) {
        return Object.fromEntries(
            str.split(/(?:^|,)((?:[^=]*)=(?:"[^"]*"|[^,]*))/)
            .filter(Boolean)
            .map(x => {
                const idx = x.indexOf('=');
                const key = x.substring(0, idx);
                const value = x.substring(idx + 1);
                const num = Number(value);
                return [key, Number.isNaN(num) ? value.startsWith('"') ? JSON.parse(value) : value : num];
            }));
    }
    function getAccessToken(channelName, playerType) {
        const body = {
            operationName: 'PlaybackAccessToken',
            variables: {
                isLive: true,
                login: channelName,
                isVod: false,
                vodID: "",
                playerType: playerType,
                platform: playerType == 'autoplay' ? 'android' : 'web'
            },
            extensions: {
                persistedQuery: {
                    version:1,
                    sha256Hash:"ed230aa1e33e07eebb8928504583da78a5173989fadfb1ac94be06a04f3cdbe9"
                }
            }
        };
        return gqlRequest(body, playerType);
    }
    function gqlRequest(body, playerType) {
        if (!GQLDeviceID) {
            GQLDeviceID = '';
            const dcharacters = 'abcdefghijklmnopqrstuvwxyz0123456789';
            const dcharactersLength = dcharacters.length;
            for (let i = 0; i < 32; i++) {
                GQLDeviceID += dcharacters.charAt(Math.floor(Math.random() * dcharactersLength));
            }
        }
        let headers = {
            'Client-ID': ClientID,
            'X-Device-Id': GQLDeviceID,
            'Authorization': AuthorizationHeader,
            ...(ClientIntegrityHeader && {'Client-Integrity': ClientIntegrityHeader}),
            ...(ClientVersion && {'Client-Version': ClientVersion}),
            ...(ClientSession && {'Client-Session-Id': ClientSession})
        };
        return new Promise((resolve, reject) => {
            const requestId = Math.random().toString(36).substring(2, 15);
            const fetchRequest = {
                id: requestId,
                url: 'https://gql.twitch.tv/gql',
                options: {
                    method: 'POST',
                    body: JSON.stringify(body),
                    headers
                }
            };
            pendingFetchRequests.set(requestId, {
                resolve,
                reject
            });
            postMessage({
                key: 'FetchRequest',
                value: fetchRequest
            });
        });
    }
    let playerForMonitoringBuffering = null;
    const playerBufferState = {
        channelName: null,
        hasStreamStarted: false,
        position: 0,
        bufferedPosition: 0,
        bufferDuration: 0,
        numSame: 0,
        lastFixTime: 0,
        isLive: true
    };
    function monitorPlayerBuffering() {
        if (playerForMonitoringBuffering) {
            try {
                const player = playerForMonitoringBuffering.player;
                const state = playerForMonitoringBuffering.state;
                if (!player.core) {
                    playerForMonitoringBuffering = null;
                } else if (state.props?.content?.type === 'live' && !player.isPaused() && !player.getHTMLVideoElement()?.ended && playerBufferState.lastFixTime <= Date.now() - PlayerBufferingMinRepeatDelay && !isActivelyStrippingAds) {
                    const m3u8Url = player.core?.state?.path;
                    if (m3u8Url) {
                      const fileName = new URL(m3u8Url).pathname.split('/').pop();
                      if (fileName?.endsWith('.m3u8')) {
                          const channelName = fileName.slice(0, -5);
                          if (playerBufferState.channelName != channelName) {
                              playerBufferState.channelName = channelName;
                              playerBufferState.hasStreamStarted = false;
                              playerBufferState.numSame = 0;
                              //console.log('Channel changed to ' + channelName);
                          }
                      }
                    }
                    if (player.getState() === 'Playing') {
                        playerBufferState.hasStreamStarted = true;
                    }
                    const position = player.core?.state?.position;
                    const bufferedPosition = player.core?.state?.bufferedPosition;
                    const bufferDuration = player.getBufferDuration();
                    if (position !== undefined && bufferedPosition !== undefined) {
                        //console.log('position:' + position + ' bufferDuration:' + bufferDuration + ' bufferPosition:' + bufferedPosition + ' state: ' + player.core?.state?.state + ' started: ' + playerBufferState.hasStreamStarted);
                        // NOTE: This could be improved. It currently lets the player fully eat the full buffer before it triggers pause/play
                        if (playerBufferState.hasStreamStarted &&
                            (!PlayerBufferingPrerollCheckEnabled || position > PlayerBufferingPrerollCheckOffset) &&
                            (playerBufferState.position == position || bufferDuration < PlayerBufferingDangerZone)  &&
                            playerBufferState.bufferedPosition == bufferedPosition &&
                            playerBufferState.bufferDuration >= bufferDuration &&
                            (position != 0 || bufferedPosition != 0 || bufferDuration != 0)
                        ) {
                            playerBufferState.numSame++;
                            if (playerBufferState.numSame == PlayerBufferingSameStateCount) {
                                console.log('Attempt to fix buffering position:' + playerBufferState.position + ' bufferedPosition:' + playerBufferState.bufferedPosition + ' bufferDuration:' + playerBufferState.bufferDuration);
                                const isPausePlay = !PlayerBufferingDoPlayerReload;
                                const isReload = PlayerBufferingDoPlayerReload;
                                doTwitchPlayerTask(isPausePlay, isReload);
                                playerBufferState.lastFixTime = Date.now();
                                playerBufferState.numSame = 0;
                            }
                        } else {
                            playerBufferState.numSame = 0;
                        }
                        playerBufferState.position = position;
                        playerBufferState.bufferedPosition = bufferedPosition;
                        playerBufferState.bufferDuration = bufferDuration;
                    } else {
                        playerBufferState.numSame = 0;
                    }
                }
            } catch (err) {
                console.error('error when monitoring player for buffering: ' + err);
                playerForMonitoringBuffering = null;
            }
        }
        if (!playerForMonitoringBuffering) {
            const playerAndState = getPlayerAndState();
            if (playerAndState && playerAndState.player && playerAndState.state) {
                playerForMonitoringBuffering = {
                    player: playerAndState.player,
                    state: playerAndState.state
                };
            }
        }
        const isLive = playerForMonitoringBuffering?.state?.props?.content?.type === 'live';
        if (playerBufferState.isLive && !isLive) {
            updateAdblockBanner({
                hasAds: false
            });
        }
        playerBufferState.isLive = isLive;
        setTimeout(monitorPlayerBuffering, PlayerBufferingDelay);
    }
    function updateAdblockBanner(data) {
        const playerRootDiv = document.querySelector('.video-player');
        if (playerRootDiv != null) {
            let adBlockDiv = null;
            adBlockDiv = playerRootDiv.querySelector('.adblock-overlay');
            if (adBlockDiv == null) {
                adBlockDiv = document.createElement('div');
                adBlockDiv.className = 'adblock-overlay';
                adBlockDiv.innerHTML = '<div class="player-adblock-notice" style="color: white; background-color: rgba(0, 0, 0, 0.8); position: absolute; top: 0px; left: 0px; padding: 5px;"><p></p></div>';
                adBlockDiv.style.display = 'none';
                adBlockDiv.P = adBlockDiv.querySelector('p');
                playerRootDiv.appendChild(adBlockDiv);
            }
            if (adBlockDiv != null) {
                isActivelyStrippingAds = data.isStrippingAdSegments;
                adBlockDiv.P.textContent = 'Blocking' + (data.isMidroll ? ' midroll' : '') + ' ads' + (data.isStrippingAdSegments ? ' (stripping)' : '');// + (data.numStrippedAdSegments > 0 ? ` (${data.numStrippedAdSegments})` : '');
                adBlockDiv.style.display = data.hasAds && playerBufferState.isLive ? 'block' : 'none';
            }
        }
    }
    function getPlayerAndState() {
        function findReactNode(root, constraint) {
            if (root.stateNode && constraint(root.stateNode)) {
                return root.stateNode;
            }
            let node = root.child;
            while (node) {
                const result = findReactNode(node, constraint);
                if (result) {
                    return result;
                }
                node = node.sibling;
            }
            return null;
        }
        function findReactRootNode() {
            let reactRootNode = null;
            const rootNode = document.querySelector('#root');
            if (rootNode && rootNode._reactRootContainer && rootNode._reactRootContainer._internalRoot && rootNode._reactRootContainer._internalRoot.current) {
                reactRootNode = rootNode._reactRootContainer._internalRoot.current;
            }
            if (reactRootNode == null && rootNode != null) {
                const containerName = Object.keys(rootNode).find(x => x.startsWith('__reactContainer'));
                if (containerName != null) {
                    reactRootNode = rootNode[containerName];
                }
            }
            return reactRootNode;
        }
        const reactRootNode = findReactRootNode();
        if (!reactRootNode) {
            return null;
        }
        let player = findReactNode(reactRootNode, node => node.setPlayerActive && node.props && node.props.mediaPlayerInstance);
        player = player && player.props && player.props.mediaPlayerInstance ? player.props.mediaPlayerInstance : null;
        if (player?.playerInstance) {
            player = player.playerInstance;
        }
        const playerState = findReactNode(reactRootNode, node => node.setSrc && node.setInitialPlaybackSettings);
        return  {
            player: player,
            state: playerState
        };
    }
    function doTwitchPlayerTask(isPausePlay, isReload) {
        const playerAndState = getPlayerAndState();
        if (!playerAndState) {
            console.log('Could not find react root');
            return;
        }
        const player = playerAndState.player;
        const playerState = playerAndState.state;
        if (!player) {
            console.log('Could not find player');
            return;
        }
        if (!playerState) {
            console.log('Could not find player state');
            return;
        }
        if (player.isPaused() || player.core?.paused) {
            return;
        }
        playerBufferState.lastFixTime = Date.now();
        playerBufferState.numSame = 0;
        if (isPausePlay) {
            player.pause();
            player.play();
            return;
        }
        if (isReload) {
            const lsKeyQuality = 'video-quality';
            const lsKeyMuted = 'video-muted';
            const lsKeyVolume = 'volume';
            let currentQualityLS = null;
            let currentMutedLS = null;
            let currentVolumeLS = null;
            try {
                currentQualityLS = localStorage.getItem(lsKeyQuality);
                currentMutedLS = localStorage.getItem(lsKeyMuted);
                currentVolumeLS = localStorage.getItem(lsKeyVolume);
                if (localStorageHookFailed && player?.core?.state) {
                    localStorage.setItem(lsKeyMuted, JSON.stringify({default:player.core.state.muted}));
                    localStorage.setItem(lsKeyVolume, player.core.state.volume);
                }
                if (localStorageHookFailed && player?.core?.state?.quality?.group) {
                    localStorage.setItem(lsKeyQuality, JSON.stringify({default:player.core.state.quality.group}));
                }
            } catch {}
            console.log('Reloading Twitch player');
            playerState.setSrc({ isNewMediaPlayerInstance: true, refreshAccessToken: true });
            postTwitchWorkerMessage('TriggeredPlayerReload');
            player.play();
            if (localStorageHookFailed && (currentQualityLS || currentMutedLS || currentVolumeLS)) {
                setTimeout(() => {
                    try {
                        if (currentQualityLS) {
                            localStorage.setItem(lsKeyQuality, currentQualityLS);
                        }
                        if (currentMutedLS) {
                            localStorage.setItem(lsKeyMuted, currentMutedLS);
                        }
                        if (currentVolumeLS) {
                            localStorage.setItem(lsKeyVolume, currentVolumeLS);
                        }
                    } catch {}
                }, 3000);
            }
            return;
        }
    }
    window.reloadTwitchPlayer = () => {
        doTwitchPlayerTask(false, true);
    };
    function postTwitchWorkerMessage(key, value) {
        twitchWorkers.forEach((worker) => {
            worker.postMessage({key: key, value: value});
        });
    }
    async function handleWorkerFetchRequest(fetchRequest) {
        try {
            const response = await window.realFetch(fetchRequest.url, fetchRequest.options);
            const responseBody = await response.text();
            const responseObject = {
                id: fetchRequest.id,
                status: response.status,
                statusText: response.statusText,
                headers: Object.fromEntries(response.headers.entries()),
                body: responseBody
            };
            return responseObject;
        } catch (error) {
            return {
                id: fetchRequest.id,
                error: error.message
            };
        }
    }
    function hookFetch() {
        const realFetch = window.fetch;
        window.realFetch = realFetch;
        window.fetch = function(url, init, ...args) {
            if (typeof url === 'string') {
                if (url.includes('gql')) {
                    let deviceId = init.headers['X-Device-Id'];
                    if (typeof deviceId !== 'string') {
                        deviceId = init.headers['Device-ID'];
                    }
                    if (typeof deviceId === 'string' && GQLDeviceID != deviceId) {
                        GQLDeviceID = deviceId;
                        postTwitchWorkerMessage('UpdateDeviceId', GQLDeviceID);
                    }
                    if (typeof init.headers['Client-Version'] === 'string' && init.headers['Client-Version'] !== ClientVersion) {
                        postTwitchWorkerMessage('UpdateClientVersion', ClientVersion = init.headers['Client-Version']);
                    }
                    if (typeof init.headers['Client-Session-Id'] === 'string' && init.headers['Client-Session-Id'] !== ClientSession) {
                        postTwitchWorkerMessage('UpdateClientSession', ClientSession = init.headers['Client-Session-Id']);
                    }
                    if (typeof init.headers['Client-Integrity'] === 'string' && init.headers['Client-Integrity'] !== ClientIntegrityHeader) {
                        postTwitchWorkerMessage('UpdateClientIntegrityHeader', ClientIntegrityHeader = init.headers['Client-Integrity']);
                    }
                    if (typeof init.headers['Authorization'] === 'string' && init.headers['Authorization'] !== AuthorizationHeader) {
                        postTwitchWorkerMessage('UpdateAuthorizationHeader', AuthorizationHeader = init.headers['Authorization']);
                    }
                    // Get rid of mini player above chat - TODO: Reject this locally instead of having server reject it
                    if (init && typeof init.body === 'string' && init.body.includes('PlaybackAccessToken') && init.body.includes('picture-by-picture')) {
                        init.body = '';
                    }
                    if (ForceAccessTokenPlayerType && typeof init.body === 'string' && init.body.includes('PlaybackAccessToken')) {
                        let replacedPlayerType = '';
                        const newBody = JSON.parse(init.body);
                        if (Array.isArray(newBody)) {
                            for (let i = 0; i < newBody.length; i++) {
                                if (newBody[i]?.variables?.playerType && newBody[i]?.variables?.playerType !== ForceAccessTokenPlayerType) {
                                    replacedPlayerType = newBody[i].variables.playerType;
                                    newBody[i].variables.playerType = ForceAccessTokenPlayerType;
                                }
                            }
                        } else {
                            if (newBody?.variables?.playerType && newBody?.variables?.playerType !== ForceAccessTokenPlayerType) {
                                replacedPlayerType = newBody.variables.playerType;
                                newBody.variables.playerType = ForceAccessTokenPlayerType;
                            }
                        }
                        if (replacedPlayerType) {
                            console.log(`Replaced '${replacedPlayerType}' player type with '${ForceAccessTokenPlayerType}' player type`);
                            init.body = JSON.stringify(newBody);
                        }
                    }
                }
            }
            return realFetch.apply(this, arguments);
        };
    }
    function onContentLoaded() {
        // This stops Twitch from pausing the player when in another tab and an ad shows.
        // Taken from https://github.com/saucettv/VideoAdBlockForTwitch/blob/cefce9d2b565769c77e3666ac8234c3acfe20d83/chrome/content.js#L30
        try {
            Object.defineProperty(document, 'visibilityState', {
                get() {
                    return 'visible';
                }
            });
        }catch{}
        let hidden = document.__lookupGetter__('hidden');
        let webkitHidden = document.__lookupGetter__('webkitHidden');
        try {
            Object.defineProperty(document, 'hidden', {
                get() {
                    return false;
                }
            });
        }catch{}
        const block = e => {
            e.preventDefault();
            e.stopPropagation();
            e.stopImmediatePropagation();
        };
        let wasVideoPlaying = true;
        const visibilityChange = e => {
            const isChrome = typeof chrome !== 'undefined';
            const videos = document.getElementsByTagName('video');
            if (videos.length > 0) {
                if (hidden.apply(document) === true || (webkitHidden && webkitHidden.apply(document) === true)) {
                    wasVideoPlaying = !videos[0].paused && !videos[0].ended;
                } else {
                    if (!playerBufferState.hasStreamStarted) {
                        //console.log('Tab focused. Stream should be active');
                        playerBufferState.hasStreamStarted = true;
                    }
                    if (isChrome && wasVideoPlaying && !videos[0].ended && videos[0].paused && videos[0].muted) {
                        videos[0].play();
                    }
                }
            }
            block(e);
        };
        document.addEventListener('visibilitychange', visibilityChange, true);
        document.addEventListener('webkitvisibilitychange', visibilityChange, true);
        document.addEventListener('mozvisibilitychange', visibilityChange, true);
        document.addEventListener('hasFocus', block, true);
        try {
            if (/Firefox/.test(navigator.userAgent)) {
                Object.defineProperty(document, 'mozHidden', {
                    get() {
                        return false;
                    }
                });
            } else {
                Object.defineProperty(document, 'webkitHidden', {
                    get() {
                        return false;
                    }
                });
            }
        }catch{}
        // Hooks for preserving volume / resolution
        try {
            const keysToCache = [
                'video-quality',
                'video-muted',
                'volume',
                'lowLatencyModeEnabled',// Low Latency
                'persistenceEnabled',// Mini Player
            ];
            const cachedValues = new Map();
            for (let i = 0; i < keysToCache.length; i++) {
                cachedValues.set(keysToCache[i], localStorage.getItem(keysToCache[i]));
            }
            const realSetItem = localStorage.setItem;
            localStorage.setItem = function(key, value) {
                if (cachedValues.has(key)) {
                    cachedValues.set(key, value);
                }
                realSetItem.apply(this, arguments);
            };
            const realGetItem = localStorage.getItem;
            localStorage.getItem = function(key) {
                if (cachedValues.has(key)) {
                    return cachedValues.get(key);
                }
                return realGetItem.apply(this, arguments);
            };
            if (!localStorage.getItem.toString().includes(Object.keys({cachedValues})[0])) {
                // These hooks are useful to preserve player state on player reload
                // Firefox doesn't allow hooking of localStorage functions but chrome does
                localStorageHookFailed = true;
            }
        } catch (err) {
            console.log('localStorageHooks failed ' + err)
            localStorageHookFailed = true;
        }
    }
    declareOptions(window);
    hookWindowWorker();
    hookFetch();
    if (PlayerBufferingFix) {
        monitorPlayerBuffering();
    }
    if (document.readyState === "complete" || document.readyState === "loaded" || document.readyState === "interactive") {
        onContentLoaded();
    } else {
        window.addEventListener("DOMContentLoaded", function() {
            onContentLoaded();
        });
    }
    window.simulateAds = (depth) => {
        if (depth === undefined || depth < 0) {
            console.log('Ad depth paramter required (0 = no simulated ad, 1+ = use backup player for given depth)');
            return;
        }
        postTwitchWorkerMessage('SimulateAds', depth);
    };
    window.allSegmentsAreAdSegments = () => {
        postTwitchWorkerMessage('AllSegmentsAreAdSegments');
    };
})();
```

</details><br>

Instagram Auto Unmute:  
[Greasyfork](https://greasyfork.org/en/scripts/486309-auto-unmute-instagram-stories-and-keyboard-shortcut-for-fullscreen-video)
<details>
<summary>
Archive (1.3.0)
</summary>

```js
// ==UserScript==
// @name        Auto unmute Instagram stories and keyboard shortcut for fullscreen video
// @namespace   Violentmonkey Scripts
// @match       https://www.instagram.com/*
// @grant       none
// @version     1.3.0
// @author      Ricky
// @description Automatically unmute stories / turn story audio on with fullscreen toggle for video
// @downloadURL https://update.greasyfork.org/scripts/486309/Auto%20unmute%20Instagram%20stories%20and%20keyboard%20shortcut%20for%20fullscreen%20video.user.js
// @updateURL https://update.greasyfork.org/scripts/486309/Auto%20unmute%20Instagram%20stories%20and%20keyboard%20shortcut%20for%20fullscreen%20video.meta.js
// ==/UserScript==

var mainLoop = setInterval(() => {
  try {
    var videoElement = document.querySelector('video');

    // New Instagram UI: volume slider with muted indicator
    var volumeSlider = document.querySelector('div[aria-label="Adjust volume"][role="slider"]');
    var mutedIcon = document.querySelector('svg[aria-label="Audio is muted"]');

    // Fallback to old selector
    var audioButton = document.querySelector('button[aria-label="Toggle audio"]');

    if (videoElement && volumeSlider && mutedIcon) {
      // New UI: click the volume slider area to unmute
      if (!volumeSlider.getAttribute('jside_done')) {
        volumeSlider.click();
        volumeSlider.setAttribute('jside_done', 'true');
      }
    } else if (videoElement && volumeSlider && !mutedIcon) {
      // Audio is not muted, reset the flag
      volumeSlider.removeAttribute('jside_done');
    } else if (videoElement && audioButton) {
      // Old UI fallback
      if (videoElement.muted && !audioButton.getAttribute('jside_done')) {
        audioButton.click();
        audioButton.setAttribute('jside_done', 'true');
      } else if (!videoElement.muted) {
        audioButton.removeAttribute('jside_done');
      }
    }
  } catch (err) {
    // Handle errors, or you can log them to the console for debugging
  }
}, 1000);

// Function to toggle fullscreen on video
function toggleVideoFullscreen() {
  var videoElement = document.querySelector('video');
  if (videoElement) {
    if (!document.fullscreenElement) {
      videoElement.requestFullscreen();
    } else {
      if (document.exitFullscreen) {
        document.exitFullscreen();
      }
    }
  }
}

// Event listener for key press
document.addEventListener('keydown', (event) => {
  if (event.key === 'f') {
    toggleVideoFullscreen();
  }
});
```

</details><br>

Youtube CPU Tamer:  
[Greasyfork](https://greasyfork.org/en/scripts/431573-youtube-cpu-tamer-by-animationframe)
[Github](https://github.com/cyfung1031/userscript-supports)
<details>
<summary>
Archive (2025.02.24.0)
</summary>

```js
// ==UserScript==
// @name                YouTube CPU Tamer by AnimationFrame
// @name:ja             YouTube CPU Tamer by AnimationFrame
// @name:zh-TW          YouTube CPU Tamer by AnimationFrame
// @namespace           http://tampermonkey.net/
// @version             2025.02.24.0
// @license             MIT License
// @author              CY Fung
// @match               https://www.youtube.com/*
// @match               https://www.youtube.com/embed/*
// @match               https://www.youtube-nocookie.com/embed/*
// @match               https://www.youtube.com/live_chat*
// @match               https://www.youtube.com/live_chat_replay*
// @match               https://music.youtube.com/*
// @exclude             /^https?://\S+\.(txt|png|jpg|jpeg|gif|xml|svg|manifest|log|ini)[^\/]*$/
// @icon                https://raw.githubusercontent.com/cyfung1031/userscript-supports/main/icons/youtube-cpu-tamper-by-animationframe.webp
// @supportURL          https://github.com/cyfung1031/userscript-supports
// @run-at              document-start
// @grant               none
// @unwrap
// @allFrames           true
// @inject-into         page

// @description         Reduce Browser's Energy Impact for playing YouTube Video
// @downloadURL https://update.greasyfork.org/scripts/431573/YouTube%20CPU%20Tamer%20by%20AnimationFrame.user.js
// @updateURL https://update.greasyfork.org/scripts/431573/YouTube%20CPU%20Tamer%20by%20AnimationFrame.meta.js
// ==/UserScript==

/*

MIT License

Copyright 2021-2025 CY Fung

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

*/

/* jshint esversion:8 */

((__CONTEXT__) => {
  'use strict';

  const win = this instanceof Window ? this : window;

  // Create a unique key for the script and check if it is already running
  const hkey_script = 'nzsxclvflluv';
  if (win[hkey_script]) throw new Error('Duplicated Userscript Calling'); // avoid duplicated scripting
  win[hkey_script] = true;

  /** @type {globalThis.PromiseConstructor} */
  const Promise = (async () => { })().constructor; // YouTube hacks Promise in WaterFox Classic and "Promise.resolve(0)" nevers resolve.
  const PromiseExternal = ((resolve_, reject_) => {
    const h = (resolve, reject) => { resolve_ = resolve; reject_ = reject };
    return class PromiseExternal extends Promise {
      constructor(cb = h) {
        super(cb);
        if (cb === h) {
          /** @type {(value: any) => void} */
          this.resolve = resolve_;
          /** @type {(reason?: any) => void} */
          this.reject = reject_;
        }
      }
    };
  })();

  const isGPUAccelerationAvailable = (() => {
    // https://gist.github.com/cvan/042b2448fcecefafbb6a91469484cdf8
    try {
      const canvas = document.createElement('canvas');
      return !!(canvas.getContext('webgl') || canvas.getContext('experimental-webgl'));
    } catch (e) {
      return false;
    }
  })();

  if (!isGPUAccelerationAvailable) {
    throw new Error('Your browser does not support GPU Acceleration. YouTube CPU Tamer by AnimationFrame is skipped.');
  }

  const timeupdateDT = (() => {

    window.__j6YiAc__ = 1;

    document.addEventListener('timeupdate', () => {
      window.__j6YiAc__ = Date.now();
    }, true);

    let kz = -1;
    try {
      kz = top.__j6YiAc__;
    } catch (e) {

    }

    return kz >= 1 ? () => top.__j6YiAc__ : () => window.__j6YiAc__;

  })();

  const cleanContext = async (win) => {
    const waitFn = requestAnimationFrame; // shall have been binded to window
    try {
      let mx = 16; // MAX TRIAL
      const frameId = 'vanillajs-iframe-v1'
      let frame = document.getElementById(frameId);
      let removeIframeFn = null;
      if (!frame) {
        frame = document.createElement('iframe');
        frame.id = frameId;
        const blobURL = typeof webkitCancelAnimationFrame === 'function' && typeof kagi === 'undefined' ? (frame.src = URL.createObjectURL(new Blob([], { type: 'text/html' }))) : null; // avoid Brave Crash
        frame.sandbox = 'allow-same-origin'; // script cannot be run inside iframe but API can be obtained from iframe
        let n = document.createElement('noscript'); // wrap into NOSCRPIT to avoid reflow (layouting)
        n.appendChild(frame);
        while (!document.documentElement && mx-- > 0) await new Promise(waitFn); // requestAnimationFrame here could get modified by YouTube engine
        const root = document.documentElement;
        root.appendChild(n); // throw error if root is null due to exceeding MAX TRIAL
        if (blobURL) Promise.resolve().then(() => URL.revokeObjectURL(blobURL));

        removeIframeFn = (setTimeout) => {
          const removeIframeOnDocumentReady = (e) => {
            e && win.removeEventListener("DOMContentLoaded", removeIframeOnDocumentReady, false);
            e = n;
            n = win = removeIframeFn = 0;
            setTimeout ? setTimeout(() => e.remove(), 200) : e.remove();
          }
          if (!setTimeout || document.readyState !== 'loading') {
            removeIframeOnDocumentReady();
          } else {
            win.addEventListener("DOMContentLoaded", removeIframeOnDocumentReady, false);
          }
        }
      }
      while (!frame.contentWindow && mx-- > 0) await new Promise(waitFn);
      const fc = frame.contentWindow;
      if (!fc) throw "window is not found."; // throw error if root is null due to exceeding MAX TRIAL
      try {
        const { requestAnimationFrame, setInterval, setTimeout, clearInterval, clearTimeout } = fc;
        const res = { requestAnimationFrame, setInterval, setTimeout, clearInterval, clearTimeout };
        for (let k in res) res[k] = res[k].bind(win); // necessary
        if (removeIframeFn) Promise.resolve(res.setTimeout).then(removeIframeFn);
        return res;
      } catch (e) {
        if (removeIframeFn) removeIframeFn();
        return null;
      }
    } catch (e) {
      console.warn(e);
      return null;
    }
  };

  cleanContext(win).then(__CONTEXT__ => {

    if (!__CONTEXT__) return null;

    const { requestAnimationFrame, setTimeout, setInterval, clearTimeout, clearInterval } = __CONTEXT__;

    /** @type {Function|null} */
    let afInterupter = null;

    const getRAFHelper = () => {
      const asc = document.createElement('a-f');
      if (!('onanimationiteration' in asc)) {
        return (resolve) => requestAnimationFrame(afInterupter = resolve);
      }
      asc.id = 'a-f';
      let qr = null;
      asc.onanimationiteration = function () {
        if (qr !== null) qr = (qr(), null);
      }
      if (!document.getElementById('afscript')) {
        const style = document.createElement('style');
        style.id = 'afscript';
        style.textContent = `
          @keyFrames aF1 {
            0% {
              order: 0;
            }
            100% {
              order: 1;
            }
          }
          #a-f[id] {
            visibility: collapse !important;
            position: fixed !important;
            display: block !important;
            top: -100px !important;
            left: -100px !important;
            margin:0 !important;
            padding:0 !important;
            outline:0 !important;
            border:0 !important;
            z-index:-1 !important;
            width: 0px !important;
            height: 0px !important;
            contain: strict !important;
            pointer-events: none !important;
            animation: 1ms steps(2, jump-none) 0ms infinite alternate forwards running aF1 !important;
          }
        `;
        (document.head || document.documentElement).appendChild(style);
      }
      document.documentElement.insertBefore(asc, document.documentElement.firstChild);
      return (resolve) => (qr = afInterupter = resolve);
    };

    /** @type {(resolve: () => void)}  */
    const rafPN = getRAFHelper(); // rAF will not execute if document is hidden

    (() => {
      let afPromiseP, afPromiseQ; // non-null
      afPromiseP = afPromiseQ = { resolved: true }; // initial state for !uP && !uQ
      let afix = 0;
      const afResolve = async (rX) => {
        await new Promise(rafPN);
        rX.resolved = true;
        const t = afix = (afix & 1073741823) + 1;
        return rX.resolve(t), t;
      };
      const eFunc = async () => {
        const uP = !afPromiseP.resolved ? afPromiseP : null;
        const uQ = !afPromiseQ.resolved ? afPromiseQ : null;
        let t = 0;
        if (uP && uQ) {
          const t1 = await uP;
          const t2 = await uQ;
          t = ((t1 - t2) & 536870912) === 0 ? t1 : t2; // = 0 for t1 - t2 = [0, 536870911], [–1073741824, -536870913]
        } else {
          const vP = !uP ? (afPromiseP = new PromiseExternal()) : null;
          const vQ = !uQ ? (afPromiseQ = new PromiseExternal()) : null;
          if (uQ) await uQ; else if (uP) await uP;
          if (vP) t = await afResolve(vP);
          if (vQ) t = await afResolve(vQ);
        }
        return t;
      }
      const inExec = new Set();
      const wFunc = async (handler, wStore) => {
        try {
          const ct = Date.now();
          if (ct - timeupdateDT() < 800 && ct - wStore.dt < 800) {
            const cid = wStore.cid;
            inExec.add(cid);
            const t = await eFunc();
            const didNotRemove = inExec.delete(cid); // true for valid key
            if (!didNotRemove || t === wStore.lastExecution) return;
            wStore.lastExecution = t;
          }
          wStore.dt = ct;
          handler();
        } catch (e) {
          console.error(e);
          throw e;
        }
      };
      const sFunc = (propFunc) => {
        return (func, ms = 0, ...args) => {
          if (typeof func === 'function') { // ignore all non-function parameter (e.g. string)
            const wStore = { dt: Date.now() };
            return (wStore.cid = propFunc(wFunc, ms, (args.length > 0 ? func.bind(null, ...args) : func), wStore));
          } else {
            return propFunc(func, ms, ...args);
          }
        };
      };
      win.setTimeout = sFunc(setTimeout);
      win.setInterval = sFunc(setInterval);

      const dFunc = (propFunc) => {
        return (cid) => {
          if (cid) inExec.delete(cid) || propFunc(cid);
        };
      };

      win.clearTimeout = dFunc(clearTimeout);
      win.clearInterval = dFunc(clearInterval);

      try {
        win.setTimeout.toString = setTimeout.toString.bind(setTimeout);
        win.setInterval.toString = setInterval.toString.bind(setInterval);
        win.clearTimeout.toString = clearTimeout.toString.bind(clearTimeout);
        win.clearInterval.toString = clearInterval.toString.bind(clearInterval);
      } catch (e) { console.warn(e) }

    })();

    let mInterupter = null;
    setInterval(() => {
      if (mInterupter === afInterupter) {
        if (mInterupter !== null) afInterupter = mInterupter = (mInterupter(), null);
      } else {
        mInterupter = afInterupter;
      }
    }, 125);
  });

})(null);
```

</details><br>

Youtube Shorts to Videos:  
[Greasyfork](https://greasyfork.org/en/scripts/479883-youtube-shorts-to-traditional-video)
<details>
<summary>
Archive (1.13)
</summary>

```js
// ==UserScript==
// @name         Youtube Shorts to Traditional Video
// @namespace    http://tampermonkey.net/
// @version      1.13
// @description  Simple script to use traditional UI to view Youtube Shorts
// @author       unblue8914
// @match        https://www.youtube.com/*
// @grant        none
// @license      MIT
// @run-at document-start

// @description   Simple script to use traditional UI to view Youtube Shorts
// @downloadURL https://update.greasyfork.org/scripts/479883/Youtube%20Shorts%20to%20Traditional%20Video.user.js
// @updateURL https://update.greasyfork.org/scripts/479883/Youtube%20Shorts%20to%20Traditional%20Video.meta.js
// ==/UserScript==

(function() {
    'use strict';
    if (window.location.pathname.match('\/shorts\/.+')) {
        window.location.replace("https://www.youtube.com/watch?v=" + window.location.pathname.split('/shorts/').pop());
    }
    document.addEventListener("yt-navigate-start", (event) => {
        const url = event.detail.url.split('/shorts/');
        if(url.length > 1){
            window.location.replace("https://www.youtube.com/watch?v=" + url.pop());
        }
    });
})();
```

</details><br>

Show Password onMouseOver:  
[Greasyfork](https://greasyfork.org/en/scripts/32-show-password-onmouseover)
<details>
<summary>
Archive (1.0)
</summary>

```js
// ==UserScript==
// @name          Show Password onMouseOver
// @namespace     http://zoolcar9.lhukie.net/
// @include       *
// @description	  Show password when mouseover on password field
// @author        LouCypher
// @license       free
// @version 0.0.1.20140630034959
// @downloadURL https://update.greasyfork.org/scripts/32/Show%20Password%20onMouseOver.user.js
// @updateURL https://update.greasyfork.org/scripts/32/Show%20Password%20onMouseOver.meta.js
// ==/UserScript==

window.setTimeout(function() {
  var passFields = document.querySelectorAll("input[type='password']");
  if (!passFields.length) return;
  for (var i = 0; i < passFields.length; i++) {
    passFields[i].addEventListener("mouseover", function() {
      this.type = "text";
    }, false);
    passFields[i].addEventListener("mouseout", function() {
      this.type = "password";
    }, false);
  }
}, 1000)
```

</details><br>

osu! Expert+:  
[Github](https://github.com/inix1257/osu_expertplus)<br><br>

(osu!) Medals:  
[Greasyfork](https://greasyfork.org/en/scripts/577191-medals)
<details>
<summary>
Archive (1.0.4)
</summary>

```js
// ==UserScript==
// @name         Medals
// @namespace    http://tampermonkey.net/
// @version      1.0.4
// @description  medals
// @author       brandwagen
// @match        https://osu.ppy.sh/*
// @icon         https://www.google.com/s2/favicons?sz=64&domain=ppy.sh
// @grant        none
// @license      MIT
// @downloadURL https://update.greasyfork.org/scripts/577191/Medals.user.js
// @updateURL https://update.greasyfork.org/scripts/577191/Medals.meta.js
// ==/UserScript==

(function() {
    'use strict';

    function main() {
        let medalPercentage;
        let userId;
        const emotes = {"4687701": "👑", // dropinx
                        "13925852": "🖐️", // brandwagen
                        "17957276": "🖐️", // josh
                        "13688990": "🐱🖐️", // lunar
                        "10306849": "🖐️", // skrub
                        "9767342": "🖐️", // flyer
                        "17274052": "🍕", // magic
                        "12453848": "🍄", // glassive
                        "9939642": "🇲🇴", // shina
                        "13588932": "🥧", // pii
                        "19647032": "🤰", // kastracja
                        "11354436": "🩷", // utiba
                        "15538779": "🥕🐇", // velamy
                        "12565402": "🐈", // meiqth
                        "19244009": "👍🏻🙂", // yegg
                        "12233680": "🐢", // nyupenyu
                        "11394526": "💜", // sandro
                        "19117999": "🫃", // craftutopia
                        "7183087": "🍔🍔"} // abby

        try {
            const data = JSON.parse(document.querySelector('[data-react="profile-page"]').dataset.initialData);
            medalPercentage = data.user.user_achievements.length / data.achievements.length * 100;
            userId = data.user.id;
        } catch (error) {
            console.error("Medals: Error reading medals data from profile page JSON", error);
            return;
        }

        const nameElement = document.querySelector(".profile-info__name > span");
        if (nameElement == null) {
            console.error("Medals: Couldn't find medal or name element");
            return;
        }

        const titleElement = document.querySelector(".profile-info__title");
        if (titleElement != null) {
            if (titleElement.textContent == "Medal Hunter") {
                titleElement.textContent = "🏅 Medal Hunter 🏅";
                titleElement.style.color = getColor(-1);
            }
        }
        // construct name element
        if (!nameElement.textContent.includes("%")) {
            const player = nameElement.textContent;
            nameElement.textContent = '';

            // emote
            if (userId in emotes) {
                const emote = document.createElement('span');
                emote.textContent = `${emotes[userId]} `;
                nameElement.appendChild(emote);
            }

            if (medalPercentage === 100) {
                // cool epic style for completionists
                const styledUsername = createSpecialUsername(`${player} | ${medalPercentage.toFixed(2)}%`);
                nameElement.appendChild(styledUsername);
            } else {
                // adds back username + hyperlinked medal percentage
                nameElement.textContent += `${player} | `;
                nameElement.style.color = getColor(medalPercentage);
                nameElement.appendChild(getOsekaiLinkElement(userId, medalPercentage));
            }
        }
    }

    function getOsekaiLinkElement(id, p) {
        const link = document.createElement('a');
        link.href = `https://osekai.net/profiles/?user=${id}`;
        link.textContent = `${p.toFixed(2)}%`;
        link.target = '_blank';
        link.style.color = getColor(p);
        return link;
    }

    function getColor(p) {
        if (p > 98) { return '#fad82e'; }
        if (p > 95) { return '#f44257'; }
        if (p > 90) { return '#e44aff'; }
        if (p > 80) { return '#48c5eb'; }
        if (p > 60) { return '#51c097'; }
        if (p > 40) { return '#9fb5d8'; }
        if (p == -1) { return '#ffcf40'; }
        return '#9dbece';
    }

    const observer = new MutationObserver((records) => {
        for (const record of records) {
            for (const node of record.addedNodes) {
                if (!(node instanceof HTMLElement)) {
                    continue;
                }

                if (
                  node.classList.contains("profile-info__name") ||
                  node.querySelector(".profile-info__name") != null
                ) {
                  observer.disconnect();
                  main();
                }
            }
        }
    });

    // this is fully ai
    function createSpecialUsername(name) {
        const el = document.createElement('span');
        el.textContent = name;
        el.style.fontWeight = 'bold';
        el.style.background = 'linear-gradient(270deg, #60edf4, #b66aed, #495afa, #ff8c68)';
        el.style.backgroundSize = '800% 800%';
        el.style.webkitBackgroundClip = 'text';
        el.style.webkitTextFillColor = 'transparent';
        el.style.textShadow = '0 0 8px rgba(255,255,255,0.6)';
        el.style.animation = 'gradientMove 5s ease infinite';

        if (!document.getElementById('username-gradient-keyframes')) {
            const style = document.createElement('style');
            style.id = 'username-gradient-keyframes';
            style.textContent = `
                @keyframes gradientMove {
                    0% { background-position: 0% 50%; }
                    50% { background-position: 100% 50%; }
                    100% { background-position: 0% 50%; }
                }
            `;
            document.head.appendChild(style);
        }

        return el;
    }

    function onVisit(event) {
        observer.disconnect();

        if (
            event != null
                ? event.detail.url.includes("//osu.ppy.sh/users/")
                : window.location.pathname.startsWith("/users/")
        ) {
            observer.observe(document, {
                childList: true,
                subtree: true,
            });
        }
    }

    document.addEventListener("turbo:visit", onVisit);
    onVisit();
})();
```

</details><br>

Instagram Default Volume:  
[Greasyfork](https://greasyfork.org/en/scripts/487108-instagram-default-volume-with-volume-booster)
<details>
<summary>
Archive (2.0.8)
</summary>

```js
// ==UserScript==
// @name         instagram Default Volume (with Volume Booster)
// @namespace    instagramDefaultVolume
// @version      2.0.8
// @description  Set your Instagram videos default volumes (Up to 300%)
// @author       Runterya
// @homepage     https://github.com/Runteryaa
// @match        *://*.instagram.com/*
// @icon         https://www.google.com/s2/favicons?sz=64&domain=instagram.com
// @grant        none
// @license      MIT
// @compatible   chrome
// @compatible   edge
// @compatible   firefox
// @compatible   opera
// @compatible   safari
// @downloadURL https://update.greasyfork.org/scripts/487108/instagram%20Default%20Volume%20%28with%20Volume%20Booster%29.user.js
// @updateURL https://update.greasyfork.org/scripts/487108/instagram%20Default%20Volume%20%28with%20Volume%20Booster%29.meta.js
// ==/UserScript==

console.log("instagramDefaultVolume Loaded");

const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
const videoGainNodes = new WeakMap();

window.addEventListener('load', () => {
    document.addEventListener('click', () => {
        if (audioCtx.state === 'suspended') {
            audioCtx.resume();
        }
    }, { once: true });

    if (!localStorage.getItem('defaultVolume')) {
        localStorage.setItem('defaultVolume', 0.2);
    }

    const findVolumeDiv = () => {
        const targetElement = document.body;
        const volumeDiv = document.createElement('div');

        volumeDiv.id = 'igDefaultVolume';
        volumeDiv.style.position = 'fixed';
        volumeDiv.style.bottom = '1px';
        volumeDiv.style.right = '1px';
        volumeDiv.style.zIndex = '2147483647';

        volumeDiv.style.display = 'flex';
        volumeDiv.style.alignItems = 'center';
        volumeDiv.style.padding = '6px 10px';
        volumeDiv.style.borderRadius = '14px';
        volumeDiv.style.cursor = 'pointer';
        volumeDiv.style.backgroundColor = '#262626';
        volumeDiv.style.border = '1px solid #363636';
        volumeDiv.style.color = '#fff';
        volumeDiv.style.fontFamily = '-apple-system, system-ui, sans-serif';
        volumeDiv.style.fontSize = '12px';
        volumeDiv.style.boxShadow = '0 4px 12px rgba(0,0,0,0.5)';
        volumeDiv.style.transition = 'all 0.2s ease';

        volumeDiv.addEventListener('mouseenter', () => volumeDiv.style.transform = 'scale(1.05)');
        volumeDiv.addEventListener('mouseleave', () => volumeDiv.style.transform = 'scale(1)');

        const svgIcon = document.createElementNS("http://www.w3.org/2000/svg", "svg");
        svgIcon.setAttribute("width", "20");
        svgIcon.setAttribute("height", "20");
        svgIcon.setAttribute("viewBox", "0 0 24 24");
        svgIcon.style.fill = "#ffffff";
        svgIcon.style.marginRight = "8px";
        svgIcon.style.cursor = "pointer";

        const titleElement = document.createElementNS("http://www.w3.org/2000/svg", "title");
        titleElement.textContent = "Click to reset to 100%";
        svgIcon.appendChild(titleElement);

        const iconPath = document.createElementNS("http://www.w3.org/2000/svg", "path");
        svgIcon.appendChild(iconPath);

        const ICON_PATHS = {
            mute: "M16.5 12c0-1.77-1.02-3.29-2.5-4.03v2.21l2.45 2.45c.03-.2.05-.41.05-.63zm2.5 0c0 .94-.2 1.82-.54 2.64l1.51 1.51C20.63 14.91 21 13.5 21 12c0-4.28-2.99-7.86-7-8.77v2.06c2.89.86 5 3.54 5 6.71zM4.27 3L3 4.27 7.73 9H3v6h4l5 5v-6.73l4.25 4.25c-.67.52-1.42.93-2.25 1.18v2.06c1.38-.31 2.63-.95 3.69-1.81L19.73 21 21 19.73 4.27 3zM12 4L9.91 6.09 12 8.18V4z",
            low: "M18.5 12c0-1.77-1.02-3.29-2.5-4.03v8.05c1.48-.73 2.5-2.25 2.5-4.02zM5 9v6h4l5 5V4L9 9H5z",
            high: "M3 9v6h4l5 5V4L7 9H3zm13.5 3c0-1.77-1.02-3.29-2.5-4.03v8.05c1.48-.73 2.5-2.25 2.5-4.02zM14 3.23v2.06c2.89.86 5 3.54 5 6.71s-2.11 5.85-5 6.71v2.06c4.01-.91 7-4.49 7-8.77s-2.99-7.86-7-8.77z"
        };

        const volumeBtn = document.createElement('button');
        volumeBtn.id = 'volumeBtn';
        volumeBtn.style.display = 'flex';
        volumeBtn.style.alignItems = 'center';
        volumeBtn.style.background = 'transparent';
        volumeBtn.style.border = 'none';
        volumeBtn.style.color = 'inherit';
        volumeBtn.style.padding = '0';
        volumeBtn.style.cursor = 'pointer';

        const volumeSelectorInput = document.createElement('input');
        volumeSelectorInput.type = 'range';

        const SPLIT_POINT = 100;
        const MAX_POINT = 130;

        volumeSelectorInput.min = 0;
        volumeSelectorInput.max = MAX_POINT;
        volumeSelectorInput.step = 1;

        const currentVol = parseFloat(localStorage.getItem('defaultVolume')) || 1;
        if (currentVol <= 1) {
            volumeSelectorInput.value = currentVol * 100;
        } else {
            volumeSelectorInput.value = 100 + ((currentVol - 1) * 15);
        }

        volumeSelectorInput.style.display = 'none';
        volumeSelectorInput.style.width = '100px';
        volumeSelectorInput.style.marginLeft = '10px';
        volumeSelectorInput.style.cursor = 'ew-resize';
        volumeSelectorInput.style.appearance = 'none';
        volumeSelectorInput.style.background = 'transparent';

        const styleSheet = document.createElement("style");
        styleSheet.textContent = `
            #igDefaultVolume input[type=range]::-webkit-slider-thumb {
                -webkit-appearance: none;
                height: 12px;
                width: 12px;
                border-radius: 50%;
                background: #ffffff;
                margin-top: -4px;
                box-shadow: 0 1px 3px rgba(0,0,0,0.5);
                transition: transform 0.1s;
            }
            #igDefaultVolume input[type=range]:active::-webkit-slider-thumb {
                transform: scale(1.3);
            }
            #igDefaultVolume input[type=range]::-webkit-slider-runnable-track {
                width: 100%;
                height: 4px;
                cursor: pointer;
                background: #555;
                border-radius: 2px;
            }
        `;
        document.head.appendChild(styleSheet);

        const volumeSelectorText = document.createElement('span');
        volumeSelectorText.style.minWidth = '40px';
        volumeSelectorText.style.textAlign = 'center';
        volumeSelectorText.style.fontWeight = '700';

        const updateSliderVisuals = (val) => {
            const max = parseInt(volumeSelectorInput.max);
            const visualFillLimit = Math.min(val, SPLIT_POINT);
            const percentage = (visualFillLimit / max) * 100;

            let color = '#0095f6';
            if (val > SPLIT_POINT) color = '#ff4d4d';

            volumeSelectorInput.style.background = `linear-gradient(to right, ${color} 0%, ${color} ${percentage}%, #555 ${percentage}%, #555 100%)`;
        };

        const updateVolumeLogic = () => {
            let val = parseFloat(volumeSelectorInput.value);
            let realVolume = 0;
            let displayPercent = 0;

            if (val <= SPLIT_POINT) {
                realVolume = val / 100;
                displayPercent = Math.round(realVolume * 100);
            } else {
                const boostProgress = (val - SPLIT_POINT) / (MAX_POINT - SPLIT_POINT);
                realVolume = 1 + (boostProgress * 2);
                displayPercent = Math.round(realVolume * 100);
            }

            volumeSelectorText.textContent = displayPercent + '%';

            if (realVolume === 0) {
                iconPath.setAttribute("d", ICON_PATHS.mute);
            } else if (realVolume < 0.5) {
                iconPath.setAttribute("d", ICON_PATHS.low);
            } else {
                iconPath.setAttribute("d", ICON_PATHS.high);
            }

            if (realVolume > 1) {
                volumeSelectorText.style.color = '#ff4d4d';
            } else {
                volumeSelectorText.style.color = 'inherit';
            }
            svgIcon.style.fill = '#ffffff';

            updateSliderVisuals(val);
            localStorage.setItem('defaultVolume', realVolume);
            setVolumeForVideos();
        };

        updateVolumeLogic();

        volumeSelectorInput.addEventListener('input', updateVolumeLogic);

        svgIcon.addEventListener('click', (e) => {
            e.stopPropagation();
            volumeSelectorInput.value = 100;
            updateVolumeLogic();
        });

        volumeDiv.appendChild(svgIcon);
        volumeDiv.appendChild(volumeBtn);
        volumeBtn.appendChild(volumeSelectorText);
        volumeBtn.appendChild(volumeSelectorInput);

        let isExpanded = false;

        volumeDiv.addEventListener('click', (event) => {
            event.stopPropagation();
            if (event.target !== volumeSelectorInput) {
                isExpanded = !isExpanded;
                if (isExpanded) {
                    volumeSelectorInput.style.display = 'block';
                    updateSliderVisuals(volumeSelectorInput.value);
                } else {
                    volumeSelectorInput.style.display = 'none';
                }
            }
        });

        targetElement.appendChild(volumeDiv);
    };

    setInterval(() => {
        if (!document.getElementById('igDefaultVolume')) {
            findVolumeDiv();
        }
    }, 1000);

    const setVolumeForVideos = () => {
        const desiredVolume = parseFloat(localStorage.getItem('defaultVolume'));
        const videos = document.getElementsByTagName('video');

        for (let video of videos) {
            try {
                if (desiredVolume <= 1.0) {
                    video.volume = desiredVolume;
                    if (videoGainNodes.has(video)) {
                        videoGainNodes.get(video).gain.value = 1;
                    }
                } else {
                    video.volume = 1.0;
                    if (!videoGainNodes.has(video)) {
                        const source = audioCtx.createMediaElementSource(video);
                        const gainNode = audioCtx.createGain();
                        source.connect(gainNode);
                        gainNode.connect(audioCtx.destination);
                        videoGainNodes.set(video, gainNode);
                    }
                    const gainNode = videoGainNodes.get(video);
                    gainNode.gain.value = desiredVolume;
                }
            } catch (err) {
                 if (desiredVolume > 1) video.volume = 1;
                 else video.volume = desiredVolume;
            }
        }
    };

    setVolumeForVideos();
    new MutationObserver(() => setVolumeForVideos()).observe(document.body, { childList: true, subtree: true });
});
```

</details><br>

Hypixel Reward Skip:  
<details>
<summary>
Archive (1.0)
</summary>

```js
// ==UserScript==
// @name         Hypixel Reward Skip
// @namespace    https://google.com/
// @version      1.0
// @description  Automatically skips the Hypixel Daily Reward video for ranked members.
// @author       SurprisedKetchup
// @match        https://rewards.hypixel.net/claim-reward/*
// @require      http://code.jquery.com/jquery-latest.js
// @grant        none
// ==/UserScript==

window.setInterval(function(){
    $(".index__skipButton___3ihHt").click();
}, 100);
```

</details><br>

## Activadas a veces

YoutubeSortByDuration:  
[Greasyfork](https://greasyfork.org/en/scripts/530129-youtubesortbyduration)
[Github](https://github.com/cloph-dsp/YouTubeSortByDuration)
<details>
<summary>
Archive (4.2)
</summary>

```js
// ==UserScript==
// @name              YouTubeSortByDuration
// @namespace         https://github.com/cloph-dsp/YouTubeSortByDuration
// @version           4.2
// @description       Supercharges your playlist management by sorting videos by duration.
// @author            cloph-dsp
// @originalAuthor    KohGeek
// @license           GPL-2.0-only
// @homepageURL       https://github.com/cloph-dsp/YouTubeSortByDuration
// @supportURL        https://github.com/cloph-dsp/YouTubeSortByDuration/issues
// @match             http://*.youtube.com/*
// @match             https://*.youtube.com/*
// @require           https://greasyfork.org/scripts/374849-library-onelementready-es7/code/Library%20%7C%20onElementReady%20ES7.js
// @grant             none
// @run-at            document-start
// @downloadURL https://update.greasyfork.org/scripts/530129/YouTubeSortByDuration.user.js
// @updateURL https://update.greasyfork.org/scripts/530129/YouTubeSortByDuration.meta.js
// ==/UserScript==

// CSS styles
const css = `
    /* Container wrapper */
    .sort-playlist {
        display: flex;
        flex-wrap: wrap;
        align-items: center;
        gap: 16px;
        padding: 12px;
        background-color: var(--yt-spec-base-background);
        border-bottom: 1px solid var(--yt-spec-10-percent-layer);
        width: 100%;
        box-sizing: border-box;
    }
    /* Controls grouping */
    .sort-playlist-controls {
        display: flex;
        align-items: center;
        gap: 8px;
    }
    /* Sort button wrapper */
    #sort-toggle-button {
        padding: 8px 16px;
        font-size: 14px;
        white-space: nowrap;
        cursor: pointer;
        background: none;
        outline: none;
    }
    /* Start (green) state */
    .sort-button-start {
        background-color: #28a745;
        color: #fff;
        border: 1px solid #28a745;
        border-radius: 4px;
        transition: background-color 0.3s, transform 0.2s;
    }
    .sort-button-start:hover {
        background-color: #218838;
        transform: translateY(-1px);
    }
    /* Stop (red) state */
    .sort-button-stop {
        background-color: #dc3545;
        color: #fff;
        border: 1px solid #dc3545;
        border-radius: 4px;
        transition: background-color 0.3s, transform 0.2s;
    }
    .sort-button-stop:hover {
        background-color: #c82333;
        transform: translateY(-1px);
    }
    /* Dropdown selector styling */
    .sort-select {
        padding: 6px 12px;
        font-size: 14px;
        border: 1px solid var(--yt-spec-10-percent-layer);
        border-radius: 4px;
        background-color: var(--yt-spec-base-background);
        color: var(--yt-spec-text-primary); /* ensure text is visible */
        transition: border-color 0.2s;
    }
    .sort-select option { color: var(--yt-spec-text-primary); }
    .sort-select:focus {
        border-color: var(--yt-spec-brand-link-text);
        box-shadow: 0 0 0 2px rgba(40, 167, 69, 0.3);
        outline: none;
    }
    /* Status log element */
    .sort-log {
        margin-left: auto;
        padding: 6px 12px;
        font-size: 13px;
        background-color: var(--yt-spec-base-background);
        border: 1px solid var(--yt-spec-10-percent-layer);
        border-radius: 4px;
        color: var(--yt-spec-text-primary);
        white-space: nowrap;
    }
`

const modeAvailable = [
    { value: 'asc', label: 'by Shortest' },
    { value: 'desc', label: 'by Longest' }
];

const debug = false;

// Config
let isSorting = false;
let scrollLoopTime = 500;
let useAdaptiveDelay = true;
let baseDelayPerVideo = 5;
let minDelay = 10;
let maxDelay = 1000;
let fastModeThreshold = 200;

let sortMode = 'asc';
let autoScrollInitialVideoList = true;
let log = document.createElement('div');
let stopSort = false;

// Fire mouse event at coordinates
let fireMouseEvent = (type, elem, centerX, centerY) => {
    const event = new MouseEvent(type, {
        view: window,
        bubbles: true,
        cancelable: true,
        clientX: centerX,
        clientY: centerY
    });

    elem.dispatchEvent(event);
};

// Simulate drag between elements
let simulateDrag = (elemDrag, elemDrop) => {
    // Get positions
    let pos = elemDrag.getBoundingClientRect();
    let center1X = Math.floor((pos.left + pos.right) / 2);
    let center1Y = Math.floor((pos.top + pos.bottom) / 2);
    pos = elemDrop.getBoundingClientRect();
    let center2X = Math.floor((pos.left + pos.right) / 2);
    let center2Y = Math.floor((pos.top + pos.bottom) / 2);

    // Mouse events for dragged element
    fireMouseEvent("mousemove", elemDrag, center1X, center1Y);
    fireMouseEvent("mouseenter", elemDrag, center1X, center1Y);
    fireMouseEvent("mouseover", elemDrag, center1X, center1Y);
    fireMouseEvent("mousedown", elemDrag, center1X, center1Y);

    // Start drag
    fireMouseEvent("dragstart", elemDrag, center1X, center1Y);
    fireMouseEvent("drag", elemDrag, center1X, center1Y);
    fireMouseEvent("mousemove", elemDrag, center1X, center1Y);
    fireMouseEvent("drag", elemDrag, center2X, center2Y);
    fireMouseEvent("mousemove", elemDrop, center2X, center2Y);

    // Events over drop target
    fireMouseEvent("mouseenter", elemDrop, center2X, center2Y);
    fireMouseEvent("dragenter", elemDrop, center2X, center2Y);
    fireMouseEvent("mouseover", elemDrop, center2X, center2Y);
    fireMouseEvent("dragover", elemDrop, center2X, center2Y);

    // Complete drop
    fireMouseEvent("drop", elemDrop, center2X, center2Y);
    fireMouseEvent("dragend", elemDrag, center2X, center2Y);
    fireMouseEvent("mouseup", elemDrag, center2X, center2Y);
};

// Scroll to position or page bottom
let autoScroll = async (scrollTop = null) => {
    const element = document.scrollingElement;
    const destination = scrollTop !== null ? scrollTop : element.scrollHeight;
    element.scrollTop = destination;
    logActivity(`Scrolling page... 📜`);
    await new Promise(r => setTimeout(r, scrollLoopTime));
};

// Log message to UI
let logActivity = (message) => {
    if (log) {
        log.innerText = message;
    }
    if (debug) {
        console.log(message);
    }
};

// Create UI container
let renderContainerElement = () => {
    // Remove old container if any
    const existing = document.querySelector('.sort-playlist');
    if (existing) existing.remove();

    // Create new container
    const element = document.createElement('div');
    element.className = 'sort-playlist';
    element.style.margin = '8px 0';

    const controls = document.createElement('div');
    controls.className = 'sort-playlist-controls';
    element.appendChild(controls);

    // Insert above playlist video list
    const list = document.querySelector('ytd-playlist-video-list-renderer');
    if (list && list.parentNode) {
        list.parentNode.insertBefore(element, list);
    } else {
        console.error('Playlist video list not found for UI injection');
    }
};

// Create sort toggle button
let renderToggleButton = () => {
    const element = document.createElement('button');
    element.id = 'sort-toggle-button';
    element.className = 'style-scope sort-button-toggle sort-button-start';
    element.innerText = 'Sort Videos';
    
    element.onclick = async () => {
        if (!isSorting) {
            // Start
            isSorting = true;
            element.className = 'style-scope sort-button-toggle sort-button-stop';
            element.innerText = 'Stop Sorting';
            await activateSort();
            // Reset
            isSorting = false;
            element.className = 'style-scope sort-button-toggle sort-button-start';
            element.innerText = 'Sort Videos';
        } else {
            // Stop
            stopSort = true;
            isSorting = false;
            element.className = 'style-scope sort-button-toggle sort-button-start';
            element.innerText = 'Sort Videos';
        }
    };

    document.querySelector('.sort-playlist-controls').appendChild(element);
};

// Create dropdown selector
let renderSelectElement = (variable = 0, options = [], label = '') => {
    const element = document.createElement('select');
    element.className = 'style-scope sort-select';
    element.onchange = (e) => {
        if (variable === 0) {
            sortMode = e.target.value;
        } else if (variable === 1) {
            autoScrollInitialVideoList = e.target.value;
        }
    };

    options.forEach((option) => {
        const optionElement = document.createElement('option')
        optionElement.value = option.value
        optionElement.innerText = option.label
        element.appendChild(optionElement)
    });

    document.querySelector('.sort-playlist-controls').appendChild(element);
};

// Create number input
let renderNumberElement = (defaultValue = 0, label = '') => {
    const elementDiv = document.createElement('div');
    elementDiv.className = 'sort-playlist-div sort-margin-right-3px';
    elementDiv.innerText = label;

    const element = document.createElement('input');
    element.type = 'number';
    element.value = defaultValue;
    element.className = 'style-scope';
    element.oninput = (e) => { scrollLoopTime = +(e.target.value) };

    elementDiv.appendChild(element);
    document.querySelector('div.sort-playlist').appendChild(elementDiv);
};

// Create status log display
let renderLogElement = () => {
    log.className = 'style-scope sort-log';
    log.innerText = 'Please wait for the playlist to fully load before sorting...';
    document.querySelector('div.sort-playlist').appendChild(log);
};

// Add CSS to page
let addCssStyle = () => {
    const element = document.createElement('style');
    element.textContent = css;
    document.head.appendChild(element);
};

// Wait for YouTube to process drag
let waitForYoutubeProcessing = async () => {
    const maxWaitTime = Math.max(scrollLoopTime * 3, 1000);
    const startTime = Date.now();
    let isProcessed = false;
    
    logActivity(`Rearranging playlist... ⏳`);
    
    // Fast polling
    const pollInterval = Math.min(50, Math.max(25, scrollLoopTime * 0.1));
    
    while (!isProcessed && Date.now() - startTime < maxWaitTime && !stopSort) {
        await new Promise(r => setTimeout(r, pollInterval));
        
        const allDragPoints = document.querySelectorAll("ytd-item-section-renderer:first-of-type yt-icon#reorder");
        const currentItems = document.querySelectorAll("ytd-item-section-renderer:first-of-type div#content a#thumbnail.inline-block.ytd-thumbnail");
        
        if (allDragPoints.length > 0 && currentItems.length > 0 && 
            document.querySelectorAll('.ytd-continuation-item-renderer').length === 0) {
            isProcessed = true;
            const timeTaken = Date.now() - startTime;
            if (timeTaken > 1000) {
                logActivity(`Video moved successfully ✓`);
            }
            
            // Stabilization wait
            await new Promise(r => setTimeout(r, Math.min(180, scrollLoopTime * 0.25)));
            return true;
        }
    }
    
    // Recovery methods

    // Fast recovery approach
    if (!isProcessed) {
        logActivity(`Ensuring YouTube updates... ⏱️`);
        
        // Click body to refocus
        document.body.click();
        await new Promise(r => setTimeout(r, 275));
        
        // Check results
        let updatedDragPoints = document.querySelectorAll("ytd-item-section-renderer:first-of-type yt-icon#reorder");
        let allThumbnails = document.querySelectorAll("ytd-item-section-renderer:first-of-type div#content a#thumbnail.inline-block.ytd-thumbnail");
        
        if (updatedDragPoints.length > 0 && allThumbnails.length > 0) {
            logActivity(`Recovered playlist view ✓`);
            await new Promise(r => setTimeout(r, 540));
            
            // Force refresh
            window.dispatchEvent(new Event('resize'));
            await new Promise(r => setTimeout(r, 180));
            
            return true;
        }
        
        // Try faster recovery (2 attempts)
        for (let i = 0; i < 2 && !stopSort; i++) {
            const areas = [
                document.querySelector('.playlist-items'), 
                document.body
            ];
            
            // Click areas
            for (const area of areas) {
                if (area && !stopSort) {
                    area.click();
                    await new Promise(r => setTimeout(r, 125));
                }
            }
            
            // Slight scroll
            if (!stopSort) {
                const scrollElem = document.scrollingElement;
                const currentPos = scrollElem.scrollTop;
                scrollElem.scrollTop = currentPos + 10; 
                await new Promise(r => setTimeout(r, 125));
                scrollElem.scrollTop = currentPos;
            }
            
            // Check if successful
            updatedDragPoints = document.querySelectorAll("ytd-item-section-renderer:first-of-type yt-icon#reorder");
            allThumbnails = document.querySelectorAll("ytd-item-section-renderer:first-of-type div#content a#thumbnail.inline-block.ytd-thumbnail");
            
            if (updatedDragPoints.length > 0 && allThumbnails.length > 0) {
                logActivity(`Recovered after attempt ${i+1} ✓`);
                await new Promise(r => setTimeout(r, (540 + i * 125)));
                
                // Force refresh
                window.dispatchEvent(new Event('resize'));
                await new Promise(r => setTimeout(r, 175));
                
                return true;
            }
            
            await new Promise(r => setTimeout(r, 315 + i * 175));
        }
        
        // Final fallback
        if (!stopSort) {
            logActivity(`Attempting final recovery method...`);
            
            document.body.click();
            await new Promise(r => setTimeout(r, 225));
            
            const playlistHeader = document.querySelector('ytd-playlist-header-renderer');
            if (playlistHeader) playlistHeader.click();
            
            await new Promise(r => setTimeout(r, 725));
            // Final check
            updatedDragPoints = document.querySelectorAll("ytd-item-section-renderer:first-of-type yt-icon#reorder");
            allThumbnails = document.querySelectorAll("ytd-item-section-renderer:first-of-type div#content a#thumbnail.inline-block.ytd-thumbnail");
            if (updatedDragPoints.length > 0 && allThumbnails.length > 0) {
                logActivity(`Final recovery successful ✓`);
                await new Promise(r => setTimeout(r, 700)); 
                // Force refresh
                window.dispatchEvent(new Event('resize'));
                await new Promise(r => setTimeout(r, 175));
                return true;
            }
            logActivity(`Recovery failed - skipping operation`);
            return false;
        }
    }
    return isProcessed;
};

// Check if playlist is fully loaded
let isPlaylistFullyLoaded = (reportedCount, loadedCount) => {
    const hasSpinner = document.querySelector('.ytd-continuation-item-renderer') !== null;
    return reportedCount === loadedCount && !hasSpinner;
};

// Check if YouTube page is ready
let isYouTubePageReady = () => {
    const playlistContainer = document.querySelector("ytd-playlist-video-list-renderer");
    const videoCountElement = document.querySelector("ytd-playlist-sidebar-primary-info-renderer #stats span:first-child");
    const dragPoints = playlistContainer ? playlistContainer.querySelectorAll("yt-icon#reorder") : [];
    
    // Spinner detection
    const spinnerElements = document.querySelectorAll('.ytd-continuation-item-renderer, yt-icon-button.ytd-continuation-item-renderer, .circle.ytd-spinner');
    const hasVisibleSpinner = Array.from(spinnerElements).some(el => {
        const rect = el.getBoundingClientRect();
        return (rect.width > 0 && rect.height > 0 && 
                rect.top >= 0 && rect.top <= window.innerHeight);
    });
    
    const reportedCount = videoCountElement ? parseInt(videoCountElement.innerText, 10) : 0;
    const loadedCount = dragPoints.length;
    const youtubeLoadLimit = 100;
    
    // Check readiness
    const basicElementsReady = playlistContainer && videoCountElement && reportedCount > 0 && loadedCount > 0;
    const hasEnoughVideos = loadedCount >= 95 || loadedCount === reportedCount;
    const fullyLoaded = (!hasVisibleSpinner && hasEnoughVideos) || 
                         (loadedCount >= 0.97 * Math.min(reportedCount, youtubeLoadLimit));
    
    const isReady = basicElementsReady && fullyLoaded;
    
    // Show status
    if (isReady) {
        if (loadedCount < reportedCount) {
            logActivity(`✅ Ready: ${loadedCount}/${reportedCount} videos (YT limit)`);
        } else {
            logActivity(`✅ Ready: ${loadedCount}/${reportedCount} videos`);
        }
    } else if (basicElementsReady) {
        if (hasVisibleSpinner) {
            logActivity(`⏳ Loading: ${loadedCount}/${reportedCount} videos`);
        } else if (loadedCount < reportedCount && loadedCount > 0) {
            logActivity(`🔄 Waiting to scroll: ${loadedCount}/${reportedCount} videos`);
        } else {
            logActivity(`🔄 Waiting: ${loadedCount}/${reportedCount || '?'} videos`);
        }
    } else {
        logActivity(`🔄 Initializing...`);
    }
    return isReady;
};

let sortVideos = async (allAnchors, allDragPoints, expectedCount) => {
    // Verify playlist fully loaded
    if (allDragPoints.length !== expectedCount || allAnchors.length !== expectedCount) {
        logActivity("Playlist not fully loaded, waiting...");
        return -1;
    }
    // Build list of current handles with durations
    let list = [];
    for (let i = 0; i < expectedCount; i++) {
        // Handle missing duration text gracefully
        const txtElem = allAnchors[i].querySelector('#text');
        let timeSp = txtElem ? txtElem.innerText.trim().split(':').reverse() : [''];
        let t = timeSp.length === 1 ? (sortMode === 'asc' ? Infinity : -Infinity)
                : parseInt(timeSp[0]) + (timeSp[1] ? parseInt(timeSp[1]) * 60 : 0) + (timeSp[2] ? parseInt(timeSp[2]) * 3600 : 0);
        list.push({ handle: allDragPoints[i], time: t });
    }
    // Create sorted reference
    let sorted = [...list];
    sorted.sort((a, b) => sortMode === 'asc' ? a.time - b.time : b.time - a.time);
    // Find first mismatch and move
    for (let i = 0; i < expectedCount; i++) {
        if (list[i].handle !== sorted[i].handle) {
            let elemDrag = sorted[i].handle;
            let elemDrop = list[i].handle;
            logActivity(`Dragging video to position ${i}`);
            try {
                simulateDrag(elemDrag, elemDrop);
                if (useAdaptiveDelay && expectedCount > fastModeThreshold) {
                    // Fast static delay for large playlists
                    await new Promise(r => setTimeout(r, 100));
                } else {
                    await waitForYoutubeProcessing();
                }
            } catch (e) {
                console.error('Drag error:', e);
                logActivity('Error during move; retrying slowly... ⏳');
                // Fallback delay and retry
                await new Promise(r => setTimeout(r, scrollLoopTime));
            }
             // Return number of sorted items (index+1) to signal success
             return i + 1;
        }
    }
    // All in order
    return expectedCount;
}

// Main sorting function
let activateSort = async () => {
    // Reset scroll cap
    autoScrollAttempts = 0;
     // Set manual sorting mode
    const ensureManualSort = async () => {
        const sortButton = document.querySelector('yt-dropdown-menu[icon-label="Ordenar"] tp-yt-paper-button, yt-dropdown-menu[icon-label="Sort"] tp-yt-paper-button');
        if (!sortButton) {
            logActivity("Sort dropdown not found. Using current mode.");
            return;
        }
        
        // Check if dropdown is already open and close it first (clean state)
        const isDropdownOpen = document.querySelector('tp-yt-paper-listbox:not([hidden])');
        if (isDropdownOpen) {
            // Click away to close the dropdown
            document.body.click();
            await new Promise(r => setTimeout(r, 100));
        }
        
        // Open the dropdown menu
        logActivity("Opening sort dropdown...");
        sortButton.click();
        await new Promise(r => setTimeout(r, 200));
        
        // Verify dropdown is visible
        const dropdownMenu = document.querySelector('tp-yt-paper-listbox:not([hidden])');
        if (!dropdownMenu) {
            // Try once more if the dropdown didn't appear
            sortButton.click();
            await new Promise(r => setTimeout(r, 250));
        }
        
        // Ensure the dropdown is visible and select the manual option
        const manualOption = document.querySelector('tp-yt-paper-listbox a:first-child tp-yt-paper-item');
        if (manualOption) {
            // Check if already selected to avoid unnecessary clicks
            const isSelected = manualOption.hasAttribute('selected') || 
                              manualOption.classList.contains('iron-selected') ||
                              manualOption.getAttribute('aria-selected') === 'true';
            
            if (!isSelected) {
                manualOption.click();
                logActivity("Switched to Manual sort mode");
                await new Promise(r => setTimeout(r, 250));
            } else {
                logActivity("Manual sort mode already active");
            }
            
            // Ensure dropdown is closed by clicking away if still open
            const stillOpen = document.querySelector('tp-yt-paper-listbox:not([hidden])');
            if (stillOpen) {
                document.body.click();
                await new Promise(r => setTimeout(r, 100));
            }
            
            // Quick verification
            const verifySort = document.querySelector('.dropdown-trigger-text');
            if (verifySort && verifySort.textContent.toLowerCase().includes('manual')) {
                logActivity("Manual sort mode confirmed ✓");
            }
            
            return true;
        } else {
            // Fallback method if first item not found
            const allOptions = document.querySelectorAll('tp-yt-paper-listbox a tp-yt-paper-item');
            for (const option of allOptions) {
                if (option.textContent.toLowerCase().includes('manual')) {
                    option.click();
                    logActivity("Found and selected Manual sort mode");
                    await new Promise(r => setTimeout(r, 250));
                    
                    // Close dropdown
                    document.body.click();
                    return true;
                }
            }
            
            // Close dropdown if option not found
            document.body.click();
            logActivity("Manual sort option not found. Using current mode.");
            return false;
        }
    };

    const manualSortSet = await ensureManualSort();
    const videoCountElement = document.querySelector("ytd-playlist-sidebar-primary-info-renderer #stats span:first-child");
    let reportedVideoCount = videoCountElement ? parseInt(videoCountElement.innerText, 10) : 0;
    const playlistContainer = document.querySelector("ytd-playlist-video-list-renderer");
    let allDragPoints = playlistContainer ? playlistContainer.querySelectorAll("yt-icon#reorder") : [];
    let allAnchors;
    
    // Set optimal delay based on playlist size
    if (useAdaptiveDelay) {
        const needsScrolling = reportedVideoCount > 95 && allDragPoints.length < reportedVideoCount && autoScrollInitialVideoList;
        
        if (needsScrolling) {
            scrollLoopTime = Math.max(500, minDelay * 8);
            logActivity(`Using scroll-safe speed (${scrollLoopTime}ms)`);
        } else {
            let videoCount = reportedVideoCount || allDragPoints.length;
            
            if (videoCount <= fastModeThreshold) {
                let fastDelay = Math.max(75, baseDelayPerVideo * Math.sqrt(videoCount * 0.75));
                scrollLoopTime = Math.min(fastDelay, 350);
                logActivity(`Using fast mode: ${scrollLoopTime}ms}`);
            } else {
                let calculatedDelay = Math.max(100, baseDelayPerVideo * Math.log(videoCount) * 2.5);
                scrollLoopTime = Math.min(calculatedDelay, 800);
                logActivity(`Using adaptive delay: ${scrollLoopTime}ms}`);
            }
        }
    }
    
    let sortedCount = 0;
    let initialVideoCount = allDragPoints.length;
    let scrollRetryCount = 0;
    stopSort = false;
    let consecutiveRecoveryFailures = 0;
    let sortFailureCount = 0; // count sortVideos failures

    // Load all videos
    if (reportedVideoCount > allDragPoints.length && autoScrollInitialVideoList) {
        logActivity(`Playlist has ${reportedVideoCount} videos. Loading all...`);
        while (allDragPoints.length < reportedVideoCount && !stopSort && scrollRetryCount < 10) {
            await autoScroll();
            let newDragPoints = playlistContainer ? playlistContainer.querySelectorAll("yt-icon#reorder") : [];
            
            if (newDragPoints.length > allDragPoints.length) {
                allDragPoints = newDragPoints;
                scrollRetryCount = 0; // Reset on progress
                logActivity(`Loading videos (${allDragPoints.length}/${reportedVideoCount})`);
            } else {
                scrollRetryCount++;
                logActivity(`Scroll attempt ${scrollRetryCount}/10...`);
                await new Promise(r => setTimeout(r, 500 + scrollRetryCount * 100));
            }

            // Check for spinner
            const spinner = document.querySelector('.ytd-continuation-item-renderer');
            if (!spinner && allDragPoints.length < reportedVideoCount) {
                logActivity(`No spinner found, but not all videos loaded. Retrying...`);
                await new Promise(r => setTimeout(r, 1000));
            } else if (!spinner) {
                break; // Exit if no spinner and no new videos
            }
        }
    }


    initialVideoCount = allDragPoints.length;
    logActivity(initialVideoCount + " videos loaded for sorting.");
    if (scrollRetryCount >= 10) {
        logActivity("Max scroll attempts reached. Proceeding with available videos.");
    }
    
    let loadedLocation = document.scrollingElement.scrollTop;
    scrollRetryCount = 0;

    // Sort videos
    const sortStartTime = Date.now();
    const maxSortTime = 900000; // 15 minutes max

    // Check timeout
    if (Date.now() - sortStartTime > maxSortTime) {
        logActivity("Sorting timed out after 15 minutes.");
        return;
    }
    
    // Stall detection: break if no progress after multiple cycles
    let lastSortedCount = -1;
    let stallCount = 0;
    while (sortedCount < initialVideoCount && stopSort === false) {
        if (sortedCount === lastSortedCount) {
            stallCount++;
        } else {
            stallCount = 0;
            lastSortedCount = sortedCount;
        }
        if (stallCount >= 3) {
            logActivity('No further progress; ending sort to avoid hang');
            break;
        }
        
        sortFailureCount = 0;
        // Check timeout
        if (Date.now() - sortStartTime > maxSortTime) {
            logActivity("Sorting timed out after 15 minutes.");
            return;
        }
        
        // Reset after recovery failures
        if (consecutiveRecoveryFailures >= 3) {
            logActivity("Too many failures. Reattempting...");
            await new Promise(r => setTimeout(r, 1500));
            consecutiveRecoveryFailures = 0;
        }
        allDragPoints = playlistContainer ? playlistContainer.querySelectorAll("yt-icon#reorder") : [];
        allAnchors = playlistContainer ? playlistContainer.querySelectorAll("div#content a#thumbnail.inline-block.ytd-thumbnail") : [];
        scrollRetryCount = 0;

        // Ensure durations loaded (up to 3 auto-scroll attempts)
        let detailRetries = 0;
        while (!allAnchors[initialVideoCount - 1]?.querySelector("#text") && !stopSort && detailRetries < 3) {
            logActivity(`Loading video details... attempt ${detailRetries + 1}`);
            await autoScroll();
            // Refresh references
            allDragPoints = playlistContainer ? playlistContainer.querySelectorAll("yt-icon#reorder") : [];
            allAnchors = playlistContainer ? playlistContainer.querySelectorAll("div#content a#thumbnail.inline-block.ytd-thumbnail") : [];
            detailRetries++;
        }
        if (detailRetries >= 3) {
            logActivity("Proceeding without full duration details...");
            // Update expected count to actual loaded elements to avoid sort blocking
            initialVideoCount = allAnchors.length;
            allDragPoints = playlistContainer.querySelectorAll("yt-icon#reorder");
        }

         // Sort if elements available
         if (allAnchors.length > 0 && allDragPoints.length > 0) {
            // Perform sorting; negative indicates missing durations
            const res = await sortVideos(allAnchors, allDragPoints, initialVideoCount);
            if (res < 0) {
                sortFailureCount++;
                if (sortFailureCount >= 3) {
                    logActivity('Unable to load durations after multiple attempts; aborting sort');
                    sortedCount = initialVideoCount;
                    break;
                }
                logActivity(`Retrying due to missing data (${sortFailureCount}/3)...`);
                await autoScroll();
                await new Promise(r => setTimeout(r, 1000));
                continue;
            }
             // Successful move
             sortedCount = res;
             consecutiveRecoveryFailures = 0;
        } else {
            logActivity("No video elements. Waiting...");
            await new Promise(r => setTimeout(r, 1500));
            consecutiveRecoveryFailures++;
        }
    }
    
    // Final status
    if (stopSort === true) {
        logActivity("Sorting canceled ⛔");
        stopSort = false;
    } else {
        logActivity(`Sorting complete ✓ (${sortedCount} videos)`);
        // Scroll to top to ensure the completion message is visible
        document.scrollingElement.scrollTo({ top: 0, behavior: 'smooth' });
    }
};

// Initialize UI
let init = () => {
    onElementReady('ytd-playlist-video-list-renderer', false, () => {
        // Avoid duplicate
        if (document.querySelector('.sort-playlist')) return;

        autoScrollInitialVideoList = true;
        useAdaptiveDelay = true;

        addCssStyle();
        renderContainerElement();
        renderToggleButton();
        renderSelectElement(0, modeAvailable, 'Sort Order');
        renderLogElement();

        const checkInterval = setInterval(() => {
            if (isYouTubePageReady()) {
                logActivity('✓ Ready to sort');
                clearInterval(checkInterval);
            }
        }, 1000);
    });
};

// Initialize script
(() => {
    init();
    // Re-init UI on in-app navigation (guard for browsers without navigation API)
    if (window.navigation && typeof navigation.addEventListener === 'function') {
        navigation.addEventListener('navigate', () => {
            setTimeout(() => {
                if (!document.querySelector('.sort-playlist')) init();
            }, 500);
        });
    }
})();
```

</details><br>


Absolute Enable Right Click:  
[Greasyfork](https://greasyfork.org/en/scripts/23772-absolute-enable-right-click-copy)
<details>
<summary>Archive (1.8.9)</summary>

```js
// ==UserScript==
// @name          Absolute Enable Right Click & Copy
// @namespace     Absolute Right Click
// @description   Force Enable Right Click & Copy & Highlight
// @shortcutKeys  [Ctrl + `] Activate Absolute Right Click Mode To Force Remove Any Type Of Protection
// @author        Absolute
// @version       1.8.9
// @include       *://*
// @icon          https://i.imgur.com/AC7SyUr.png
// @compatible    Chrome Google Chrome + Tampermonkey
// @grant         GM_registerMenuCommand
// @license       BSD
// @copyright     Absolute, 2016-Oct-06
// @downloadURL https://update.greasyfork.org/scripts/23772/Absolute%20Enable%20Right%20Click%20%20Copy.user.js
// @updateURL https://update.greasyfork.org/scripts/23772/Absolute%20Enable%20Right%20Click%20%20Copy.meta.js
// ==/UserScript==

(function() {
    'use strict';

    var css = document.createElement('style');
    var head = document.head;

    css.type = 'text/css';

    css.innerText = `* {
        -webkit-user-select: text !important;
        -moz-user-select: text !important;
        -ms-user-select: text !important;
         user-select: text !important;
    }`;

    function main() {

        var doc = document;
        var body = document.body;

        var docEvents = [
            doc.oncontextmenu = null,
            doc.onselectstart = null,
            doc.ondragstart = null,
            doc.onmousedown = null
        ];

        var bodyEvents = [
            body.oncontextmenu = null,
            body.onselectstart = null,
            body.ondragstart = null,
            body.onmousedown = null,
            body.oncut = null,
            body.oncopy = null,
            body.onpaste = null
        ];

        [].forEach.call(
            ['copy', 'cut', 'paste', 'select', 'selectstart'],
            function(event) {
                document.addEventListener(event, function(e) { e.stopPropagation(); }, true);
            }
        );

        alwaysAbsoluteMode();
        enableCommandMenu();
        head.appendChild(css);
        document.addEventListener('keydown', keyPress);
    }

    function keyPress(event) {
        if (event.ctrlKey && event.keyCode == 192) {
            return confirm('Activate Absolute Right Click Mode!') == true ? absoluteMode() : null;
        }
    }

    function absoluteMode() {
        [].forEach.call(
            ['contextmenu', 'copy', 'cut', 'paste', 'mouseup', 'mousedown', 'keyup', 'keydown', 'drag', 'dragstart', 'select', 'selectstart'],
            function(event) {
                document.addEventListener(event, function(e) { e.stopPropagation(); }, true);
            }
        );
    }

    function alwaysAbsoluteMode() {
        let sites = ['example.com','www.example.com'];
        const list = RegExp(sites.join('|')).exec(location.hostname);
        return list ? absoluteMode() : null;
    }

    function enableCommandMenu() {
        var commandMenu = true;
        try {
            if (typeof(GM_registerMenuCommand) == undefined) {
                return;
            } else {
                if (commandMenu == true ) {
                    GM_registerMenuCommand('Enable Absolute Right Click Mode', function() {
                        return confirm('Activate Absolute Right Click Mode!') == true ? absoluteMode() : null;
                    });
                }
            }
        }
        catch(err) {
            console.log(err);
        }
    }

    var blackList = [
        'youtube.com','.google.','.google.com','greasyfork.org','twitter.com','instagram.com','facebook.com','translate.google.com','.amazon.','.ebay.','github.','stackoverflow.com',
        'bing.com','live.com','.microsoft.com','dropbox.com','pcloud.com','box.com','sync.com','onedrive.com','mail.ru','deviantart.com','pastebin.com',
        'dailymotion.com','twitch.tv','spotify.com','steam.com','steampowered.com','gitlab.com','.reddit.com'
    ]

    var enabled = false;
    var url = window.location.hostname;
    var match = RegExp(blackList.join('|')).exec(url);

    if (window && typeof window != undefined && head != undefined) {

        if (!match && enabled != true) {

            main();
            enabled = true

            //console.log(location.hostname);

            window.addEventListener('contextmenu', function contextmenu(event) {
                event.stopPropagation();
                event.stopImmediatePropagation();
                var handler = new eventHandler(event);
                window.removeEventListener(event.type, contextmenu, true);
                var eventsCallBack = new eventsCall(function() {});
                handler.fire();
                window.addEventListener(event.type, contextmenu, true);
                if (handler.isCanceled && (eventsCallBack.isCalled)) {
                    event.preventDefault();
                }
            }, true);
        }

        function eventsCall() {
            this.events = ['DOMAttrModified', 'DOMNodeInserted', 'DOMNodeRemoved', 'DOMCharacterDataModified', 'DOMSubtreeModified'];
            this.bind();
        }

        eventsCall.prototype.bind = function() {
            this.events.forEach(function (event) {
                document.addEventListener(event, this, true);
            }.bind(this));
        };

        eventsCall.prototype.handleEvent = function() {
            this.isCalled = true;
        };

        eventsCall.prototype.unbind = function() {
            this.events.forEach(function (event) {}.bind(this));
        };

        function eventHandler(event) {
            this.event = event;
            this.contextmenuEvent = this.createEvent(this.event.type);
        }

        eventHandler.prototype.createEvent = function(type) {
            var target = this.event.target;
            var event = target.ownerDocument.createEvent('MouseEvents');
            event.initMouseEvent(
                type, this.event.bubbles, this.event.cancelable,
                target.ownerDocument.defaultView, this.event.detail,
                this.event.screenX, this.event.screenY, this.event.clientX, this.event.clientY,
                this.event.ctrlKey, this.event.altKey, this.event.shiftKey, this.event.metaKey,
                this.event.button, this.event.relatedTarget
            );
            return event;
        };

        eventHandler.prototype.fire = function() {
            var target = this.event.target;
            var contextmenuHandler = function(event) {
                event.preventDefault();
            }.bind(this);
            target.dispatchEvent(this.contextmenuEvent);
            this.isCanceled = this.contextmenuEvent.defaultPrevented;
        };

    }

})();
```

</details><br>

# Extensiones

## Pasivas

## Activas

## Normalmente Desactivadas

### Para mi para editar esto xd:
git add .
git commit -m "udate"
git push
