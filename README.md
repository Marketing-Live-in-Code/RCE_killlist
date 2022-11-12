# 三部曲－最淺白的RCE攻擊，誅九族大法
(參考資料：[splitline](https://github.com/splitline/py-sandbox-escape)課程講義)
在文章「[二部曲－最淺白的RCE攻擊，黑名單防禦大法]()」中，明顯展示黑名單防禦大法的不足，因此本篇文章選用刪除的方式，將環境中所有有安全疑慮的方法全部移除，能否有效的達到防護的效果呢？

## 誅九族大法
```python
def make_secure():
     kill = ['open',
             'eval'
             ]
     for func in kill:
         del builtins.dict[func]
 make_secure()
```
依照設想，利用del在sendbox執行之前，先將__dict__中所有有安全問題的方法全部刪除（make_secure()方法），這樣有心人士想使用這些危險的函數、方法，也沒得用了，應該安全了吧？

您可能會發現，在make_secure()方法中的kill陣列，裏頭刪除的函數只有open、eval，為何沒有刪除其他的危險函數，譬如exec這個方法，原因在於誅九族大法也會造成sendbox程式無法使用一些函式，譬如sendbox中會使用到exec，因此雖然exec是有危險性的，但也無法移除。有此可知本身能刪除的數量有限，這個方式的第一個缺陷已經顯現了。

漏洞何在？
### Python2版本

若是在python2當中，reload方法是內建的，所以也必須要將reload函數也一併刪除，否則有心人士也可以用下方程式碼，重新將`__builtins__`下載，這樣前篇文章的手法又重演了。
```python
reload(__builtins__)
```

### Python3版本
python3的使用者也不要高興的太早，雖然reload方法不是內建，因此無法重新將__builtins__下載，但其他地方依然還是可以找到相關危險函數，還記得python所有東西都是物件（object）嗎？可以用「__mro__」找到所有物件繼承的鍊結。由下圖8可以發現，大家最後都指回去object。
![顯示所有物件的鍊結](https://i.imgur.com/V5ALW6F.png)
我們就來看看在基本的object物件下，會不會有我們想要的RCE方法存在。
```python
[].class
[].class.bases
[].class.bases[0]
[].class.bases[0].subclasses()
```
![尋找RCE方法](https://i.imgur.com/L5yBiJS.png)
下圖顯示object的子類別數量，一共有1272個，數量之龐大，因此使用terminal來顯示，並將結果放到文字編輯器中解析。
![用terminal執行](https://i.imgur.com/H8gBB3W.png)

在文字編輯器中進行整理後，發現`os._wrap_close`，找到這個危險套件了，在編號118行，代表是在陣列中的編號117中，這個編號位置可能會因為版本的不同而有所差異。
![在文字編輯器中找到os套件](https://i.imgur.com/WfyVylG.png)
```python
[].class.bases[0].subclasses()[117].init
[].class.bases[0].subclasses()[117].init.globals__
[].class.bases[0].subclasses()[117].init.globals__['system']('ls')
```
因此進入`os._wrap_close`的建構子（`__init__`）中，然後搜尋所有的全域變數（`__globals__`），發現裡頭有os的system方法，那就可以使用此方法，下任何的電腦指令了，在下圖的範例中就列出了該工作目錄下的檔案。
![使用system方法執行任意指令](https://i.imgur.com/d4k7YFX.png)

由此可知，如果要使用珠九族大法，一來有些必要的函式雖然有資安疑慮，但苦於程式中還要使用，因此無法刪除，另一方面，就算可以全部刪除，從object還是可以找到危險函數，範例中只有使用陣列[]，字串等等都還沒有使用，因此珠九族永遠殺不完，甚至要把全國（全部函式）殺光光才安全。
![在sendbox中實際執行情況](https://i.imgur.com/kdHTOkn.png)
總結三部曲會發現「到道一尺，魔高一丈」，在資訊的領域，沒有所謂的絕對安全，只能夠盡力的從各個方面進行防禦，那這時候您可能會問，那還花那麼多錢做資訊安全幹嘛？如果不花這些錢，駭客就可以把您公司的server當家裡廚房，自由的來來去去了，若消費者知道這間公司是這樣，您還會想買它的產品嗎？您是該公司的上下游，還會想跟該公司做生意嗎？。

(參考資料：[splitline](https://github.com/splitline/py-sandbox-escape)課程講義)
