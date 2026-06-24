# Userscripts

Para instalar Userscripts:
[ViolentMonkey](https://addons.mozilla.org/en-US/firefox/addon/violentmonkey/)

4chan X:
[Greasyfork](https://greasyfork.org/es/scripts/7750-4chan-x) 
[Github](https://github.com/ccd0/4chan-x/)

TwitchAdSolutions:
[Github](https://github.com/pixeltris/TwitchAdSolutions (vaft))

osu! Expert+:
[Github](https://github.com/inix1257/osu_expertplus)

(osu!) Medals:
[Greasyfork](https://greasyfork.org/en/scripts/577191-medals)
<details>
<summary>
Code Archive (1.0.4)
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

# Extensiones

## Pasivas

## Activas

## Solo a veces

### Para mi para editar esto xd:
git add .
git commit -m "udate"
git push
