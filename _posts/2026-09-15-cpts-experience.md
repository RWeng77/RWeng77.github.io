---
title: "CPTS 心得"
date: 2026-09-15 00:00:00 +0800
categories: [Certification]
tags: [CPTS, HTB, certification, pentesting]
---

# CPTS 心得   
## 前言   
這篇心得都是自己的想法，並沒有使用生成式 AI 進行任何修正或潤飾，有任何資訊錯誤或有問題的地方還請見諒   
   
人權證明 ： [https://academy.hackthebox.com/achievement/badge/1e7084ba-b08b-11f1-82d1-bea50ffe6cb4](https://academy.hackthebox.com/achievement/badge/1e7084ba-b08b-11f1-82d1-bea50ffe6cb4)    

首先說明一下為什麼是挑選這張作為證照噴錢之旅的第一站，說實在沒有什麼特別的原因，最主要是因為沒錢考 OffSec 的證照而已，對就是這麼膚淺的理由XD。其次是看過身邊朋友在考取 OffSec 證照時有遇到很多問題以及說教材沒更新之類的評價，所以當時就沒有納入考量   
   
## 準備工作   
先介紹最世俗的花了多少錢，我是使用學生方案 $8/M，因為我不確定會在 Role Path 上花多少時間，要考的時候再花了 $210 買 Exam Voucher，中間因為一些事情所以打的斷斷續續的，具體時間是 2025/09 - 2026/05，所以是 9\*8+210 = $282     
   
另一個我會推薦的是 Silver Annual $490/Y ，會送 2 次 Exam Vouchers （記得當初好像看只有送 1 次而已 QQ，虧麻了）   
   
再來是 HTB Lab VIP+ 25$/M，其它就懶得算了也不敢算 ww   
### Official Resources   
官方的資源是 HTB Academy 平台上的教材，因為一定要打滿才能考試，上面的內容非常的充實，照著官方 Penetration Tester 的 path 基本上可以 100% 可以把 14 把 flag 打滿，甚至會充分到有點 overkill，但我想要分享的是遇到的一些問題。   
   
首先是自己眼殘，這個問題環繞了我整個 CPTS 考試，Penetration Tester Path 的第一個 module 是 Penetration Testing Process，裡面完整介紹了整個滲透測試的流程， Module 的第 2 頁列了洋洋灑灑的 36 個 Modules，我以為要全部打完才能考試，所以我認真的開始瞎忙。打到一半發現有一個 Tier 4 的 OSINT Module ，想說幹這麼可能那麼貴，而且為什麼考試會用到大量的 OSINT 不太合理，回去細看才發現真正的 path ==   
   
其次就是部分 Modules 的開出來的 Machine 效能問題，尤其是 Windows AD 相關的 Modules，有時候會有 Instance 開不起的狀況。除此之外，距離台灣最近的 Server 應該是在新加坡，所以自己在戳 time-based SQLi 的時候戳的很痛苦，看到連 Flag format 都是錯的輸出真的很躁   
![image_1789181283877_0](/assets/img/cpts/image_1789181283877_0.png)    
而且有爬過各論壇或官方渠道，遇到同樣問題的不是少數，所以我有丟去 DC 官方問，可以參考官方的回覆   
> Correct me if I'm wrong, SQLMap can detect the SQL Injection as time-based blind injection, which will depend on the connectivity to the target from your network, where any subtle fluctuation can result in SQLMap to detect the wrong character
基本上就是網路問題要自己處理喔啾咪～   
   
另外還有 HTB 有開自己的 [CPTS Preparation Track](https://app.hackthebox.com/tracks/76)，但這個我當時在準備的時候沒有看到所以我沒有打到XD   
   
最後就是教材更新的頻率，我自己是沒看過 OffSec 的教材，聽說蠻舊的， 而 HTB 上的教材大部分也都是 3-4 年前的內容，會介紹到的 CVE 或攻擊手法也都是舊的，所以不用想在上面看到太新穎的東西。除了上面印象比較深的以外其他就沒什麼印象了，因為解完 Path 其實也已經有一段時間了   
### Unofficial Resources   
跟大部分看到的心得文章一樣，我參考了 BRM 的 [blog](https://www.brunorochamoura.com/posts/cpts-experience/)，他分享了很多關於考試的技巧以及撰寫報告的方法 ，以及打了 IppSec 的 [unofficial CPTS Prep](https://www.youtube.com/playlist?list=PLidcsTyj9JXItWpbRtTg6aDEj10_F17x5) 作為練習   
![image_1789428844905_0](/assets/img/cpts/image_1789428844905_0.png)    
基本上是挑 Medium 跟 Hard 作為練習，還有 HTB LABS 中的 Seasonal Machine，自己是當季解到 Ruby Tier，而 List 中的 Insane 機器則是配飯在看，因為實在是打到有點疲勞了XD   
   
喔對有的心得還說會去打 Pro Lab 當做準備，但這個我就完全沒摸過了   
### 筆記   
這個必須老實說，~~我的筆記約等於沒做~~，在打 Role Path 的時候一開始有寫，然而後面就放推了   
   
**極度不推薦這種作**法！！！**極度不推薦這種作**法！！！**極度不推薦這種作**法！！！   
   
因為在後續考試的時候遇到不同情境在想要怎麼戳的時候還要去花大量時間查工具的用法，雖然很多別人在 Github 上整理的 CPTS Cheatsheet 之類的東西，但還是根據學習情況準備一個自己看得懂的東西比較好，也是我結束考試以後暫時最大的目標   
## 正式考試   
一張 Exam Voucher 的考試機會是 2 次，滿分為 100 分，共 14 把 flag  ，及格是要求拿到 85 才算過，基本上是要拿到前 12 把 flag   
   
特別要求是需要產出一份商業等級的測試報告，並且必須要交才算分 or 解鎖第 2 次考試機會，就算交空白報告也行，當然並不推薦交空白的報告，因為審核報告的考官會視你的報告內容給建議，所以請務必記得交報告 owob   
   
而 2 次考試機會的環境是一樣的，所以可以儘量多拉一些資訊，如果真的第一次失敗了可以在中間間隔的 14 天複盤   
   
在考試之前有爬文看到會有魔王關 flag 1 跟 flag 8，實際在打的時候反而卡我最久是 flag 6，也是導致我第一次考試失敗的地方，回頭看下來整個人像是跟中了鏡花水月一樣 XD，但真的跟爬到的文中說的一樣只要 1 跟 8 過了就會開始飆進度，飆著飆著 14 把 flag 就噴完了   
![image_1789431268480_0](/assets/img/cpts/image_1789431268480_0.png)    
如果正常的話我會在 85 分就先停手寫報告，但是我拿到 85 分時候後第 2 次考試還剩 7 天多 + 有人慫恿我嘗試拿滿，所以我只有稍微整理一下就繼續打了，還好沒打多久就破台 uwu   
   
至於考試中的記錄以及截圖，我自己的作法把拿到資訊後或執行特定步驟後做截圖，然後馬上把檔名改成 <box name>\_<step info>，例如 Linux01\_CVE\_xxxx\_xxxxx\_RCE.png 或 Windows03\_nxc\_shares.png 之類的，這樣後續在做報告的時候看名字+對照 time line 可以直接就知道哪張在幹嘛不用特別點進去看，我還會額外加工圖片中的資訊，例如將指令部分框起來再做說明，有缺的話就再回去補，~~也可以最後 reset 機器一次截完~~   
   
回過頭審視考試題目**整體**其實並沒有想象中那樣難到很誇張，像是可以看到 flag 2 跟 3 之間的間隔只有 3 min XD， 而真的難關（ flag 8 ）後可以串的攻擊鏈很明確，真的就是卡在 Entry Point 而已，甚至有兩台機器我都是先打到 Root flag 才拿到 User flag   
   
這邊附上我的 Time Line，可以看到非常死人的作息，因為有靈感就是一直戳到累才停 XD   

| Run | Date | Time | Event |
|:---|:---|:---|:---|
| 1 | 2026/08/10 | 18:00 | Run 1 開始 |
| 1 | 2026/08/11 | 02:24 | flag 1 |
| 1 | 2026/08/11 | 13:38 | flag 2 |
| 1 | 2026/08/11 | 13:41 | flag 3 |
| 1 | 2026/08/11 | 19:30 | flag 4 |
| 2 | 2026/08/14 | XX:XX | flag 5 |
| 1 | 2026/08/19 | 18:00 | Run 1 失敗 |
| 1 | 2026/08/23 | XX:XX | Run 1 Feedback |
| 2 | 2026/09/03 | 22:11 | Run 2 開始 |
| 2 | 2026/09/03 | 23:05 | flag 6 |
| 2 | 2026/09/03 | 23:27 | flag 7 |
| 2 | 2026/09/06 | 03:56 | flag 9 |
| 2 | 2026/09/06 | 03:57 | flag 8 |
| 2 | 2026/09/06 | 09:37 | flag 11 |
| 2 | 2026/09/06 | 09:38 | flag 10 |
| 2 | 2026/09/06 | 10:55 | flag 12 |
| 2 | 2026/09/07 | 02:39 | flag 13 |
| 2 | 2026/09/07 | 05:21 | flag 14 |
| 2 | 2026/09/12 | XX:XX | 送出報告 |
| 2 | 2026/09/15 | 06:24 | 結果 |

最終送出的報告是 158 頁，略高於爬文查到的 120 幾頁，比較多是因為我真的怕沒過所以截了一堆圖去佐證我的想法，實際感覺不用那麼多圖也沒差 (?   
   
最後得到的 Feedback 看起來是跟別人一樣的罐頭回覆 :(    
> Hi ReFa,
>
> Congratulations on passing the HTB Certified Penetration Testing Specialist (CPTS) exam! You are now an official holder of the CPTS certification. Feel free to share your achievement on LinkedIn, etc. You earned it! Working through all of the content in this job role path and mastering it by gaining 100 points on this exam is no small feat!
>
> We found your report to be very well done. You captured the description and impact of each vulnerability very well. You gave actionable remediation recommendations that do not break the line of independence that we must maintain as pentesters (essential third-party auditors) by recommending specific technologies or attempting to re-write the customer's code in the report.
>
> Here is some constructive feedback on content areas you could focus on to strengthen your skill-set:
>
> It is always good to add descriptive captions to both command output and shell output so the technical people reading it (and possibly using it as a guide to reproduce + test their remediation efforts) can quickly identify what is being shown.
> Where possible, it can be good to show a few more screenshots/command output to ensure that anyone reading the report can reproduce the issues shown.
> Overall your report was excellent, well presented, precise, neat, and professional, and the tips above are just constructive feedback.
>
> Again, congratulations on a job well done, and feel free to show off your hard-earned certificate!
>
> Thank you, William Moody (@bmdyy)

廢話有點多XD 差不多分享到這邊吧！   
