# Userscripts

## Siempre Activadas

Para instalar Userscripts:
[ViolentMonkey](https://addons.mozilla.org/en-US/firefox/addon/violentmonkey/)

4chan X:
[Greasyfork](https://greasyfork.org/es/scripts/7750-4chan-x) 
[Github](https://github.com/ccd0/4chan-x/)

TwitchAdSolutions:
[Github](https://github.com/pixeltris/TwitchAdSolutions (vaft))

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
</details>

Youtube CPU Tamer:
[Greasyfork](https://greasyfork.org/en/scripts/431573-youtube-cpu-tamer-by-animationframe)
[Github](https://github.com/cyfung1031/userscript-supports)

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
</details>

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
</details>

osu! Expert+:
[Github](https://github.com/inix1257/osu_expertplus)

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
</details>

Instagram Default Volume:
[Greasyfork](https://greasyfork.org/en/scripts/487108-instagram-default-volume-with-volume-booster)

Pagetual:
[Website](https://pagetual.hoothin.com/)
[Greasyfork](https://greasyfork.org/en/scripts/438684-pagetual)
[Github](https://github.com/hoothin/UserScripts)

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

</details>

## Activadas a veces

YoutubeSortByDuration:
[Greasyfork](https://greasyfork.org/en/scripts/530129-youtubesortbyduration)
[Github](https://github.com/cloph-dsp/YouTubeSortByDuration)


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
</details>
# Extensiones

## Pasivas

## Activas

## Normalmente Desactivadas

### Para mi para editar esto xd:
git add .
git commit -m "udate"
git push
