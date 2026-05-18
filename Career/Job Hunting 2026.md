---
tags:
  - god_bless_me
  - career
---


| Company      | Tier | PHASE 0 (coffee time) | PHASE 0(resume / recruitment system) |
| ------------ | ---- | :-------------------: | :----------------------------------: |
| Google       | 0    |           v           |                  v                   |
| Amazon       | 0    |                       |                                      |
| Nvidia       | 0    |                       |                                      |
| Meta         | 0    |                       |                                      |
| ASPEED       | 0    |                       |                                      |
| RealTek      | 0    |                       |                  v                   |
| MediaTek     | 0    |                       |                  v                   |
| TSMC         | 0    |                       |                  v                   |
| Novatek (聯詠) | 0.5  |                       |                  v                   |
| Sifive       | 0.5  |           v           |                  v                   |
| ROKU         | 0.5  |                       |                                      |
| Qualcomm     | 0.5  |                       |                                      |
| Ubiquiti     | 1    |                       |                                      |
| Micron       | 1    |                       |                                      |
| Ambarella    | 1.5  |                       |      104 [[Resume - v1]] (4/17)      |
| KLA          | 2    |                       |                                      |
| Canonocial   | 2    |                       |                  x                   |
| Andes        | 2    |                       |                                      |
| Wistron (緯創) | 2    |                       |                                      |
| Quanta (廣達)  | 2    |                       |                                      |
| ASUS         | 3    |                       |                                      |
| Compal (仁寶)  | 3    |                       |                                      |
| Supermicro   | 4    |                       |                                      |
| 系微           | 4    |                       |                                      |

---

# Google

| PHASE |                                                       |
| ----- | ----------------------------------------------------- |
| 0     | HR phone call (4/16)                                  |
| 1     | sent resume by Google Career [[Resume - v2]] (4/24)   |
| 2     | HR phone call (4/29)                                  |
| 3     | coding interview (5/11)                               |
| 4     | HR phone call (5/13)                                  |
| 5     | Behavior Interview (5/18)<br>Googleness & Leardership |
| 6     | coding interview                                      |
| 7     | coding interview                                      |
|       |                                                       |
|       |                                                       |

### Coding interview - 1 (中文)

#### 情境題

設計一個系統: 假設今天有一個裝置有註冊 ISR, 在 ISR 觸發的時候，我們會透過給定的 API 拿到溫度資料，假設此時我們讓其他使用者可以透過一個 `register()` function 來註冊，這些註冊的使用者會在 ISR 發生的時候，透過註冊的 callback 來被通知並拿到溫度資料。

#### Feedback
 >  面試表現挺好，performance rating 都挺好，對 library 都很熟悉，對 Linux Kernel 很熟悉

#### Takeaways
- ISR 裡面拿不到 current (task_struct), 所以設計用 current 當作 key 的方式時要考慮 context 是否在 ISR 裡面
- 可以思考如果不用 current 當 key, 改用一個 atomic variable, 這樣每次 `register()` 的時候都會拿到一個 unique 的 ID ?


###  Behavior (中文)

- 面對工作上有比較 ambiguous 的任務要怎麼處理？
- 遇到一個很緊急的任務，但是你知道開發時間來不及，怎麼辦？
- 假設有另外一個 team 是開發跟你的team一模一樣的功能
	- 你會怎麼去跟對方取的聯絡？
	- 取得聯絡之後你想得到什麼訊息？
	- 你會想要去阻止這件事情發生嗎？
- 假設專案中有4個成員，大家做的份量是一樣的，但是有某一個成員他在 presentation 的時候把功勞都說成他自己的要怎麼辦？


### Coding interview -2

### Coding interview -3

- Priority Queue 也有考過

---

# SiFive

| PHASE |                                                             |
| ----- | ----------------------------------------------------------- |
| 0     | sent resume by 104 [[Resume - v1]] (4/14)                   |
| 1     | HR phone call (4/22)<br>HR                                  |
| 2     | phone interview (4/24)<br>manager 1                         |
| 3     | coding interview (5/8)<br>team lead 1 + team lead 2         |
| 4     | oversea manager interview (5/12 morning.)<br>manager 2      |
| 5     | onsite interview (5/12 noon.)<br>HR + manager 1 + manager 3 |
|       |                                                             |

---

# TSMC

| PHASE |                                                               |
| ----- | ------------------------------------------------------------- |
| 0     | sent resume by TSMC website [[Resume - v1]] (4/22)            |
| 1     | 收到面試邀約 (4/27)                                                 |
| 2     | HackerRank (5/4)                                              |
| 3     | Manager interview (5/5)                                       |
| 4     | HR interview (5/13)                                           |
| 5     | onsite testing (5/15)<br>- English test<br>- Personality test |
|       |                                                               |

---

### Manager Interview

- 對於主管的領導風格，有什麼喜歡和不喜歡的地方
- 對這份工作的期待？
	- 技術
	- 領導能力
- 對自己3-5年後的期待？
	- 技術
	- 領導能力
- 你對這個職務的了解？ 知道要做什麼嗎？
- 可以接受輪班嗎？

### HR Interview

- 有沒有曾經有過自己不如別人的感覺？
- 過去有沒有遇過什麼挫折？ 怎麼解決？
	- 挫折來源於對自身認知和現實的落差
		- 1. 提升自己實力
		- 2. 自己的評價要和現實相符
- 如果發現同事偷帶手機會怎麼處理？
- 可以接受輪班嗎？

---

# ASPEED

| PHASE |     |
| ----- | --- |
| 0     |     |
| 1     |     |
|       |     |



---

# Mediatek

| PHASE |                                                   |
| ----- | ------------------------------------------------- |
| 0     | sent resume by 104 website [[Resume - v2]] (4/27) |
| 1     |                                                   |
|       |                                                   |

---

# Realtek

| PHASE |                                                                                       |
| ----- | ------------------------------------------------------------------------------------- |
| 0     | sent resume by [RealTek](https://recruit.realtek.com/) website [[Resume - v2]] (4/27) |
| 1     |                                                                                       |
|       |                                                                                       |

---

# Novatek

| PHASE |                                           |
| ----- | ----------------------------------------- |
| 0     | sent resume by 104 [[Resume - v2]] (4/24) |
| 1     |                                           |
|       |                                           |


---

# Canonical

## Linux Kernel Engineer

| PHASE |                                                  |
| ----- | ------------------------------------------------ |
| 0     | sent resume by greenhouse [[Resume - v1]] (4/20) |
| 0     | email rejected (4/22)                            |

## PC Platforms Kernel Engineer - Ubuntu Linux

| PHASE |                                                        |
| ----- | ------------------------------------------------------ |
| 0     | sent resume by greenhouse [[Resume - Embedded]] (4/22) |
| 1     | email rejected (4/24)                                  |
