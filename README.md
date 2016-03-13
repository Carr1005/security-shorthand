# 關於Password Hashing

## 前言
因為想要準備一套會員機制的模組，必須面對像是密碼、Cookie的保存問題，基於職業道德^^"，想研究一下要怎麼implement才是比較安全的做法，文章以自言自語的方式進行，是幫助我在看完一大堆相關文章後，釐清實作上一些措施的因果關係，避免產出不知所以然的程式碼。一些名詞解釋目前先不打算寫上來，因為reference的連結裡都寫得很好了。對這領域沒有深耕，也只是歸納一些不負責心得，蠻希望有大大可以狠狠甩臉指正的XD


## 密碼不加上salt做hash (X)
```
MD5(password)
```
大家會用的密碼重複的機率是很大的，因為可能拿來當密碼的不外乎可能是名詞、名字、生日或身分ID，其組成性質有很多限制，在字符所有可能的排列組合中，其實占的比例並不大。
假設今天我們因為資料庫的外洩，密碼欄位雖然都是hash值，但是dictionary(收錄各種可能是密碼的字符組合，並將它們都做hash，完成一張有著一對對明碼跟hash值的precomputed table)現在非常普遍，只要拿著洩漏資料庫中密碼的hash值來比對，有很大的機會可以比對到。如果你的會員量蠻大的，對密碼欄位做group，那些常用密碼的hash值，可能就會被發現是高頻率的。而先從這些下手做dictionary attack，成功機率大大提高，也相對有效率(解了一個就等於解了很多個)，這樣的做法被稱為Statistics Attack(或Reverse lookup table)。


##密碼加上固定salt做hash (X)
```
MD5(password+salt)
```
在上述資料庫外洩的情況，倘若加了固定的salt，的確會讓precompute table失效，但如果這個固定的salt也存在資料庫的某個欄位，那attacker只要找到它，並將它加在本來dictionary裡的字符組合，就可以重新運算hash值，產生新的對照表，這樣情形就跟沒加salt一樣了。此時資料庫中相同的密碼，其hash值還是一樣，還是可以group出高頻率的hash值。
所以保護salt變得重要，因為它是固定的，如果不是存在資料庫，那就可能是被hard-coded在後端程式碼，那就要確保你的後端程式碼不會外洩，但資料庫外洩有一個很大的可能是因為你的後端程式碼已經暴露了，所以比較好的方法還是讓每個使用者的密碼搭配的salt都是不一樣的!


## 密碼加上random salt做hash
```
MD5(password+random salt) 
```
讓每個password都加上一個random的salt再做hash，這個salt值不需要再做甚麼處理，直接存在資料庫，為甚麼固定的時候保護就很重要，現在就可以那麼赤裸的存在資料庫? 因為這不影響它要達到的效果:

+  **杜絕Statistics Attack**

   每個人的salt值都是不一樣的，所以即使密碼是一樣的，其hash出來的值也會不同，Attacker的確可以拿某一筆hash值跟它相對應的salt開始利用dictionary重算對照表，並且開始做比對。但它已經無法像salt是固定時一樣，先group，再找到好下手的高頻率hash值，倘若真的被對到了，也就只危害一個帳號。

+  **降低危害效率**

   除了不怕因為相同密碼而變成Statistics Attack的肥羊，salt是固定的情況下暴露，只要重新計算一張precompute table，就可以開始比對資料庫裡所有的hash值，而當每個人的salt都不同，那precompute table就要不斷重算。


目前的方法，讓attacker在即使能看到資料庫裡面的資料，也必須對於每一對密碼跟hash值重新使用像是dictionary這樣brute force的方法去破解，但可能會有幾個疑問:

+  **既然都可以看到資料庫的資料了，那attacker還需要去破解密碼，然後登入查看該帳號的相關資料嗎 ?**

   首先，讓attacker不知道密碼還是很重要的，倘若密碼真正的值被得知了，attacker很可能用這組密碼去試圖登入別的網站，因為人們總是會常用同一組密碼。又或者一些線上交易有關的服務，會要求再一次輸入密碼。(不常在網路上購物，純屬猜想，如果真的有這種機制，顯然不是很安全)

+  **資料庫都外洩了，attacker有很大的機會有辦法直接以一對明碼跟hash值，換掉某一會員原本的資料的，這樣加密不就沒意義了嗎?**

   這在剛剛假設的線上交易的情形的確就很危險了，雖然如此原本的密碼還是沒有洩漏，若要預防SQL Injection來達成這個目的，後端程式的流程設計就很重要，可以讓後端程式碼在連接資料庫方面分別有read權限，和read_write權限的帳號，想清楚可能注入點所對應的功能是否需要給到read write權限。如果發現hash 值被盜走並且破解了，還是趕緊通知該會員重設密碼，修補程式漏洞吧。


## Pepper，加還不加
Pepper(secret key)搭配HMAC(keyed-hash message authentication code)這樣的方法，目的是即使在salt跟hash過的值雙雙曝露之下，假設attacker絕對不知道pepper，讓他連dictionary這樣的brute force攻擊都無法進行，但pepper必須絕對保密才有意義(跟salt相反)。通常會使用的pattern會像:
```
finalhv = MD5( HMAC(pepper,password) + salt )
```
即使利用資料庫裡能見的finalhv值跟salt，brute force出 HMAC ( pepper , password )的值，因為pepper的絕對保密，它也無法算出最後的password。
到這裡又要自問自答一下:

+ **pepper雖然不知道，但不能用brute force的方法猜嗎 ?**

  這樣的情況下要猜pepper的值的話，假設它是random的128 bit，保證算到天荒地老算不出來。[連結裡有計算範例](http://stackoverflow.com/questions/1354999/keep-me-logged-in-the-best-approach)

+  **pepper為甚麼一定要搭配HMAC ? 不能用一般的hash function就好?**

  HMAC誕生的原因，是因為一般hash function 例如MD5，SHA-1 … 等函式的 H(key+message) 跟 H(message+key) 使用方式，分別會造成length-extention attack跟hash collision，HMAC則以 H( key + H(key+message) ) 的方式來設計。所以HMAC其實還是會以某種一般hash function為基底，例如 HMAC-MD5就是以MD5為基底，但HMAC  function可以克服其基底有的上述漏洞。

  一般hash function的設計會使得attacker可以在不知道key值的情況下施以 length-extension attack(所以絕對不會被知道的key，就變成根本不需要被知道)，雖然還不清楚要如何利用這點加速做到有效的破解，但是HMCA可以防止這樣的情況(意即必須知道key值(pepper)才有威脅，key值的保密是有意義的)。

  這邊尚需釐清的是，倘若不用HMAC來跟pepper合作而是一般hash function，單單靠著pepper的絕對保密，理論上還是讓attacker幾乎猜不到，讓他無法進一步算password值，只是不知道MD5( MD5(pepper,password) + salt )這樣的pattern，內部造成的length-extension attack是否會影響pepper絕對保密帶來的功用，但幾篇文章看下來，似乎對於上述兩種攻擊並不是太過於擔心。

**加入pepper的做法其實不常被實做**，因為要把secret key保護好不容易，secret key不能存在資料庫是基本的，最好還要放在不同server，真的實際應用通常還會使用到HSM這樣的特別的硬體。(能考量這樣的成本，基本上已經是非常大規模的服務了)


## 重要的其實是拖慢速度
實作上，不管有沒有implement加pepper的做法，更重要的是一定要使用KDF(Key Derivation Function)，一般的hash function設計時，其實運算速度是求快的，這對於attacker是有利的。以MD5為例，它不被建議使用，並非其hash collision的問題，而是因為運算過快，加上現在GPU的運算能力非常強大，如果單單只用hash function和random salt，brute force的攻擊其實威脅性蠻大的。KDF會讓hash的動作做大量的iteration，來拉長最後產出key值(password加工到最後的值)的時間。


## 容易搞混的部分
PBKDF2是蠻常被實作的KDF，這個Funtion內部的Algorithm又會使用到HMAC函式，但大部分看到的解釋僅僅是說HMAC的特性很適合拿來做KDF的一部份，HMAC在裡面的使用方法蠻特別的會把我們的password拿來當key使用而非message，至於詳細原因為何可能還要花時間研究(XD就是不會研究了)。
**但並不是說因為PBKDF2裡面有用到HMAC，所以一定要提供pepper**，剛剛有說到PBKDF2內部的HMAC，是拿我們要保護的password來扮演HMAC中key的腳色(本來該由pepper擔綱)，所以我們一樣只需要提供salt跟password就可以implement PBKDF2了。
那如果我們要也想加入pepper來提高安全性呢，那我們流程應該變成，使用HMAC，以password跟pepper為input，並把 HMAC的ouput視為新的password，與salt一起當做PBKDF2的input。


References
--------------

### 總覽式的文章

 * https://crackstation.net/hashing-security.htm#properhashing (2016 0219)
   
   這篇寫得非常好，時間新，而且提到許多有關實作的建議、跟會用到的library，作者有PBKDF2的repository，這篇速記基本上以這篇文章為主軸。

 * http://security.blogoverflow.com/2013/09/about-secure-password-hashing (2013 0913)

 * http://security.stackexchange.com/questions/211/how-to-securely-hash-passwords/31846#31846 (2013 0302)
   
   以上兩篇對於KDF有更詳盡的介紹。 

### Pepper，HMAC，KDF
 
 * http://security.stackexchange.com/questions/41754/what-is-the-purpose-of-a-pepper (2013 0904)
   
   這篇的評論部分有點出KDF裡的HMAC的使用方法稍有不同。

 * http://security.stackexchange.com/questions/29951/salted-hashes-vs-hmac (2013 0130)
   
   這篇提到HMAC的特性適合使用在PBKDF2裡。
 
 * http://crypto.stackexchange.com/questions/8286/recommended-way-of-adding-a-pepper-secret-key-to-password-before-hashing

   詳細說明如何在使用KDF之前，加入pepper的機制。

 * http://crypto.stackexchange.com/questions/20578/definition-of-pepper-in-hash-functions
   
   Pepper定義的討論

 * http://stackoverflow.com/questions/5051529/hmac-vs-simple-md5-hash
   
   HMAC vs 一般Hash function的討論

 * http://stackoverflow.com/questions/8952807/openssl-digest-vs-hash-vs-hash-hmac-difference-between-salt-hmac
   
   這篇以非常簡單的方式解釋KDF的演算法，有助於之後implement KDF，值得一看。

### Length-Extension-Attacks

 * https://blog.skullsecurity.org/2012/everything-you-need-to-know-about-hash-length-extension-attacks